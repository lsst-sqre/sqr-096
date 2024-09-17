################
UWS data service
################

.. abstract::

   Proposes replacing separately-managed UWS databases in each application that uses the UWS protocol with a shared backend service that handles all database operations, using a single database for all applications, and provides a REST API to the applications.

Problem statement
=================

Rubin Observatory expects to develop a significant number of web services for the Rubin Science Platform that accept a request from a user, perform some calculation or transformation on scientific data, and return the results.
The IVOA has a standardized protocol, the `Universal Worker Service Pattern <https://www.ivoa.net/documents/UWS/>`__ for services of this type.
Implementing async stateful services, which will often be needed since the calculation or transformation may take some time, using UWS requires tracking metadata about pending, in progress, and completed jobs.
Generally this is done in a relational database.

SQuaRE provides `a framework for implementing UWS-based services <https://safir.lsst.io/user-guide/uws/index.html>`__, but currently every such service needs its own separate database.
For services that run in Science Platform instances hosted in Google Cloud, this requires Terraform work for every service to set up separate databases, passwords, and Google service accounts.
Each application also requires a relatively complex Kubernetes configuration including Cloud SQL Auth Proxy and workload identity bindings.

The current application design looks like this:

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

Proposed solution summary
=========================

The schema of the UWS database is embedded in the `Safir UWS library`_ and therefore already does not change between applications using that library.
The UWS database operations are therefore suitable for moving into a separate web service that provides a REST API to a backing store for UWS job metadata.
UWS-based applications would then make web service calls to that API.
Compatibility concerns can be handled through normal REST API compatibility techniques, which are more flexible than database schemas.

.. _Safir UWS library: https://safir.lsst.io/user-guide/uws/index.html

The new design would look like this:

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

With this design, only the UWS service would be aware of the schema of the underlying UWS database.
It would therefore manage the schema, including any schema migrations, independently of any application, and would internally handle any forward or backward compatibility conversions between the REST API and the current schema.
This decouples any database management concerns from maintenance of the applications.
The applications would use the UWS service as a regular web service, using a client that would be built into the `Safir UWS library`_.

With this design, the Terraform work, the Cloud SQL Auth Proxy, and the workload identity management would all be handled by the UWS service, and none of that configuration would be required for a new UWS-based application.

.. note::

   In this proposal, the scope of this service is only internally-written Rubin Science Platform applications that would be using the `Safir UWS library`_.
   This design does not propose replacing the UWS databases used by other services, such as the CADC TAP service used by the Rubin Science Platform.
   It would be possible to unify *all* UWS databases used by the Science Platform inside this service, and there may be some advantages (such as a cross-service history API) in doing so, but that would be substantial additional work and is not a necessary part of this design.

API
===

The REST API for the UWS service will evolve as the service is implemented and tested with various applications, but the starting point would be roughly the internal API of the storage layer of the existing `Safir UWS library`_.

.. _authentication:

Authentication
--------------

There are two possible ways, with different trade-offs, to authenticate application requests to the UWS service.
The bot token approach is the simplest, so we will likely start with that, but the second approach has some useful properties that are worth consideration.

Bot tokens
""""""""""

Each application that needs to talk to the UWS service gets its own Gafaelfawr token, created via a Kubernetes ``GafaelfawrServiceTokens`` resource, to use for that purpose.
The application adds that token to an ``Authentication: bearer`` header in all requests to the UWS service.

This decouples user authentication from internal authentication to the UWS service, which avoids the problems with direct user access to the UWS service described in :ref:`authentication-delegated`.
It's also conceptually simpler.
The drawback is that the service always has access to modify the jobs of any user and has to explicitly include the username in the API requests to the UWS service.

This approach requires allocating a separate scope for access to the UWS service, since regular users should not have direct access.
They should only use the UWS service indirectly via requests to UWS endpoints of the user-facing application.
We could use ``write:uws`` for this purpose, or we could create a new scope prefix (``service:``, ``internal:``, or ``bot:``) for scopes of this type that are only used internally by other Science Platform services and are never granted to users.

A simple implementation of this approach would give every service access to the records of any other service, and rely on the service to only access its own records.
A possible improvement would be to have the UWS service look at the username associated with the request, remove an initial ``bot-`` prefix from that username, and then treat that username as the requesting service, limiting access to only records for that specific service.
This is a little bit awkward, but seems like a worthwhile improvement.

.. _authentication-delegated:

Delegated tokens
""""""""""""""""

A conceptually cleaner design would be for UWS-based applications to request a delegated token for the user and then use that delegated token to authenticate to the UWS service.
The UWS service can then get the identity information for both the application and the user on whose behalf the application is operating from the token and not rely on the application specifying either.
An application will then not be able to affect records for users who are not actively making requests, which is a small but nice security and robustness improvement.

There are two issues with this approach that would require some Gafaelfawr development work to fix.

The first and most significant is that, with the current Gafaelfawr design, this would allow users to access the UWS service directly, bypassing the application.
This is undesirable; the UWS service is an implementation detail of the application, and making changes to it directly without going through the application could break the application.
Worse, the user could set the result of some job to GCS bucket URLs that the user should not have access to and then retrieve the result via the application, relying on the application GCS object signing to give it access to the contents of those bucket objects.

