################
UWS data service
################

.. abstract::

   Describes the shared backend service that handles all IVOA UWS database operations via a REST API.
   This system replaced separately-managed UWS databases for each application, simplifying and centralizing database management.

Problem statement
=================

Rubin Observatory expects to develop a significant number of web services for the Rubin Science Platform that accept a request from a user, perform some calculation or transformation on scientific data, and return the results.
The IVOA has a standardized protocol, the `Universal Worker Service Pattern <https://www.ivoa.net/documents/UWS/>`__ for services of this type.
Implementing async stateful services using UWS, which will often be needed since the calculation or transformation may take some time, requires tracking metadata about pending, in progress, and completed jobs.
Generally this is done in a relational database.

Prior to December of 2024, every such service had to have its own separate database.
For services that run in Science Platform instances hosted in Google Cloud, this requires Terraform work for every service to set up separate databases, passwords, and Google service accounts.
Each application also required a relatively complex Kubernetes configuration including Cloud SQL Auth Proxy and workload identity bindings, and complex Helm chart machinery to handle database schema updates when necessary.

.. _Safir: https://safir.lsst.io

The previous application design looks like this:

.. mermaid::

   graph LR
       subgraph Application pod
           app[Application] <--> proxy[SQL Proxy]
       end

       proxy <--> database[Database]

This is somewhat simplified.
Often the application will have a separate worker backend, and both the database worker and the frontend will talk to the database, each with their own proxies.

We are likely to want to change the schema of the UWS database over time, both in response to possible evolution of the UWS standard and as we discover additional metadata we need to store.
SQuaRE provides a support library to aid in using Alembic_ for this purpose, but each application still has to manage the Alembic migrations separately.

.. _Alembic: https://alembic.sqlalchemy.org/en/latest/

Since we anticipate needing multiple applications following this pattern, often written and maintained by teams that are not experts in Kubernetes, web services, or Terraform, we would prefer to hide the complexity of managing the UWS database from the application.

Solution summary
================

We consolidated all UWS databases for Rubin-developed Python services into a single database managed by a new backend web service named Wobbly_.
That service provides a REST API to the UWS job metadata backing store.
UWS-based applications make database updates with REST API calls and do not need to know about the underlying database schema.
Even better, those applications can make calls to that backend service using delegated user tokens, allowing Wobbly to determine the user and service from the token and ensure that each service and user can only see their own jobs.

.. _Wobbly: https://github.com/lsst-sqre/wobbly/

Compatibility concerns as the schema changes over time can be handled through normal REST API compatibility techniques, which are more flexible than database schemas.

The new design looks like this:

.. mermaid::

   graph LR
       ingress[Ingress]
       uws[UWS service]

       app[Application] <--> ingress

       subgraph Ingress
           direction TB
           ingress <--> gafaelfawr[Gafaelfawr]
       end

       subgraph UWS pod
           uws <--> proxy[SQL Proxy]
       end

       ingress <--> uws
       proxy <--> database[Database]

Authentication by the application to the UWS service would be handled by Gafaelfawr_ as with any other Rubin Science Platform service.
See :ref:`authentication` for more details.

.. _Gafaelfawr: https://gafaelfawr.lsst.io/

Only the UWS service is aware of the schema of the underlying UWS database.
It manages the schema, including any schema migrations, independently of any application, and internally handles any forward or backward compatibility conversions between the REST API and the current schema.
This decouples any database management concerns from maintenance of the applications.
The applications use the UWS service as a regular web service, using a client built into the `Safir UWS library`_.

.. _Safir UWS library: https://safir.lsst.io/user-guide/uws/index.html

With this design, most of the Terraform work and the Cloud SQL Auth Proxy are handled by the UWS service, and none of that configuration is required for a new UWS-based application.

.. note::

   The scope of this service is only internally-written Rubin Science Platform applications that would be using the `Safir UWS library`_.
   This design does not propose replacing the UWS databases used by other services, such as the CADC TAP service used by the Rubin Science Platform.
   It would be possible to unify *all* UWS databases used by the Science Platform inside this service, and there may be some advantages (such as a cross-service history API) in doing so, but that would be substantial additional work and is not a necessary part of this design.

API
===

