Abstract
========

The identity management, authentication, and authorization component of the Rubin Science Platform is responsible for maintaining a list of authorized users and their associated identity information, authenticating their access to the Science Platform, and determining which services they are permitted to use.
This tech note collects decisions, analysis, and trade-offs made in the implementation of the system that may be of future interest.
It also collects a list of intended future work.
The historical background here may be useful for understanding design and implementation decisions, but would clutter other documents and distract from the details of the system as implemented.

.. note::

   This is part of a tech note series on identity management for the Rubin Science Platform.
   The other two primary documents are DMTN-234_, which describes the high-level design; and DMTN-224_, which describes the implementation.
   See the `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ for a complete list of related documents.

.. _DMTN-234: https://dmtn-234.lsst.io/
.. _DMTN-224: https://dmtn-224.lsst.io/

Identity
========

Federated identity
------------------

We considered other approaches to accepting federated identity besides using CILogon_ and COmanage_.

.. _CILogon: https://www.cilogon.org/
.. _COmanage: https://www.incommon.org/software/comanage/

Supporting InCommon directly was rejected as too complex; direct support of federated authentication is complex and requires a lot of ongoing maintenance work.

There are several services that provide federated identity as a service.
Most of them charge per user.
Given the expected number of users of the eventual production Science Platform, CILogon and its COmanage service appeared to be the least expensive option.
It also builds on a pre-existing project relationship and uses a service run by a team with extensive experience supporting federated authentication for universities and scientific collaborations.

Subsequent to that decision, we became aware of Auth0_ and its B2C authentication service, which appears to be competitive with CILogon on cost and claims to also support federated identity.
We have not done a deep investigation of that alternative.

.. _Auth0: https://auth0.com/

We considered using GitHub rather than InCommon as an identity source, and indeed used GitHub for some internal project deployments and for the data preview releases through DP0.2.
However, not every expected eventual user of the Science Platform will have a GitHub account, and GitHub lacks COmanage's support for onboarding flows, approval, and self-managed groups.
We also expect to make use of InCommon as a source of federated identity since it supports many of our expected users, and GitHub does not provide easy use of InCommon as a source of identities.

GitHub
------

In the original GitHub integration, we used the token returned by GitHub from the OAuth 2.0 authorization flow to retrieve metadata from the user's account and then discarded it.

However, only the organization and team membership at the time the user authorizes the OAuth App are passed to that app.
The authorization is then remembered, and if the user authenticates to the same Science Platform again, they are not prompted again for what organization data to release.
Instead, the prior authorization is reused, including the old organization data, and thus new team memberships are not picked up by the Science Platform.
This means that even if the user logs out (destroying their Gafaelfawr session token and cookie) and logs back in, Gafaelfawr will not see any new organization information because the user will not have released it from the authorization screen.

We solved this problem by retaining the GitHub token for the user's authentication by storing it in the user's encrypted Gafaelfawr session cookie.
If the user explicitly logs out, that token is retrieved and used to revoke the user's OAuth App authorization.
This forces the user back to the OAuth App authorization screen the next time they log in, which in turn causes GitHub to release updated organization information (subject to organization data visibility).

Prior authorizations with incomplete information may still be reused if the user never explicitly logs out, only lets their cookies expire, but at least we can document that explicitly logging out and logging back in will fix any missing organizational data.

It's common for clients of the GitHub API to not worry about pagination, since most requests don't result in responses that are long enough to be paginated.
The original Gafaelfawr OAuth 2.0 client code did not support pagination when retrieving information about the user.
However, the user's list of teams is often long enough that GitHub will paginate it.
If the client does not retrieve all pages, users may be mysteriously denied access for inobvious reasons.
We learned this lesson the hard way, and Gafaelfawr now correctly supports pagination of the team list.

COmanage
========

After choosing COmanage as the user identity store, we had to make several decisions about how to configure it, what identity management features it should provide, and what features we should implement external to it.

Enrollment flow
---------------

It's possible to then configure a return URL to which the user goes after enrollment is complete, but that's probably not that useful when we're using an approval flow.

We will need to customize the email messages and web pages presented as part of the approval flow.
This has not yet been done.

It's not clear yet whether we will need to automate additional changes to a person's record after onboarding, such as adding them to groups, or if this will be handled manually during the approval process.
If we do need to automate this, we may need to do that via the COmanage API.

Email verification issue
^^^^^^^^^^^^^^^^^^^^^^^^

Currently, user onboarding has a bug: After choosing their name, email, and username, the user is sent an email message to confirm that they have control over that email address.
The link in the mail message has a one-time code in it, and confirms the email address when followed.
However, sites with anti-virus integrated with their email system (such as AURA) often pre-fetch all URLs seen in email addresses.
Since no authentication or confirmation is required when following the link, this means that any email address at such a domain is automatically confirmed without any human interaction, posing both a security flaw and a UI problem because the user will get a confusing error message when they follow that link manually.

We will need to work with the COmanage maintainers to either require authentication to confirm the email address or to require a button that one has to click rather than doing the confirmation automatically.

User approval
^^^^^^^^^^^^^

COmanage does not preserve the affiliation information sent by the identity provider, if any.
Affiliation in COmanage must be set to one of a restricted set of values, and the affiliation given by identity providers is free-form.
In our test instance, the affiliation was forced to always be "affiliate" to avoid this problem.

If we want to make use of the affiliation sent by the upstream identity provider for approval decisions, we will have to write a COmanage plugin.
The difficult part of that is defining what the business logic should be.

To see the affiliation attributes sent by an identity provider, go directly to CILogon_ and log on via that provider.
On the resulting screen, look at the User Attributes section.

Group management
----------------

We had two primary options for managing groups via COmanage: using COmanage Registry groups, or using Grouper_.
In both cases, there are limitations on how much we can customize the UI without a lot of development.

.. _Grouper: https://spaces.at.internet2.edu/display/Grouper/Grouper+Wiki+Home

Quota calculation is not directly supported with either system and in either case would need custom development (either via a plugin or via a service that used the group API).
Recording quota information for groups locally and using the group API (or LDAP) to synchronize the list of groups with the canonical list looks like the easiest path.

COmanage Registry groups
^^^^^^^^^^^^^^^^^^^^^^^^

(This is the option that we chose.)

Advantages:

.. rst-class:: compact

#. Uses the same UI as the onboarding and identity management process
#. Possible (albeit complex) to automatically generate GIDs using ``voPosixGroup`` (see :ref:`voPosixGroup <voposixgroup>`)

Disadvantages:

.. rst-class:: compact

#. No support for nested groups
#. Groups cannot own other groups
#. No support for set math between groups
#. No generic metadata support, so group quotas would need to be maintained separately (presumably by a Rubin-developed service)
#. There currently is a rendering bug that causes each person to show up three times when editing the group membership, but this will be fixed in the 4.0.0 release due in the second quarter of 2021

Grouper
^^^^^^^

Advantages:

.. rst-class:: compact

#. Full support for nested groups
#. Groups can own other groups
#. Specializes in set math between groups if we want to do complex authorization calculations
#. Arbitrary metadata can be added to groups via the API, so we could use Grouper as our data store rather than a local database

Disadvantages:

.. rst-class:: compact

#. More complex setup and data flow
#. Users have to interact with two UIs, the COmanage one for identities and the Grouper UI for group management
#. No support for automatic GID generation

Grouper supports a REST API.
However, it appears to be very complex and documented primarily as a Java API.
We were unable to locate a traditional REST API description for it.
The API looks to be fully functional but it makes a number of unusual choices, such as ``T`` and ``F`` strings instead of proper booleans.

Using the API appears to require a lot of reverse engineering from example traces.
See, for instance, the `example of assigning an attribute value to a group <https://github.com/Internet2/grouper/blob/master/grouper-ws/grouper-ws/doc/samples/assignAttributesWithValue/WsSampleAssignAttributesWithValueRestLite_json.txt>`__.

