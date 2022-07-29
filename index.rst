:tocdepth: 1

.. sectnum::

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

Some deployments of the Science Platform must support federated user authentication via SAML alongside other common authentication methods such as OAuth 2.0 (GitHub) and OpenID Connect (Google).
We considered other approaches to accepting federated identity besides using CILogon_ and COmanage_.

.. _CILogon: https://www.cilogon.org/
.. _COmanage: https://www.incommon.org/software/comanage/

Running a SAML Discovery Service and integrating with the various authentication federations was rejected as too complex and requiring too much ongoing work.
Other services already specialize in this; staffing people for the Science Platform to maintain such a system is not a good use of observatory resources.

There are several services that provide federated identity as a service.
Most of them charge per user.
Given the expected number of users of the eventual production Science Platform, CILogon and its COmanage service appeared to be the least expensive option.
It also builds on a pre-existing project relationship and uses a service run by a team with extensive experience supporting federated authentication for universities and scientific collaborations.

We considered using GitHub rather than InCommon as an identity source, and indeed used GitHub for some internal project deployments and for the data preview releases through DP0.2.
However, not every expected eventual user of the Science Platform will have a GitHub account, and GitHub lacks COmanage's support for onboarding flows, approval, and self-managed groups.
We also expect to make use of InCommon as a source of federated identity since it supports many of our expected users, and GitHub does not provide easy use of InCommon as a source of identities.

Vendor evaluations of CILogon and GitHub as identity providers for the Science Platform can be found in SQR-045_ and SQR-046_, respectively.

.. _SQR-045: https://sqr-045.lsst.io/
.. _SQR-046: https://sqr-046.lsst.io/

Subsequent to that decision, we became aware of Auth0_ and its B2C authentication service, which appears to be competitive with CILogon on cost and claims to also support federated identity.
We have not done a deep investigation of that alternative.

.. _Auth0: https://auth0.com/

No process for handling of changes in institutional affiliation has been defined.
This will need to be addressed before production deployment.
Suppose, for instance, a user has access via the University of Washington, and has also configured GitHub as an authentication provider because that's more convenient for them.
Now suppose the user's affiliation with the University of Washington ends and observatory data rights policy says they should no longer have access.
If the user continues to authenticate via GitHub, the Science Platform needs to recognize this affiliation change and implement it via changed group memberships or changed scopes (or possibly suspending the account entirely).
This mechanism should also handle user tokens the user has previously created, including invalidating them if necessary.
The design currently has no mechanism for this.

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

Federated authentication
------------------------

The current identity management design assumes all services protected by the identity management system run within the same Kubernetes cluster.
While it is possible to create a token for one cluster and use it in another cluster, this use case is not optimized and is assumed to be uncommon.
Each Science Platform deployment has its own independent instantiation of this authentication system, without any cross-realm authentication.

The obvious drawback of this approach compared to a federated token design is that there are some cases where authenticated communication between deployments would be useful.
One example would be a Science Platform deployment with a partial set of services and a system for delegating requests for missing services to another, more complete deployment.
These use cases are not well-supported by the current design.