The REST API of the Wobbly service intentionally does not follow the IVOA UWS standard precisely, although it tries to use similar terminology to avoid unnecessary confusion.
It is a standard FastAPI REST API that uses Gafaelfawr_ for authentication and JSON for the serialization format.

.. _authentication:

Authentication
--------------

UWS-based services authenticate to Wobbly by using delegated tokens.
This means that all calls from the UWS-based service to Wobbly are made on behalf of the relevant user.
Wobbly itself enforces visibility restrictions, only returning job records for the service and user associated with the presented token.

In order to accomplish this, the Wobbly service is protected by a `Gafaelfawr service-only ingress <https://gafaelfawr.lsst.io/user-guide/gafaelfawringress.html#service-only-ingresses>`__.
An ingress of this type can only be accessed using internal tokens, which are delegated tokens created for a specific service.

The Phalanx_ configuration for Wobbly includes a whitelist of service names that are permitted to use the UWS job service.
Attempted access via internal tokens issued to any other service will be rejected by Gafaelfawr.

.. _Phalanx: https://phalanx.lsst.io

Wobbly retrieves the username and service name for a given request from the ``X-Auth-Request-User`` and ``X-Auth-Request-Service`` HTTP headers set by the ingress from the Gafaelfawr authorization response.
It then uses those values to constrain the SQL queries performed by the route handler, and to populate the owner and service metadata for newly-created jobs.

If the service attempts to retrieve a job ID that belongs to a different service or user, Wobbly will return an HTTP 404 error, exactly as if the job didn't exist at all.

Wobbly supports a separate admin API that allows Phalanx environment administrators to see jobs for all users and services to aid in debugging problems.
This route uses a separate ingress with traditional Gafaelfawr authentication and requires the ``exec:admin`` scope.

Application routes
------------------

The routes used by applications are:

.. _keyset pagination: https://dmtn-224.lsst.io/#pagination

GET /jobs
    List the jobs for the authenticated user.
    Takes optional query parameters to limit records by phase and creation date.
    Supports `keyset pagination`_ by creation date and then internal job ID in reverse order of creation (newest first), using the ``cursor`` and ``limit`` query parameters.
    Links to the first, previous, and next pages of results are returned in the HTTP ``Link`` header.
    The total count of available records is not returned.

    The `Safir UWS library`_ uses the ``limit`` query parameter to implement the IVOA UWS ``LAST`` parameter, but otherwise does not use pagination, since the IVOA UWS API is not paginated.
    We may change this in the future to limit memory consumption in the Wobbly server if we have problems with applications retrieving large lists of jobs.

    The full job records are returned, including results and errors, but the result URLs will be whatever internal URL the application stored, not a signed URL suitable for providing to a client.
    There is no equivalent to the stripped-down IVOA ShortJobDescription record.

POST /jobs
    Create a new job record.
    Returns an HTTP 201 status code with the HTTP ``Location`` header set to the URL for the new job.
    The owner and service for the job are set automatically based on the authentication credentials, as are other standard fields such as ``phase`` and ``creation_time``.

    All jobs are created in the ``PENDING`` phase.
    If the application wishes to immediately start the job, it must make a subsequent ``PATCH`` request to the job URL (see below) once it has queued the job in its local job queue system.

GET /jobs/<job-id>
    Retrieve a job record by job ID.
    As with ``GET /jobs``, the result URLs will be whatever internal URL the application stored, not a signed URL suitable for providing to a client.

DELETE /jobs/<job-id>
    Delete a job.
    This removes the job entirely rather than moving it to the archived state.
    It's used for user job deletions.
    To abort the job, instead use ``PATCH`` and change the ``phase`` to ``ABORTED``.

PATCH /jobs/<job-id>
    Change attributes of the job.
    This is used for all state transitions, as well as for updating the destruction time and execution duration.

    The state transition is determined by the content of the ``phase`` parameter in the ``PATCH`` body.
    If it is omitted, that indicates an update to the user-controlled metadata of the job and does not perform a state transition.
    Otherwise, ``phase`` may be set to ``ABORTED``, ``QUEUED`` (must be accompanied by the queue system message ID), ``EXECUTING`` (must be accompanied by the start time), ``COMPLETED`` (must be accompanied by the result list), or ``ERROR`` (must be accompanied by a list of errors).
    ``HELD``, ``SUSPENDED``, and ``ARCHIVED`` are not supported in the initial implementation.