A sample Grouper API call:

.. code-block:: console

   $ curl --silent -u GrouperSystem:XXXXXXXX \
     'https://group-registry-test.lsst.codes/grouper-ws/servicesRest/json/v2_5_000/groups/etc%3Asysadmingroup/members' \
     | jq .

We didn't investigate this further since we decided against using Grouper for group management.

.. _gid:

Numeric GIDs
------------

Getting numeric GIDs into the LDAP entries for each group isn't well-supported by COmanage.
The LDAP connector does not have an option to add arbitrary group identifiers to the group LDAP entry.

We decided to avoid this problem by assigning UIDs and GIDs outside of COmanage using `Google Firestore`_.
Here are a few other possible options we considered.

.. _Google Firestore: https://cloud.google.com/firestore

COmanage group REST API
^^^^^^^^^^^^^^^^^^^^^^^

Arbitrary identifiers can be added to groups, so a group can be configured with an auto-incrementing unique identifier in the same way that we do for users, using a base number of 200000 instead of 100000 to keep the UIDs and GIDs distinct (allowing the UID to be used as the GID of the primary group).
Although that identifier isn't exposed in LDAP, it can be read via the COmanage REST API using a URL such as::

    https://<registry-url>/registry/identifiers.json?cogroupid=7

The group ID can be obtained from the ``/registry/co_groups.json`` route, searching on a specific ``coid``.
Middleware running on the Rubin Science Platform could cache the GID information for every group, refresh it periodically, and query for the GID of a new group when seen.