In order to make this safe, therefore, Gafaelfawr would have to gain a new concept of a route that can only be accessed by internal tokens with an associated service.
This would prevent direct user access but still allow access on behalf of the user by a service with a delegated token.
If requesting a delegated token required a Kubernetes configuration change, this would restore the desired security boundary.

Unfortunately, although this is not advertised and not desired, a user can create arbitrary internal tokens for themselves with arbitrary usernames by directly accessing the Gafaelfawr endpoint intended for the ingress.
This is a known problem that has not yet been fixed because currently this is not a meaningful security boundary, but it would become one with this change.

This is fixable by changing all ingresses to access Gafaelfawr through its internal Kubernetes ``Service`` and then removing the ingress-facing route from the public Gafaelfawr ``Ingress``.
We can then rely on the Kubernetes ``NetworkPolicy`` to prevent users from talking to Gafaelfawr directly, and the ingress will refuse to route user requests to that Gafaelfawr route.
This is work that we want to do anyway and which is easier now that nearly all services use ``GafaelfawrIngress`` resources.
But it is a fairly large configuration change.

The second problem is more minor: currently, the service associated with an internal token is not added to an HTTP header in the incoming request.
The UWS service would therefore have to make a request to the Gafaelfawr token-info endpoint for every request to determine the associated service, which increases the latency cost of this design.
We would probably want to add the associated service, if available, to an HTTP request header set by the ingress.

Application routes
------------------

The initial anticipated routes used by applications are:

GET /users/<username>/jobs
    List the jobs for the given user.
    Takes query parameters to limit records returned by phase, creation date, or count of records.
    This API should use support pagination eventually, but we can probably skip that for the initial implementation.

POST /users/<username>/jobs
    Create a new job record.
    Returns a redirect to the GET endpoint for the new job record.

GET /users/<username>/jobs/<job-id>
    Retrieve a job record by job ID.

DELETE /users/<username>/jobs/<job-id>
    Delete a job.
    This removes the job entirely rather than moving it to the archived state.
    It's used for user job deletions.

PATCH /users/<username>/jobs/<job-id>
    Change attributes of the job.
    At first, only the destruction time and the execution duration may be changed.
    More attributes may be added later.
    The body is just the attributes of the job record to update.
    Returns the modified job record.

POST /users/<username>/jobs/<job-id>/complete
    Mark the given job as completed.
    The body of the POST is the results of the job.
    Returns a redirect to the GET endpoint for the job.

POST /users/<username>/jobs/<job-id>/fail
    Mark the given job as failed.
    The body of the POST is the error returned by the job.
    Returns a redirect to the GET endpoint for the job.

POST /users/<username>/jobs/<job-id>/phase
    Mark the given job as queued, executing, or aborted.
    The body will be the new phase.
    For example, ``{"phase": "executing"}``.
    Returns a redirect to the GET endpoint for the job.

These routes assume that the authentication credentials from the application are used to determine the service whose records are being retrieved or manipulated.
Records for other applications would then be invisible.
If all applications have access to all records, all of these routes should have ``/services/<service>`` prepended so that the application can specify the UWS service whose records it is trying to retrieve or manipulate.

These routes assume the authentication model where the service has its own credentials and does not use delegated credentials.
If the design discussed in :ref:`authentication-delegated` is adopted instead, the leading ``/users/<username>`` can be removed from the routes.
All responses would then be restricted to the service and username information derived from the authentication token.

Admin routes
------------

Although we don't want users to be able to query the UWS service directly, we do want environment administrators to be able to do so in order to debug problems.
We may also have other services that should have global access to all UWS records for any application and user, and for which the UWS service API may be more convenient than direct database access.

In the simplest authentication model where all applications have full access to the UWS API for any application and user, the same routes, with a ``/services/<service>/users/<user>`` prefix, can be used for both admin access and application access.
Otherwise, we should add a separate set of routes with the ``/services/<service>/users/<user>`` prefix, restricted to an admin scope, that can specify arbitrary services and users, and not allow applications direct access to those routes.

Schema
======

In the initial design, the current schema used by the `Safir UWS library`_ can be used nearly without modification.
We would only need to add a new ``service`` field to the ``jobs`` table that records which service the record is for.
All services should be able to share the same job ID range and rely on job ID assignment via a database autoincrement key.

The UWS schema already uses a generic representation of both job parameters and job results, and we expect to keep that generic representation (although possibly in a different format) across possible changes to the UWS protocol.

Performance and scaling
=======================

This design will incur some unavoidable additional latency for operations that touch the UWS jobs database.
Instead of a database call through a proxy, each request will require three HTTP requests (application to ingress, ingress to Gafaelfawr, ingress to UWS service) plus the same database call through a proxy.
However, the additional latency should be small and the cost of the database call should still dominate, particularly for write operations.

The UWS service in this design is stateless, relying entirely on the underlying database for state management, and therefore can easily be horizontally scaled as needed, although it's also very light-weight and likely won't require much scaling.

There is one scaling advantage in this design, namely that only the UWS service will need to maintain an open connection pool to the database, and therefore the open connection demands and corresponding memory demands on the underlying database will reduce.
In the current design, every application has its own open connection pool, requiring the database to handle more open but usually idle connections.