The Wobbly data model differs from the IVOA UWS data model in one significant way: Wobbly supports storing multiple errors for a job.
Currently, the `Safir UWS library`_ only supports storing one error and only uses the first error returned, but supporting multiple errors in the Wobbly data model was cleaner.
Eventually that may be plumbed through to the Safir library and converted into a single error for IVOA UWS protocol purposes.

The job parameters stored with each job are, from the Wobbly perspective, a generic serialized JSON object.
Wobbly stores and returns that object but never interprets it.
The `Safir UWS library`_ uses it to serialize and deserialize the parameters class for a given service, which can be an arbitrary Pydantic model.

Wobbly therefore does not store either the original input used when creating the job (which in the IVOA UWS protocol is a sequence of key/value pairs) or the output format (an XML document representing those key/value pairs).
The application must provide a way to create a parameters model from query or form input and produce an XML document corresponding to the parameters model.
The `Safir UWS library`_ is entirely agnostic about how the application chooses to do that and how much structure, and what kind of structure, it uses when storing job parameters.

Admin routes
------------

Although we don't want users to be able to query the UWS service directly, we do want environment administrators to be able to do so in order to debug problems.
We may also have other services that should have global access to all UWS records for any application and user, and for which the UWS service API may be more convenient than direct database access.

This API is read-only and supports the following routes:

GET /admin/jobs
    List jobs for any user and service.

GET /admin/services
    List all services for which jobs are stored in the database.
    Note that this may not match the list of services allowed to use Wobbly to store UWS job information, which is configured in Phalanx.

GET /admin/services/<service>/users
    List all users that have at least one job stored for the given service.

GET /admin/services/<service>/users/<user>/jobs
    List all jobs for the given service and user.
    This is equivalent to the ``GET /jobs`` API for application use, but allows authentication with ``exec:admin`` scope instead.

GET /admin/services/<service>/users/<user>jobs/<job-id>
    Retrieve a specific job by ID.

GET /admin/users
    List all users with at least one job stored in the database, regardless of service.

GET /admin/users/<user>/jobs
    List all jobs for the given user, regardless of service.

All of the routes that end in ``/jobs`` use the same search parameters and pagination method as the ``GET /jobs`` route for applications.

Job results
===========

Wobbly only stores a URL to the job results.
It accepts that URL from the service and returns it when asked for the job record, and otherwise doesn't interact with the job result in any way.

The `Safir UWS library`_ currently assumes that either job results are stored in a :abbr:`GCS (Google Cloud Storage)` bucket or the URL returned by the worker is pre-signed or otherwise authenticated and can be passed directly back to the user.
If the URL is to an object in a GCS bucket (based on the URL having a scheme of ``s3`` or ``gs`` instead of ``https``), the Safir UWS library turns it into a presigned URL whenever it is returned by the application to a user.

This unfortunately means that most UWS-based applications will still require Google Cloud Storage access and therefore Terraform setup so that they can store results in GCS and generate pre-signed URLs when the user requests the job results.
Ideally we would prefer for UWS applications to not need Google API access, since that adds considerably to the complexity of deploying a new application and means that the application is not portable to environments that cannot use GCS.
This will require a different authentication model for accessing the results, and we have not yet come up with a good option.

Job expiration
==============

Currently, job expiration (the destruction time parameter for a job) is not handled.
Nothing happens to jobs that pass their destruction time.
They remain in the same status that the were in previously.

This is not ideal, since the GCS bucket into which results are stored will generally have an expiration time, and therefore old job records will point to results that no longer exist even though their phase does not represent that.
This will have to be fixed in the future.

There are two options here:

#. Drive expiration from each UWS application.
   This allows the application to cancel any job worker that might still be running.
   The drawback is that this reintroduces a more complex authentication model, since job expiration is not associated with a user request and cannot use delegated credentials.
   Instead, services would need a token created with a ``GafaelfawrServiceToken`` resource and a separate route accessible with those tokens, with either a scope restriction and a new scope or some other access control mechanism.