.. _voposixgroup:

voPosixGroup
^^^^^^^^^^^^

Another option is to enable ``voPosixGroup`` and generate group IDs that way.
However, that process is somewhat complex.

COmanage Registry has the generic notion of a `Cluster <https://spaces.at.internet2.edu/display/COmanage/Clusters>`__.
A Cluster is used to represent a CO Person's accounts with a given application or service.

Cluster functionality is implemented by Cluster Plugins.
Right now there is one Cluster Plugin that comes out of the box with COmanage, the `UnixCluster plugin <https://spaces.at.internet2.edu/display/COmanage/Unix+Cluster+Plugin>`__.

The UnixCluster plugin is configured with a "GID Type."
From the documentation: "When a CO Group is mapped to a Unix Cluster Group, the CO Group Identifier of this type will be used as the group's numeric ID."
CO Person can then have a UnixCluster account that has associated with it a UnixCluster Group, and the group will have a GID identifier.

To have the information about the UnixCluster and the UnixCluster Group provisioned into LDAP using the ``voPosixAccount`` objectClass, define a `CO Service <https://spaces.at.internet2.edu/display/COmanage/Registry+Services>`__ for the UnixCluster.
In that configuration you need to specify a "short label", which will become value for an LDAP attribute option.
Since the ``voPosixAccount`` objectClass attributes are multi-valued, you can represent multiple "clusters," and they are distinguised by using that LDAP attribute option value.
For example::

    dn: voPersonID=LSST100000,ou=people,o=LSST,o=CO,dc=lsst,dc=org
    sn: KORANDA
    cn: SCOTT KORANDA
    objectClass: person
    objectClass: organizationalPerson
    objectClass: inetOrgPerson
    objectClass: eduMember
    objectClass: voPerson
    objectClass: voPosixAccount
    givenName: SCOTT
    mail: SKORANDA@CS.WISC.EDU
    uid: http://cilogon.org/server/users/2604273
    isMemberOf: CO:members:all
    isMemberOf: CO:members:active
    isMemberOf: scott.koranda UnixCluster Group
    voPersonID: LSST100000
    voPosixAccountUidNumber;scope-primary: 1000000
    voPosixAccountGidNumber;scope-primary: 1000000
    voPosixAccountHomeDirectory;scope-primary: /home/scott.koranda