In return, this design provides massive improvements in simplicitly and ease of understanding.
It enables the use of opaque tokens backed by a centralized data store (see :ref:`Token format <token-format>`, which in turn provides simple revocation and short tokens.
It limits the scope of compromise of the identity management system to a single deployment.
It also avoids the numerous complexities around token lifetime, management, logging, and concepts of identity inherent in a federated design.
For example, different deployments of the Science Platform are likely to have different sets of authorized users, which would have to be taken into account for cross-cluster authentication and authorization.

Forced multifactor authentication
---------------------------------

Ideally, we would like to force multifactor authentication for administrators to make it harder for a single password compromise to compromise the entire Science Platform.
Unfortunately, Google and GitHub do not expose this information in their OAuth metadata, and therefore it's hard to know how someone authenticated when they came through CILogon (which will always be the case for a deployment using federated identity).

Two possible approaches to consider (neither of which have been implemented):

- Use a separate authentication path for administrators that forces use of a specific Google Cloud Identity domain with appropriate multifactor authentication requirements.
  This would require implementing Google authentication directly in Gafaelfawr and supporting two configured authentication methods for the same deployment, which is somewhat unappealing for complexity reasons.

- Address the issue via policy.
  In order to be added to the administrators group in COmanage (the one that maps to an ``admin:token`` scope, or other similar privileged scopes), require that all configured sources of identity use multifactor authentication.
  We probably couldn't enforce this programmatically, since the administrator could add another source of identity and it would be hard to know that this has happened, but that may not be necessary.
  One variation on this approach that's worth considering is to restrict the most privileged access to a second account (conventionally, ``<username>-admin``) kept separate from regular day-to-day use and testing of the Science Platform.

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

The current enrollment approach relies solely on the "Self Signup with Approval" flow, but an invitation flow may make more sense in some cases since it allows pre-approval of the user.
Currently, the user has to be told to go through the signup process and then the approver has to check back once this has been done and finish the approval, which requires an additional point of coordination.

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

.. _token-format:

Token format
------------

There are four widely-deployed choices for API authentication:

#. HTTP Basic with username and password
#. Opaque bearer tokens
#. :abbr:`JWTs (JSON Web Tokens)`
#. Client TLS certificates

The first two are roughly equivalent except that HTTP Basic imposes more length restrictions on the authenticator, triggers browser prompting behavior, and has been replaced by bearer token authentication in general best practices for web services.
Client TLS certificates provide the best theoretical security since they are not vulnerable to network interception of credentials, but are more awkward to manage on the client side and cannot be easily cut-and-pasted.
Client TLS certificates also cannot be used in HTTP Basic fallback situations with software that only supports that authentication mechanism.

Opaque bearer tokens and JWTs are therefore the most appealing.
The same token can then be used via HTTP Basic as a fallback for some legacy software that only understands that authentication mechanism.

JWTs are standardized and widely supported by both third-party software and by libraries and other tools, and do not inherently require a backing data store since they contain their own verification information.
However, JWTs are necessarily long.
An absolutely minimal JWT (only a ``sub`` claim with a single-character identity) using the ``ES256`` algorithm to minimize the signature size is 181 octets.
With a reasonable set of claims for best-practice usage (``aud``, ``iss``, ``iat``, ``exp``, ``sub``, ``jti``, and ``scope``), again using the ``ES256`` algorithm, a JWT containing only identity and scope information and no additional metadata is around 450 octets.

Length matters because HTTP requests have to pass through various clients, libraries, gateways, and web servers, many of which impose limits on HTTP header length, either in aggregate or for individual headers.
Multiple services often share the same cookie namespace and compete for those limited resources.

These constraints become more severe when supporting HTTP Basic.
The username and password fields of the HTTP Basic ``Authorization`` header are often limited by implementations to 256 octets.
Some software imposes limits as small as 64 octets under the assumption that these fields only need to hold traditional, short usernames and passwords.

Even minimal JWTs are therefore dangerously long, and best-practice JWTs are too long to use with HTTP Basic authentication.

Opaque bearer tokens avoid this problem.
An opaque token need only be long enough to defeat brute force searches, for which 128 bits of randomness are sufficient.
For various implementation reasons, it is desirable to have a random token ID and a separate random secret and to add a standard prefix to all opaque tokens, but even with this taken into account, a token with a four-octet identifying prefix and two 128-bit random segments, encoded in URL-safe base64 encoding, is only 49 octets.

The HTTP Basic requirement only applies to the request from the user to the authentication gateway for the Science Platform.
The length constraints similarly matter primarily for the HTTP Basic requirement and for authentication from web browsers, which may have a multitude of cookies and other necessary headers.
It would therefore be possible to use JWTs inside the Science Platform and only use opaque tokens outside.
However, this adds complexity by creating multiple token systems.
It would also be harder to revoke specific JWTs, should that be necessary for security reasons.
A single token mechanism based on opaque bearer tokens, where each token maps to a corresponding session stored in a persistent data store, achieves the authentication goals with a minimum of complexity.

This choice forgoes the following advantages of using JWTs internally:

- Some third-party services may consume JWTs directly and expect to be able to validate them.
  Gafaelfawr therefore had to implement OpenID Connect authentication (with separate JWT tokens) as an additional authentication flow unrelated to the token authentication system used by most routes.
  However, this implementation can be minimal and is limited in scope to only Science Platform services that require OpenID Connect (which are expected to be a small subset of services and may not be required in the federated identity deployment case at all).

- If a user API call sets off a cascade of numerous internal API calls, avoiding the need to consult a data store to validate opaque tokens could improve performance.
  JWTs can be verified directly without needing any state other than the (relatively unchanging) public signing key.
  In practice, however, Redis appears to be fast enough that this is not a concern.

- JWTs are apparently becoming the standard protocol for API web authentication.
  Preserving a JWT component to the Science Platform will allow us to interoperate with future services, possibly outside the Science Platform, that require JWT-based authentication.
  It also preserves the option to drop opaque bearer tokens entirely if the header length and HTTP Basic requirements are relaxed in the future (by, for example, no longer supporting older software with those limitations).

The primary driver for using opaque tokens rather than JWTs is length, which in turn is driven by the requirement to support HTTP Basic authentication.
If all uses of HTTP Basic authentication can be shifted to token authentication and that requirement dropped, the decision to use opaque tokens rather than JWTs could be revisited.
However, using short tokens still provides benefits for each cut and paste of tokens, and provides a simple and reliable revocation mechanism.

Closely related to this decision is to (where possible) dynamically look up group membership rather than storing it with (or in) the authentication token.
The primary advantage of storing group membership and other authorization information in the token is faster access to the data: the authorization information can be retrieved without querying an external source.
Token scopes, for example, are stored with the token to make use of this property.
But group membership is often dynamic, and users may not want to (and will be confused by having to) revoke their token and recreate it to see changes to their access.
The current approach uses a compromise of dynamic group membership, static scopes tied to the token, and a five-minute cache to avoid excessive load on the underlying group system and excessive query latency in Gafaelfawr.

Notebook Aspect notebooks will still likely have to be relaunched to pick up new or changed group memberships, since the user's GIDs are determined when the notebook pod is launched.

Token scopes
------------

For user-created API tokens, there will be a balance between the security benefit of more restricted-use tokens and the UI complexity of giving the user a lot of options when creating a token.
The balance the identity management design strikes is to reserve scopes for controlling all access to a particular service, or controlling admin access to a service versus regular access.
Controlling access to specific data sets within the service is done with groups, not scopes.

This appears to strike a reasonable balance between allowing users and service configuration to limit the access of delegated tokens, and avoiding presenting the user with too many confusing options when creating a new token.
This policy is discussed further in DMTN-235_.

.. _DMTN-235: https://dmtn-235.lsst.io/

HTTP Basic Authentication
-------------------------

The protocol for HTTP Basic Authentication, using ``x-oauth-basic`` as either the username or password along with the token, is reportedly based on GitHub support for HTTP Basic Authentication.
GitHub currently appears to recognize tokens wherever they're put and does not require the ``x-oauth-basic`` string.
(This would likely be wise for Gafaelfawr to do as well, but it has not yet been implemented.)

The password is probably the better place to put the token in HTTP Basic Authentication, since software will know to protect or obscure it, but common practice in other APIs that support using tokens for HTTP Basic Authentication is to use the username.
Gafaelfawr therefore supports both.
As a fallback, if neither username nor password is ``x-oauth-basic``, it assumes the username is the token, but this is not documented (except here) since we'd prefer users not use it.

User self groups
----------------

Each user will appear to the Rubin Science Platform to also be the sole member of a group with the same name as the username and the same GID as the UID.
This is a requirement for POSIX file systems underlying the Notebook Aspect and for the Butler service (see DMTN-182_ for the latter).

.. _DMTN-182: https://dmtn-182.lsst.io/

These groups will not be managed in COmanage or Grouper.
They will be synthesized by Gafaelfawr in response to queries about the user.
This work is not yet done.

OpenID Connect flow
-------------------

Currently, when Gafaelfawr acts as an OpenID Connect provider, it does not do any access control and does not check the scopes of the token.
It relies entirely on the service initiating the OpenID Connect flow to do authorization checks.

Each OpenID Connect client must be configured with a client ID and secret in an entry in a JSON blob in the Gafaelfawr secret.
It would be possible to add a list of required scopes to that configuration and check the authenticating token against those scopes during the OpenID Connect authentication.
If the user's scopes are not sufficient, Gafaelfawr could reject the authentication with an error.

The configuration of OpenID Connect clients is currently rather obnoxious, since it requires manipulating a serialized JSON blob inside the Gafaelfawr secret.
It would be nice to have a better way of configuring the client IDs and any supporting configuration, such as a list of scopes, and associating them with client secrets kept in some secure secret store.

Currently, Gafaelfawr does not register the ``redirect_uri`` parameter from an OpenID Connect client.
As long as the client authenticates, it allows redirection to any URL within the same domain.
If the valid ``redirect_uri`` values were registered along with the client and validated against the provided ``redirect_uri``, Gafaelfawr could extend OpenID Connect support to relying parties outside of the Science Platform deployment.
This would allow chaining Gafaelfawr instances.

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

Data precendence
----------------

Older versions of Gafaelfawr used complex logic to decide whether to look for user identity information in Redis, Firestore, or LDAP depending on the overall configuration and whether the token was created via the admin API.
After some practical experience trying to maintain that logic, Gafaelfawr switched to the current model of a strict precedence hierarchy.
If the data element is in Redis, that's used by preference.
Otherwise, it's taken from Firestore, then LDAP, and if all of those fail, it's considered empty.

This model simplifies the handling of each authentication request and moves the logic for handling data sources to the login handler, where it's easy to handle.
During login, Gafaelfawr chooses whether to store user identity data in Redis based on its configuration of sources for identity information.
If the data is coming from some external source like Firestore or LDAP, it is not stored in Redis.
If it is coming from GitHub or from OpenID Connect ID token claims, it is stored in Redis.
The precedence logic will then use the right data sources for subsequent requests.

An advantage of this approach in addition to simplicity is that it allows administrators creating tokens via the token API to choose whether they want to override external data sources.
If they specify identity information for the token, it's stored in Redis and overrides external sources.
Otherwise, external sources will be used as configured.

Cookies
-------

Authentication cookies are stored as session cookies, rather than as cookies with an expiration tied to the lifetime of the user's credentials.
The latter is, on the surface, a more obvious design, but setting an expiration time on a cookie means the cookie is persisted to disk across browser sessions.
Session cookies are slightly more secure because they are not persisted to disk outside of the session recovery code, and are deleted when the user closes their browser.
They have the drawback of therefore sometimes requiring more frequent reauthentication.

More importantly, the identity management system needs to store various other information, such as login state, that does not have an obvious expiration time.
The token and the other information could be divided into separate cookies, but that adds complexity with little benefit.

Cookies are encrypted primarily to prevent easy tampering or snooping, and because it's easy to do and has no drawbacks.
The encryption does not protect against theft of the entire cookie.
The cookie still represents a bearer token, and an attacker who gains access to the cookie can reuse that cookie from another web browser and gain access as the user.

The current design uses domain-scoped cookies and assumes the entire Science Platform deployment runs within a single domain.
This is not a good long-term assumption, since there are serious web security drawbacks to using a single domain and a single web security context.
See DMTN-193_ for more information, including a new proposed design that will likely be adopted in the future.

.. _dmtn-193: https://dmtn-193.lsst.io/

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
- Register and validate ``remote_uri`` for OpenID Connect clients, and relax the requirement that they be in the same domain
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

Standards
---------

`JSON:API`__
    The (at the time of this writing) release candidate for the upcoming JSON:API 1.1 specification.

__ https://jsonapi.org/format/1.1/

OpenAPI__
    The OpenAPI specification for RESTful APIs.
    Provides a schema and description of an API and supports automatic documentation generation.
    Used by FastAPI_.

__ https://swagger.io/specification/

`OpenID Connect Core 1.0`__
    The core specification of the OpenID Connect protocol.

__ https://openid.net/specs/openid-connect-core-1_0.html

`OpenID Connect Discovery 1.0`__
    OpenID Connect discovery mechanisms, including the specification for the metadata returned by the provider metadata endpoint.

__ https://openid.net/specs/openid-connect-discovery-1_0.html

`RFC 6749: The OAuth 2.0 Authorization Framework`__
    The specification for the OAuth 2.0 authorization framework, on top of which OpenID Connect was built.

__ https://datatracker.ietf.org/doc/html/rfc6749

`RFC 6750: Bearer Token Usage`__
    Documents the syntax for ``WWW-Authenticate`` and ``Authorization`` header fields when using bearer tokens.
    The attributes returned in a challenge in a ``WWW-Authenticate`` header field are defined here.

__ https://datatracker.ietf.org/doc/html/rfc6750

`RFC 7517: JSON Web Key (JWK)`__
    The specification of the JSON Web Key format, including JSON Web Key Sets (JWKS).

__  https://datatracker.ietf.org/doc/html/rfc7517

`RFC 7519: JSON Web Token (JWT)`__
    The core specification for the JSON Web Token format.

__ https://datatracker.ietf.org/doc/html/rfc7519

`RFC 7617: The Basic HTTP Authentication Scheme`__
    Documents the syntax for ``WWW-Authenticate`` and ``Authorization`` header fields when using HTTP Basic Authentication.

__ https://datatracker.ietf.org/doc/html/rfc7617

`RFC 7807: Problem Details for HTTP APIs`__
    Defines a "problem detail" as a way to carry machine-readable details of errors in a HTTP response.
    This avoids the need to define new error response formats for HTTP APIs.

__ https://datatracker.ietf.org/doc/html/rfc7807

`RFC 8288: Web Linking`__
    The standard for the ``Link`` HTTP header and its relation types.

__ https://datatracker.ietf.org/doc/html/rfc8288

Other documentation
-------------------

`CILogon OpenID Connect`__
    Documentation for how to use CILogon as an OpenID Connect provider.
    Includes client registration and the details of the OpenID Connect protocol as implemented by CILogon.

__ https://www.cilogon.org/oidc

`FastAPI`_
    The documentation for the FastAPI Python framework.

.. _FastAPI: https://fastapi.tiangolo.com/

`GitHub OAuth Apps`__
    How to create an OAuth App for GitHub, request authentication, and parse the results.

__ https://docs.github.com/en/developers/apps/building-oauth-apps

`GitHub Users API`__
    APIs for retrieving information about the authenticated user.
    See also `user emails <https://docs.github.com/en/rest/users/emails>`__ and `teams <https://docs.github.com/en/rest/teams>`__.

__ https://docs.github.com/en/rest/users

Blog posts
----------

`Best Practices for Designing a Pragmatic RESTful API`_
    An excellent and opinionated discussion of various areas of RESTful API design that isn't tied to any specific framework or standard.

.. _Best Practices for Designing a Pragmatic RESTful API: https://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api

`Five ways to paginate in Postgres`__
    A discussion of tradeoffs between pagination techniques in PostgreSQL, including low-level database performance and PostgreSQL-specific features.

__ https://www.citusdata.com/blog/2016/03/30/five-ways-to-paginate/

`JSON API, OpenAPI and JSON Schema Working in Harmony`__
    Considerations for which standards to use when designing a JSON REST API.

__ https://apisyouwonthate.com/blog/json-api-openapi-and-json-schema-working-in-harmony

`The Benefits of Using JSON API`__
    An overview of JSON:API with a comparison to GraphQL.

__ https://nordicapis.com/the-benefits-of-using-json-api/