#. Have Wobbly automatically expire job records that are older than the destruction time.
   This can be done either by deleting the job record entirely or by moving it to an ``ARCHIVED`` phase and deleting the result references.
   This is necessarily decoupled from the job execution framework, so jobs that are still running when their destruction time passes won't be aborted, but given that job timeouts are generally on the order of a few hours and destruction times are generally on the order of six months, this is unlikely to be a serious problem.
   This approach is simpler and doesn't require a new authentication model, and services can control the destruction times of their jobs through the destrution time validation callback supported by the `Safir UWS library`_.

At present, option two seems like the better approach, but currently neither are implemented.

Schema
======

The database schema used by Wobbly is very close to the UWS data model with a few modifications:

- Each job record has an additional column, ``service``, that tracks the associated service and limits query results to the authenticated service.

- The job parameters are stored as a PostgreSQL JSONB column.
  As mentioned above, these are not interpreted by Wobbly.

- Job errors are stored via a one-to-many relationship to a separate table.
  This allows job errors to have a separate model, which simplifies a lot of the modeling.
  It also means Wobbly technically supports recording multiple errors for a job, although this is not currently used.

Error codes were, in previous versions of Safir, represented by an enum.
In the Wobbly data model, the error code is a string, since every IVOA protocol appears to use its own distinct and conflicting error codes.

Indices are designed for the service use case.
Some possible admin queries will fall outside the indices and may require table scans.
Admin queries are expected to be rare.

Wobbly uses Alembic_ (via the `Safir schema management support <https://safir.lsst.io/user-guide/database/schema.html>`__) to manage the database schema.

Performance and scaling
=======================

This design will incur some unavoidable additional latency for operations that touch the UWS jobs database.
Instead of a database call through a proxy, each request will require three HTTP requests (application to ingress, ingress to Gafaelfawr, ingress to UWS service) plus the same database call through a proxy.
Hopefully, the additional latency should be small and the cost of the database call should still dominate, particularly for write operations.

If the Gafaelfawr authentication step adds too much delay, we could enable ingress caching of Gafaelfawr responses for the Wobbly service.

The UWS service in this design is stateless, relying entirely on the underlying database for state management, and therefore can easily be horizontally scaled as needed, although it's also very light-weight and likely won't require much scaling.
Most of the performance burden will fall on the underlying database.

There is one scaling advantage in this design, namely that only the UWS service will need to maintain an open connection pool to the database, and therefore the open connection demands and corresponding memory demands on the underlying database will reduce.
In the current design, every application has its own open connection pool, requiring the database to handle more open but usually idle connections.

Currently, the Wobbly service does not do any caching.
It's not obvious that caching would be helpful, and maintaining cache consistency across horizontally-scaled Wobbly instances would be challenging.

Currently, synchronous jobs and waiting for job status changes are both done via polling Wobbly, which means repeated HTTP and SQL requests.
See :ref:`remove-db-worker` for a possible way to address that.

Future work
===========

In addition to handling job expiration and possibly rethinking the way job results are stored, both discussed above, here are some other pieces of future work we may want to do to improve this design.

.. _remove-db-worker:

Remove the database worker
--------------------------

We adopted Wobbly without changing the basic design of an application based on UWS.
(See :dmtn:`208` for the model application used for Wobbly development.)
This meant retaining the two-worker backend model, where one worker runs the scientific code and stores the results in a Google Cloud Storage bucket and another worker updates the database record.
The only change was to have the database worker use Wobbly instead of a direct SQL connection.

This design was originally chosen because of the heavy dependencies required for direct SQL access.
Now, with Wobbly, updating the job status and storing results or errors only requires an HTTP client and Pydantic.
Rubin Science Pipelines containers already include Pydantic, the Pydantic version constraints are (at least currently) not very demanding, and a suitable HTTP client can easily be installed on top.

The next obvious step is therefore to eliminate the separate database worker and move the code to update the job record in Wobbly to a wrapper around the backend worker function.
This will have the additional advantage that then the backend job will not complete until the job results or errors have been stored.
Waiting for job completion can then be done by waiting for the queued job to complete, which is not currently possible because the separate database worker job is not visible to the frontend.
That, in turn, will remove the need to poll Wobbly, potentially making the frontend more responsive and reducing load on Wobbly and the underlying database.