This reflects a CO Service for the UnixAccount using the short label "primary."
With a second UnixCluster and CO Service with short label "slac" to represent an account at SLAC, this record would have additionally::

    voPosixAccountGidNumber;scope-slac: 1000001

The UnixCluster object and UnixCluster Group objects and all the identifiers are usually established during an enrollment flow.

Grouper
^^^^^^^

Grouper does not have built-in support for assigning numeric GIDs to each group out of some range.
It is possible to cobble something together using the ``idIndex`` that Grouper generates (see `this discussion <https://lists.internet2.edu/sympa/arc/grouper-users/2017-01/msg00087.html>`__ and `this documentation <https://spaces.at.internet2.edu/display/Grouper/Integer+IDs+on+Grouper+objects>`__), but it would require some development.

Alternately, groups can be assigned arbitrary attributes that we define, so we can assign GIDs to groups via the API, but we would need to maintain the list of available GIDs and ensure there are no conflicts.
Grouper also does not appear to care if the same attribute value is assigned to multiple groups, so we would need to handle uniqueness.

Custom development
^^^^^^^^^^^^^^^^^^

We could enhance (or pay someone to enhance) the LDAP Provisioning Plugin to allow us to express an additional object class in the group tree in LDAP, containing a numeric GID identifier.

Authentication
==============

User self groups
----------------

Each user will appear to the Rubin Science Platform to also be the sole member of a group with the same name as the username and the same GID as the UID.
This is a requirement for POSIX file systems underlying the Notebook Aspect and for the Butler service (see DMTN-182_ for the latter).

.. _DMTN-182: https://dmtn-182.lsst.io/

These groups will not be managed in COmanage or Grouper.
They will be synthesized by Gafaelfawr in response to queries about the user.
This work is not yet done.

HTTP Basic Authentication
-------------------------

The protocol for HTTP Basic Authentication, using ``x-oauth-basic`` as either the username or password along with the token, is reportedly based on GitHub support for HTTP Basic Authentication.
GitHub currently appears to recognize tokens wherever they're put and does not require the ``x-oauth-basic`` string.
(This would likely be wise for Gafaelfawr to do as well, but it has not yet been implemented.)

The password is probably the better place to put the token in HTTP Basic Authentication, since software will know to protect or obscure it, but common practice in other APIs that support using tokens for HTTP Basic Authentication is to use the username.
Gafaelfawr therefore supports both.
As a fallback, if neither username nor password is ``x-oauth-basic``, it assumes the username is the token, but this is not documented (except here) since we'd prefer users not use it.

OpenID Connect flow
-------------------

Currently, when Gafaelfawr acts as an OpenID Connect provider, it does not do any access control and does not check the scopes of the token.
It relies entirely on the service initiating the OpenID Connect flow to do authorization checks.

Each OpenID Connect client must be configured with a client ID and secret in an entry in a JSON blob in the Gafaelfawr secret.
It would be possible to add a list of required scopes to that configuration and check the authenticating token against those scopes during the OpenID Connect authentication.
If the user's scopes are not sufficient, Gafaelfawr could reject the authentication with an error.

The configuration of OpenID Connect clients is currently rather obnoxious, since it requires manipulating a serialized JSON blob inside the Gafaelfawr secret.
It would be nice to have a better way of configuring the client IDs and any supporting configuration, such as a list of scopes, and associating them with client secrets kept in some secure secret store.

InfluxDB tokens
---------------

Gafaelfawr contains support for minting authentication tokens for InfluxDB 1.x.
This version of InfluxDB_ expects a JWT (using the ``HS256`` algorithm) created with a symmetric key shared between the InfluxDB server and the authentication provider.

.. _InfluxDB: https://www.influxdata.com/

InfluxDB 2.0 dropped this authentication mechanism, so we do not expect to continue using it indefinitely.
It therefore isn't mentioned in the design or implementation documents.

Storage
=======

Gafaelfawr stores data in both a SQL database and in Redis.
Use of two separate storage systems is unfortunate extra complexity, but Redis is poorly suited to store relational data about tokens or long-term history, while PostgreSQL is poorly suited for quickly handling a high volume of checks for token validity.

Token API
=========

The token API design follows the recommendations in `Best Practices for Designing a Pragmatic RESTful API`_.
This means, among other implications:

- Identifiers are used instead of URLs
- The API does not follow HATEOAS_ principles
- The API does not attempt to be self-documenting (see the OpenAPI-generated documentation instead)
- Successful JSON return values are not wrapped in metadata
- ``Link`` headers are used for pagination

.. _HATEOAS: https://en.wikipedia.org/wiki/HATEOAS

See that blog post for more reasoning and justification.
See :ref:`References <references>` for more research links.

All URLs for the REST API for token manipulation start with ``/auth/api/v1``.

The API is divided into two parts: routes that may be used by an individual user to manage and view their own tokens, and routes that may only be used by an administrator.
Administrators are defined as users with authentication tokens that have the ``admin:token`` scope.
The first class of routes can also be used by an administrator and, unlike an individual user, an administrator can specify a username other than their own.

There is some minor duplication in routes (``/auth/api/v1/tokens`` and ``/auth/api/v1/users/{username}/tokens`` and similarly for token authentication and change history).
This was done to simplify the security model.
Users may only use the routes under the ``users`` collection with their own username.
The routes under ``/tokens`` and ``/history`` allow searching for any username, creating tokens for any user, and seeing results across all usernames.
They are limited to administrators.
This could have instead been enforced in more granular authorization checks on the more general routes, but this approach seemed simpler and easier to understand.
It also groups all of a user's data under ``/users/{username}`` and is potentially extensible to other APIs later.

Token UI
========

We considered serving the token UI using server-rendered HTML and a separate interface from the API, but decided against it for two reasons.
First, having all changes made through the API (whether by API calls or via JavaScript) ensures that the API always has parity with the UI, ensures that every operation can be done via an API, and avoids duplicating some frontend code.
Second, other Rubin-developed components of the Science Platform are using JavaScript with a common style dictionary to design APIs, so building the token UI using similar tools will make it easier to maintain a standard look and feel.

For the initial release, the token UI was included with Gafaelfawr.
It was written in JavaScript using React_ and minimized using Gatsby_.
Gatsby is probably overkill for this small JavaScript UI, but was used because it was also used in other SQuaRE development.

.. _React: https://reactjs.org/
.. _Gatsby: https://www.gatsbyjs.com/

Shipping the UI with Gafaelfawr turned out to be awkward, requiring a lot of build system work and noise from updating JavaScript dependencies.
It also made it harder to give it a consistent style and integrate it properly with the rest of the Science Platform UI.
The plan, therefore, is to move the logic of the UI into another Science Platform JavaScript UI service (possibly the one that provides the front page of the Science Platform) and remove the UI that's shipped with the Gafaelfawr Python application.

.. _remaining:

Remaining work
==============

The following requirements should be satisfied by the Science Platform identity management system, but are not yet part of the design.
The **IDM-XXXX** references are to requirements listed in SQR-044_, which may provide additional details.

.. _SQR-044: https://sqr-044.lsst.io/

.. rst-class:: compact