Appendix: Options considered
============================

Below are design choices we considered when developing this approach.
This discussion is primarily of historical interest.

Authentication
--------------

We considered two possible ways, with different trade-offs, to authenticate application requests to the UWS service.
We decided to take the delegated token approach, described in :ref:`authentication`, since it seemed like the more elegant solution and had useful additional security properties.

Option 1: Bot tokens
""""""""""""""""""""

Each application that needs to talk to the UWS service gets its own Gafaelfawr token, created via a Kubernetes ``GafaelfawrServiceTokens`` resource, to use for that purpose.
The application adds that token to an ``Authentication: bearer`` header in all requests to the UWS service.

This decouples user authentication from internal authentication to the UWS service, which avoids the problems with direct user access to the UWS service described in :ref:`authentication-delegated`.
It's also conceptually simpler.
The drawback is that the service always has access to modify the jobs of any user and has to explicitly include the username in the API requests to the UWS service.

This approach requires allocating a separate scope (see :dmtn:`235`) for access to the UWS service, since regular users should not have direct access.
They should only use the UWS service indirectly via requests to UWS endpoints of the user-facing application.
We could use ``write:uws`` for this purpose, or we could create a new scope prefix (``service:``, ``internal:``, or ``bot:``) for scopes of this type that are only used internally by other Science Platform services and are never granted to users.

A simple implementation of this approach would give every service access to the records of any other service, and rely on the service to only access its own records.
A possible improvement would be to have the UWS service look at the username associated with the request, remove an initial ``bot-`` prefix from that username, and then treat that username as the requesting service, limiting access to only records for that specific service.
This is a little bit awkward, but seems like a worthwhile improvement.

.. _authentication-delegated:

Option 2: Delegated tokens
""""""""""""""""""""""""""

A conceptually cleaner design would be for UWS-based applications to request a delegated token for the user and then use that delegated token to authenticate to the UWS service.
The UWS service can then get the identity information for both the application and the user on whose behalf the application is operating from the token and not rely on the application specifying either.
An application will then not be able to affect records for users who are not actively making requests, which is a small but nice security and robustness improvement.

There were two issues with this approach that required some Gafaelfawr development work to fix.

The first and most significant is that, with the previous Gafaelfawr design, this would allow users to access the UWS service directly, bypassing the application.
This is undesirable; the UWS service is an implementation detail of the application, and making changes to it directly without going through the application could break the application.
Worse, the user could set the result of some job to GCS bucket URLs that the user should not have access to and then retrieve the result via the application, relying on the application GCS object signing to give it access to the contents of those bucket objects.

In order to make this safe, therefore, a new concept of a route that can only be accessed by internal tokens with an associated service had to be introduced in Gafaelfawr.
This prevents direct user access but still allows access on behalf of the user by a service with a delegated token.
Since requesting a delegated token requires a Kubernetes configuration change to the ``GafaelfawrIngress`` resource, this restores the desired security boundary.

Unfortunately, although this was not advertised and not desired, a user previously could create arbitrary internal tokens for themselves with arbitrary usernames by directly accessing the Gafaelfawr ``/auth`` endpoint intended for the ingress.
This was a known problem that we postponed addressing since it was not a meaningful security boundary, but it became one with this change.

We fixed this by changing all ingresses to access Gafaelfawr through its internal Kubernetes ``Service`` and then removing the ingress-facing route from the public Gafaelfawr ``Ingress``.
We were then able to rely on the Kubernetes ``NetworkPolicy`` to prevent users from talking to Gafaelfawr directly, and the ingress will refuse to route user requests to that Gafaelfawr route.

The second problem is more minor: currently, the service associated with an internal token is not added to an HTTP header in the incoming request.
The UWS service would therefore have had to make a request to the Gafaelfawr token-info endpoint for every request to determine the associated service, which would have increased the latency cost of this design.
This was addressed by adding a new ``X-Auth-Request-Service`` header to the headers set by the Gafaelfawr integration with the ingress.

In this model, the UWS service itself does not require any token scopes.
Instead, there is an allow list of services whose internal tokens are permitted to talk to the UWS service, and a separate admin route that allows environment administrators to see the data for any service.