- Implement user self-groups (groups with the same name as the username)
- Use multiple domains to control JavaScript access and user cookies
- Filter out the token from ``Authorization`` headers of incoming requests
- Restrict OpenID Connect authentication by scope
- Force two-factor authentication for administrators (IDM-0007)
- Force reauthentication to provide an affiliation (IDM-0009)
- Changing usernames (IDM-0012)
- Handling duplicate email addresses (IDM-0013)
- Disallow authentication from pending or frozen accounts (IDM-0107)
- Logging of COmanage changes to users (IDM-0200)
- Logging of authentications via Kafka to the auth history table (IDM-0203)
- Authentication history per federated identity (IDM-0204)
- Last used time of user tokens (IDM-0205)
- Email notification of federated identity and user token changes (IDM-0206)
- Freezing accounts (IDM-1001)
- Deleting accounts (IDM-1002)
- Setting an expiration date on an account (IDM-1003, IDM-1301)
- Notifying users of upcoming account expiration (IDM-1004)
- Notifying users about email address changes (IDM-1101)
- User class markers (IDM-1103, IDM-1310)
- Quotas (IDM-1200, IDM-1201, IDM-1202, IDM-1203, IDM-1303, IDM-1401, IDM-1402, IDM-2100, IDM-2101, IDM-2102, IDM-2103, IDM-2201, IDM-3003)
- Administrator verification of email addresses (IDM-1302)
- User impersonation (IDM-1304, IDM-1305, IDM-2202)
- Review newly-created accounts (IDM-1309)
- Merging accounts (IDM-1311)
- Logging of administrative actions tagged appropriately (IDM-1400, IDM-1403, IDM-1404)
- Affiliation-based groups (IDM-2001)
- Group name restrictions (IDM-2004)
- Expiration of group membership (IDM-2005)
- Group renaming while preserving GID (IDM-2006)
- Correct handling of group deletion (IDM-2007)
- Groups owned by other groups (IDM-2009)
- Logging of group changes (IDM-2300, IDM-2301, IDM-2302, IDM-2303, IDM-2304, IDM-2305, IDM-4002)
- API to COmanage (IDM-3001)
- Scale testing (IDM-4000)
- Scaling of group membership (IDM-4001)

.. _references:

References
==========

The `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ lists all of the identity management tech notes.
This is a list of additional references to standards and blog discussions that were useful in development the design and implementation.

Blog posts
----------

`Best Practices for Designing a Pragmatic RESTful API`_
    An excellent and opinionated discussion of various areas of RESTful API design that isn't tied to any specific framework or standard.

`Five ways to paginate in Postgres`_
    A discussion of tradeoffs between pagination techniques in PostgreSQL, including low-level database performance and PostgreSQL-specific features.

`JSON API, OpenAPI and JSON Schema Working in Harmony`_
    Considerations for which standards to use when designing a JSON REST API.

`The Benefits of Using JSON API`_
    An overview of JSON:API with a comparison to GraphQL.

.. _Best Practices for Designing a Pragmatic RESTful API: https://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api
.. _Five ways to paginate in Postgres: https://www.citusdata.com/blog/2016/03/30/five-ways-to-paginate/
.. _JSON API, OpenAPI and JSON Schema Working in Harmony: https://apisyouwonthate.com/blog/json-api-openapi-and-json-schema-working-in-harmony
.. _The Benefits of Using JSON API: https://nordicapis.com/the-benefits-of-using-json-api/

Standards
---------

`FastAPI`_
    The documentation for the FastAPI Python framework.

`JSON:API`_
    The (at the time of this writing) release candidate for the upcoming JSON:API 1.1 specification.

OpenAPI_
    The OpenAPI specification for RESTful APIs.
    Provides a schema and description of an API and supports automatic documentation generation.
    Used by FastAPI_.

`RFC 7807`_
    This document defines a "problem detail" as a way to carry machine-readable details of errors in a HTTP response to avoid the need to define new error response formats for HTTP APIs.

`RFC 8288`_
    This specification defines a model for the relationships between resources on the Web ("links") and the type of those relationships ("link relation types").
    It also defines the serialisation of such links in HTTP headers with the Link header field.

.. _FastAPI: https://fastapi.tiangolo.com/
.. _JSON:API: https://jsonapi.org/format/1.1/
.. _OpenAPI: https://swagger.io/specification/
.. _RFC 7807: https://tools.ietf.org/html/rfc7807
.. _RFC 8288: https://tools.ietf.org/html/rfc8288
