####################################################
Implementation decisions for RSP identity management
####################################################

.. abstract::

   The identity management, authentication, and authorization component of the Rubin Science Platform is responsible for maintaining a list of authorized users and their associated identity information, authenticating their access to the Science Platform, and determining which services they are permitted to use.
   This tech note collects decisions, analysis, and trade-offs made in the implementation of the system that may be of future interest.
   It also collects a list of intended future work.
   The historical background here may be useful for understanding design and implementation decisions, but would clutter other documents and distract from the details of the system as implemented.

.. note::

   This is part of a tech note series on identity management for the Rubin Science Platform.
   The other two primary documents are :dmtn:`234`, which describes the high-level design; and :dmtn:`224`, which describes the implementation.
   See the `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ for a complete list of related documents.

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

Vendor evaluations of CILogon and GitHub as identity providers for the Science Platform can be found in :sqr:`045` and :sqr:`046`, respectively.

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
It enables the use of opaque tokens backed by a centralized data store (see :ref:`token-format`), which in turn provides simple revocation and short tokens.
It limits the scope of compromise of the identity management system to a single deployment.
It also avoids the numerous complexities around token lifetime, management, logging, and concepts of identity inherent in a federated design.
For example, different deployments of the Science Platform are likely to have different sets of authorized users, which would have to be taken into account for cross-cluster authentication and authorization.

We are supporting a limited type of federated identity by exposing an OpenID Connect server that can be used to request an instance of the Science Platform authenticate users and return user metadata that can be used at other sites.
This is primarily for International Data Access Centers (IDACs).
See :dmtn:`253` for more details.

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
  However, we have already used the ``<username>-admin`` accounts in the lsst.cloud Google Cloud Identity domain as the identities for administrative access to COmanage, and COmanage doesn't support using the same identity for both administering COmanage itself and being a member of a COmanage organization.

COmanage
========

After choosing COmanage as the user identity store, we had to make several decisions about how to configure it, what identity management features it should provide, and what features we should implement external to it.
See :sqr:`055` for the details of the current COmanage configuration.

Enrollment flow
---------------

In order to save work for the approver, all users are automatically added to the general users group (``g_users``) when approved.
Additional group memberships must be added manually by the approver or by some other group owner.

The current enrollment approach relies solely on the "Self Signup with Approval" flow, but an invitation flow may make more sense in some cases since it allows pre-approval of the user.
Currently, the user has to be told to go through the signup process and then the approver has to check back once this has been done and finish the approval, which requires an additional point of coordination.
We have made extensive customizations of the "Self Signup with Approval" flow, which would need to be duplicated in any other flow we decided to use.

Names
^^^^^

Ideally, we would prompt for two names: the nickname by which the person wants to be known, and the full name they use in professional contexts for matching and approval purposes.

We do not want to parse either name into components.
This creates `tons of cultural problems <https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/>`__.

Unfortunately, COmanage's data model requires names be broken into components and doesn't have a data model that neatly matches this design goal.
For now, we are restricting the name fields to given and family name, making family name optional, and adding explanatory text asking people to use the name they use in professional contexts.

We may revisit this later.

Email verification
^^^^^^^^^^^^^^^^^^

By default, COmanage confirms email addresses by sending an email message with a link which, when visited, confirms that the user can receive email at that address.
This approach has problems with email anti-virus systems that retrieve all links in incoming messages to check them for malware.
That anti-virus check will automatically confirm the email address with no user interaction required, thus defeating the point of the check.

COmanage added a configuration setting to address this, allowing one to force a confirmation page or authentication or both to confirm an address.
Initially, we tried a configuration that required the user to press a :guilabel:`Confirm` button after visiting the page.
The hope was that anti-virus systems wouldn't interact with the retrieved page, and thus won't confirm the email address with that setting.
Alas, this proved too optimistic: NASA's email server seemed to be aggressively following all links on the page and still broke the email verification.

Our current configuration therefore goes one step further and instead sends a verification code in the email message without any links.
The user then has to enter that verification code into a web form they're shown during the enrollment process in order to verify their email.

User approval
^^^^^^^^^^^^^

COmanage does not preserve the affiliation information sent by the identity provider, if any.
Affiliation in COmanage must be set to one of a restricted set of values, and the affiliation given by identity providers is free-form.
In our test instance, the affiliation was forced to always be "affiliate" to avoid this problem.

If we want to make use of the affiliation sent by the upstream identity provider for approval decisions, we will have to write a COmanage plugin.
The difficult part of that is defining what the business logic should be.

To see the affiliation attributes sent by an identity provider, go directly to CILogon_ and log on via that provider.
On the resulting screen, look at the User Attributes section.

COmanage also does not retain the GitHub username for users that use GitHub as their authentication mechanism.
For users authenticating with GitHub, essentially no information about the user except for their email address on GitHub is retrieved.
For GitHub and Google authentications, approval will likely need to be done based on information outside of COmanage.

We're using the default access control rule for approving petitions (the COmanage terminology for approving new users) and putting anyone who will be approving new users in the ``CO:admins`` group.
This means they also have access to change the configuration of the COmanage instance, which isn't desirable, but that also means they can edit people, which is desirable.

We experimented with creating a separate approvers group and modifying the enrollment flow to send approvals to that group instead, but we ran into two problems:

- The list of pending petitions normally shows up in the sidebar under People, but the People heading is hidden if the user is not in ``CO:admins``, so the only visibility of pending petitions is through the notification.
  That also requires clicking through the notification and finding the URL for the petition (or waiting for the email message).

- There is no way for the user to edit people without being a member of ``CO:admins``.
  While that isn't common, it seems like something we'll need eventually.

We've therefore stayed with putting approvers in ``CO:admins`` and asking them not to change the configuration.

We would like to whitelist anyone who can authenticate via specific US institutions or can verify an email address at one of those institutions so that they don't need to be explicitly approved, but we have so far not been able to find a good way to implement this in COmanage.

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
#. Possible (albeit complex) to automatically generate GIDs using ``voPosixGroup`` (see :ref:`voposixgroup`)

Disadvantages:

.. rst-class:: compact

#. Groups cannot own other groups
#. Very limited support for set math between groups (negative membership is supported, but not much else)
#. No generic metadata support, so group quotas would need to be maintained separately (presumably by a Rubin-developed service)
#. There currently is a rendering bug that causes each person to show up three times when editing the group membership, but this will be fixed in the 4.0.0 release due in the second quarter of 2021

Grouper
^^^^^^^

Advantages:

.. rst-class:: compact

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
  JWTs, via OpenID Connect, are also the standard way of delegating authentication to a different site.
  Gafaelfawr therefore had to implement OpenID Connect authentication (with separate JWT tokens) as an additional authentication and authorization flow unrelated to the token authentication system used by most routes.
  However, this implementation can be minimal and is limited in scope to only IDACs and Science Platform services that require OpenID Connect (which are expected to be a small subset of services and may not be required in the federated identity deployment case at all), and is also limited to only the ID token.
  The access token returned by OpenID Connect is a regular opaque token.

- If a user API call sets off a cascade of numerous internal API calls, avoiding the need to consult a data store to validate opaque tokens could improve performance.
  JWTs can be verified directly without needing any state other than the (relatively unchanging) public signing key.
  In practice, however, Redis appears to be fast enough that this is not a concern.

- JWTs are apparently becoming the standard protocol for API web authentication.
  Preserving a JWT component to the Science Platform will allow us to interoperate with future services, possibly outside the Science Platform, that require JWT-based authentication.
  It also preserves the option to drop opaque bearer tokens entirely if the header length and HTTP Basic requirements are relaxed in the future (by, for example, no longer supporting older software with those limitations).

The primary driver for using opaque tokens rather than JWTs is length, which in turn is driven by the requirement to support HTTP Basic authentication.
If all uses of HTTP Basic authentication can be shifted to token authentication and that requirement dropped, the decision to use opaque tokens rather than JWTs could be revisited.
However, using short tokens still provides minor benefits for each cut and paste of tokens, and provides a simple and reliable revocation mechanism.

Closely related to this decision is to (where possible) dynamically look up group membership rather than storing it with (or in) the authentication token.
The primary advantage of storing group membership and other authorization information in the token is faster access to the data: the authorization information can be retrieved without querying an external source.
Token scopes, for example, are stored with the token to make use of this property.
But group membership is often dynamic, and users may not want to (and will be confused by having to) revoke their token and recreate it to see changes to their access.
The current approach uses a compromise of dynamic group membership, static scopes tied to the token, and a five-minute cache to avoid excessive load on the underlying group system and excessive query latency in Gafaelfawr.

Regardless of the group membership approach taken by the authentication system, Notebook Aspect notebooks would still have to be relaunched to pick up new or changed group memberships, since the user's GIDs are determined when the notebook pod is launched and are part of the Kubernetes pod definition.

HTTP Basic Authentication
-------------------------

In the current implementation, the user can put the token in either the username or password field, and the other field is ignored.
If both fields are tokens, they must be identical or the authentication is rejected.
We considered arbitrarily picking one to prefer, but using two mismatched tokens feels like a misconfiguration that may be confusing, so diagnosing it with an error seemed better.

The password is probably the better place to put the token in HTTP Basic Authentication, since software will know to protect or obscure it, but common practice in other APIs that support using tokens for HTTP Basic Authentication is to use the username.
Gafaelfawr therefore supports both.

Previously, the user had to put ``x-oauth-basic`` in whatever field wasn't the token.
This was reportedly based on GitHub support for HTTP Basic Authentication.
However, GitHub currently recognizes tokens wherever they're put and does not require the ``x-oauth-basic`` string, so Gafaelfawr was switched to match.
This approach is easier to document and explain.

OpenID Connect flow
-------------------

Currently, when Gafaelfawr acts as an OpenID Connect provider, it does not do any access control and does not check the scopes of the token.
It relies entirely on the service initiating the OpenID Connect flow to do authorization checks.

Each OpenID Connect client must be configured with a client ID, secret, and return URL in an entry in a JSON blob in the Gafaelfawr secret.
It would be possible to add a list of required scopes to that configuration and check the authenticating token against those scopes during the OpenID Connect authentication.
If the user's scopes are not sufficient, Gafaelfawr could reject the authentication with an error.

The configuration of OpenID Connect clients is currently rather obnoxious, since it requires manipulating a serialized JSON blob inside the Gafaelfawr secret.
It would be nice to have a better way of configuring the client IDs and any supporting configuration, such as a list of scopes, and associating them with client secrets kept in some secure secret store.
It may be possible to do this with the new Phalanx secrets sync tool using a layout similar to pull secrets, or by moving the configuration other than the secrets into the main Gafaelfawr configuration and merging that with the secrets from a different source (perhaps a convention for secret names, perhaps something like pull-secret).

The implementation of the OpenID Connect protocol in Gafaelfawr is not fully conformant and doesn't support several optional features.
See the discussion in :dmtn:`253` for more details.

The original implementation returned a JWT as both the ID token and the access token, and then accepted access tokens for authentication to the userinfo endpoint.
While this worked, it's not in the spirit of the OpenID Connect protocol and would create other problems if we ever wished to issue access tokens that could be used to authenticate to services other than Gafaelfawr.
The current implementation instead creates a Gafaelfawr token of type ``oidc`` with no scopes and returns that as the access token instead, and requires the token used to authenticate to the OpenID Connect userinfo endpoint to be of type ``oidc``.
If we later want to allow these tokens to be used for other purposes, we could allow the client to request scopes and add scopes to this access token.

Currently, the ``oidc`` token does not record the OpenID Connect client to which it was issued.
This should be added, but requires extending the database schema further.

InfluxDB tokens
---------------

Gafaelfawr used to contain support for minting authentication tokens for InfluxDB 1.x.
This version of InfluxDB_ expects a JWT (using the ``HS256`` algorithm) created with a symmetric key shared between the InfluxDB server and the authentication provider.

.. _InfluxDB: https://www.influxdata.com/

We never ended up using the Gafaelfawr integration, instead using username and password because it was easier to manage across deployments.
InfluxDB 2.0 then dropped this authentication mechanism, so we removed the Gafaelfawr support.
Hopefully, future InfluxDB releases will be able to use the OpenID Connect support.

Protecting multiple domains
---------------------------

Ideally, each Science Platform service with distinct security properties should be separated into its own domain so that the services do not share JavaScript origins.
This makes various web security attacks much more difficult.
For a comprehensive discussion, see :dmtn:`193`.

The best implementation of multiple domain support would be to maintain separate authentication credentials for each domain and reauthenticate the user (generally without requiring user credential re-entry) when the user moves from one domain to another.
This adds some additional complexity but allows the credentials for each domain to be kept isolated and ensures they cannot be used for other domains, even if they somehow leak through Gafaelfawr's attempts to keep them combined to the ingress.

This requires a fairly complex authentication redirect dance between each domain used by the Science Platform, however.
It would be roughly akin to using the OpenID Connect protocol internally; while some simplifications can be applied, most of the protocol is still needed.

In the short term, to enable multiple domains before that work has been done (and, in particular, to isolate each user JupyterLab instance in a separate per-user domain, since these services have the highest risk of web security attacks), Gafaelfawr optionally implements domain-scoped cookies.
When enabled, the same authentication cookie will be sent by web browsers to the parent domain of the Science Platform and any child domains.

In this mode, the security of the DNS records for the Science Platform domain are paramount.
An attacker who can point a child domain of the Science Platform at a compromised server will be able to collect the authentication cookie for the Science Platform and then freely act as any user who visits their server.
Provided that control of DNS records is maintained, however, this provides multiple domain support at a much lower complexity cost.

In order to protect services against misconfiguration, Gafaelfawr rejects any ingress that accepts cookie authentication if cookies would not be sent to that domain, either because subdomains are not enabled and the ingress is hosted on a different domain or because the ingress is hosted on a domain that is not a subdomain of the Science Platform domain.
Any such ingress must acknowledge that cookie authentication will not be supported by explicitly disabling it by setting ``config.allowCookies`` to false in the ``GafaelfawrIngress``.
Bearer token authentication via the ``Authorization`` header will still be supported.

Authorization
=============

Token scopes
------------

For user-created API tokens, there will be a balance between the security benefit of more restricted-use tokens and the UI complexity of giving the user a lot of options when creating a token.
The balance the identity management design strikes is to reserve scopes for controlling all access to a particular service, or controlling admin access to a service versus regular access.
Controlling access to specific data sets within the service is done with groups, not scopes.

This appears to strike a reasonable balance between allowing users and service configuration to limit the access of delegated tokens, and avoiding presenting the user with too many confusing options when creating a new token.
This policy is discussed further in :dmtn:`235`.

Originally, all requested scopes for delegated tokens were also added as required scopes for access to a service.
The intent was to (correctly) prevent delegated tokens from having scopes that the user's authenticating token did not have, thus allowing the user to bypass access controls.
However, in practice this turned out to be too restrictive.
The Portal Aspect, a major use of delegated tokens, wanted to request various scopes since it could make use of them if they were available, but users who did not have those scopes should still be able to access the Portal and get restricted functionality.

The default was therefore changed so that the list of delegated scopes was an optional request.
The delegated token gets all of the requested scopes that its parent token has, but if any are missing, they're left off the scopes for the delegated token but the authentication still succeeds.
If the service wants the delegated scopes to be mandatory, it can add them to the authorization scopes so that a user must have all of the scopes or is not allowed to access the service.

Required scopes
---------------

Originally, Gafaelfawr required any authenticated ingress require at least one scope.
The goal was to prevent accidentally exposing a service to more people than intended by not specifying a scope and assuming any authenticated user should have access.

However, the Wobbly service for UWS job storage (see :sqr:`096`) wanted to use only the ``onlyServices`` option without any specific required scope.
Any application in the allow list, regardless of the scope used to protect it, should be able to use Wobbly to store UWS jobs.
Gafaelfawr was therefore modified to allow an explicit empty scope list.

In theory, an empty scope list could be restricted to ``GafaelfawrIngress`` resources that also set ``onlyServices``.
Currently, an empty scope list is always allowed, but we may make that change if we encounter problems with misuse of the empty scope list without ``onlyServices``.

User metadata
=============

OpenID Connect and LDAP
-----------------------

The initial implementation of OpenID Connect as a source of authentication supported nearly-arbitrary combinations of data from LDAP and data from the OpenID Connect ID token.
Previously, different Science Platform environments used different combinations of sources of data.
This is no longer the case; now, all deployments that use OpenID Connect get all of the user metadata from LDAP, and Gafaelfawr no longer supports getting information other than the username from the ID token.

One of the problems with getting data from the ID token is that Keycloak, a very common OpenID Connect provider, cannot provide GID information in the ID token (at least with standard LDAP configurations).
It can be configured to provide a list of groups and a list of GIDs, but not correlate the two or keep the same ordering.
Since the Science Platform relies on GIDs for correct operation, in practice direct queries to LDAP are required.

We therefore limited authentication support to only three configurations: GitHub, COmanage plus LDAP, or OpenID Connect plus LDAP.
For the last two methods, only the username from the OpenID Connect ID token is used, and all other data will be retrieved from LDAP.

GIDs
----

The initial implementation of the identity management system assigned a UID but not a primary GID, only GIDs for each group.
Instead, the Notebook Aspect blindly assumed that it could use a GID equal to the UID when spawning lab pods, and no other part of the system used a primary GID.

This approach did not work for the USDF, where UID and GID spaces overlap, and users are already assigned a primary GID by the local identity management system.
Blindly copying the UID caused lab pods to be running with unexpected GIDs that could overlap with other groups.

The concept (and data element) of a primary GID was introduced to solve this problem and added to the other types of deployments.
For GitHub and federated identity deployments, they use user private groups with a GID matching the UID, so that GID (equal to the UID) can also be made the primary GID.

We considered making the primary GID field optional, and it still formally is within the Gafaelfawr data model, but we expect to make it mandatory in the future.
The Notebook Aspect requires the primary GID field be set.

We also at first attempted to enforce a rule that every group have a GID, and groups without GIDs were ignored.
Unfortunately, CC-IN2P3's deployment using Keycloak only had a list of groups available, not GIDs, and they still needed to use those groups to calculate scopes.
We therefore made the GID optional and allowed groups without GIDs to count for scopes.
However, in practice CC-IN2P3 ended up needing the GIDs for groups for the Notebook Aspect, so this support is also expected to be removed in the future.

ForgeRock support
-----------------

The CC-IN2P3 deployment of the Science Platform uses ForgeRock Identity Management as its ultimate source of some identity information.
Originally, CC-IN2P3 wanted to avoid using LDAP and expose user metadata via Keycloak.
When we discovered that it was not possible for Keycloak to provide groups with their GIDs, Gafaelfawr implemented limited support for API calls to the ForgeRock Identity Management Server to retrieve the GID of a group.

CC-IN2P3 eventually switched to LDAP for user metadata, which is ideal since that's the mechanism used in other places that don't use GitHub.
We therefore dropped ForgeRock support in Gafaelfawr.

User private groups
-------------------

Ideally, we'd prefer to implement user private groups (where each user is a member of a group with a matching name and the same GID as the user's UID) for all deployments.
Using user private groups allows all access control to be done based on group membership, which is part of the authorization design for Butler (see :dmtn:`182`).
Unfortunately, when a local identity management system is in play, there's no good way to do this because there's no safe GID to assign to the user.
The local identity management system should also be canonical for the user's primary GID.

We therefore implement user private groups only for the federated identity case, where we control the UID and GID spaces and can reserve all the GIDs that match UIDs for user private groups and always synthesize the group, and for the GitHub case, where we blindly use the user ID as a group ID for the user private group and the primary GID.
These user-private groups are not managed in COmanage or GitHub.
They are synthesized by Gafaelfawr in response to queries about the user.
For GitHub, this is not ideal since it may conflict with a team ID and thus a regular group ID, but given the small number of users and the large ID space, we're hoping we won't have a conflict.

For deployments with a local identity management system, since the user's GIDs may have to correspond to expected GIDs for file systems maintained outside the scope of the Science Platform and requiring compatibility with other local infrastructure, we do not attempt to implement user private groups.
Either they are provided by the local identity management system, or they're not.

Ingress integration
===================

``X-Auth-Request`` headers
--------------------------

Gafaelfawr exposes some information about the user to the protected application via ``X-Auth-Request-*`` headers.
These can be requested via annotations on the NGINX ingress and then will be filled out in the headers of all relevant requests as received by the service.

The original implementation tried to expose everything Gafaelfawr knew about the user in headers: full name, UID, group membership, all of their token scopes, their client IP, the logic used to authorize them, and so forth.
We discovered in practice that no application used all of that information, and exposing some of it caused other problems.
For example, the user's full name could be UTF-8, but HTTP headers don't allow UTF-8 by default, resulting in errors from the web service plumbing of backend services.
For another example, the group data exposed was just a list of groups without GIDs, so services that needed the GIDs would need to obtain this another way anyway.

In the current implementation, all of these headers have been dropped except for ``X-Auth-Request-User`` (containing the username), ``X-Auth-Request-Email`` (if we have an email address), ``X-Auth-Request-Service`` (containing the associated service if this is an internal token), and ``X-Auth-Request-Token`` (containing a delegated token, if one was requested).
Username is the most widely used information, and some applications care only about it (for logging purposes, for example) and not any other user information.
Email is used by the Portal and may be used by other applications.
Service is used by services that take requests from other services on behalf of users and need to know which service is making the request, such as the proposed UWS storage service (:sqr:`096`).

Applications that need more information than this should request a delegated token, either notebook or internal, and then use that token to make a request to the ``user-info`` route, which will return all of the user's identity information in as JSON, avoiding the parsing and character set problems of trying to insert it into and read it out of headers.

Private ingress routes
----------------------

Gafaelfawr has two dedicated endpoints that are used by ingress-nginx_, via the ``auth-url`` annotation, to make an auth subrequest for each incoming request without cached authentication information.
One handles authenticated ingresses and one handles anonymous ingresses, which only need redaction of the ``Cookie`` and ``Authorization`` headers to avoid leaking tokens (see :dmtn:`193`).

.. _ingress-nginx: https://kubernetes.github.io/ingress-nginx/

Originally, these routes were exposed to users directly via the Gafaelfawr Kubernetes ``Ingress``.
Users were not intended to use them, but direct accesses from users were relatively harmless.
The worst that a user could do is create new notebook or internal tokens for themselves with the same scopes and lifetime as their session token, which was not ideal but not obviously dangerous.

Once restricted service-to-service authentication support was added, this changed.
Some services, such as the UWS storage service described in :sqr:`096`, need to allow access on behalf of a user from other services, but should not allow direct access from users.
Direct access could break the underlying data model or even cause security problems.

To address this, we moved these routes to a prefix that wasn't exposed outside the cluster via the Gafaelfawr ``Ingress`` and changed the ``Ingress`` resources for protected services to use the internal URL of the Gafaelfawr ``Service`` resource for their auth subrequests.
The Kubernetes ingress is allowed to contact the ``Service`` directly because it is whitelisted in the Gafaelfawr ``NetworkPolicy``.

This works and solves the problem.
These routes are now no longer accessible directly to users, and therefore cannot be used to bypass service restrictions by making arbitrary internal tokens.
However, this caused two problems:

- In order to construct the ``Ingress`` from a ``GafaelfawrIngress``, Gafaelfawr has to know the internal domain of the Kubernetes cluster to construct the URL for the service endpoint.
  This requires knowing the cluster domain, which unfortunately is not easily knowable inside the cluster.
  Gafaelfawr assumes the Kubernetes default of ``cluster.local``, but a specific cluster may have changed this to something else.
  In those cases, Gafaelfawr will have to be configured with its correct internal URL.

- This approach does not work with vCluster_ clusters when the ingress pod is running in the host cluster outside of the vCluster.
  In this case, the annotation on the ``Ingress`` inside the vCluster will references the service within that cluster, but that service is not available under that DNS name in the host cluster.
  Instead, the host cluster has to refer to it by a much more complicated name that includes the vCluster name mangling and namespacing, which is not knowable to Gafaelfawr inside the vCluster.
  Either the correct internal URL has to be manually configured or the annotation on the ``Ingress`` has to be transformed when it is copied to the host cluster.

  .. _vCluster: https://www.vcluster.com/

Note that the second case is an unsupported Phalanx_ configuration, which is the true root of the problem.
Phalanx and Gafaelfawr only support working with an ingress-nginx_ deployment in the same Kubernetes cluster.
While this issue with having the ingress in the host cluster instead of the vCluster can be worked around, that configuration is likely to cause other problems.

``OPTIONS`` requests
--------------------

Gafaelfawr implements a platform-wide CORS policy that prevents CORS preflight requests from being sent to underlying services unless they come from a hostname within that instance of the Science Platform.
This ensures that, regardless of the CORS policy implemented by the underlying application, cross-site requests are only allowed within the Science Platform.

This requires Gafaelfawr to analyze ``OPTIONS`` requests and apply a different policy to them.
The ``OPTIONS`` request from a CORS preflight check will not contain any credentials (either via cookies or the ``Authorization`` header), even if the subsequent request will be authenticated.
Therefore, Gafaelfawr must accept unauthenticated ``OPTIONS`` requests and, based on the CORS preflight policy, allow them through to the protected service even if it would otherwise require authentication.

The initial implementation of this idea had a serious security vulnerability: Ingress routes that used `authentication caching <https://gafaelfawr.lsst.io/user-guide/gafaelfawringress.html#caching>`__ would cache the successful response to the ``OPTIONS`` request and then allow unauthenticated requests to the underlying service.
This was because the key used for ``auth_request`` response caching in NGINX used only the contents of the ``Authorization`` and ``Cookie`` headers.
This was fixed by adding the request method to the cache key, at the small cost of separately caching authentication replies for ``GET``, ``POST``, etc.

The second problem with the initial implementation of this approach was for WebDAV servers.
The ``OPTIONS`` HTTP method predates CORS and has a valid meaning apart from CORS preflight requests, although it is not widely used or supported.
However, the one exception is WebDAV, which uses an ``OPTIONS`` request as part of the WebDAV protocol negotiation.
This, like other ``OPTIONS`` requests, can be sent unauthenticated even if the subsequent requests would be authenticated, but it does not contain the CORS headers.

By default, Gafaelfawr rejects all ``OPTIONS`` requests that are not valid CORS preflight requests (that are missing one of the ``Origin`` or ``Access-Control-Request-Method`` HTTP headers).
However, to allow WebDAV ``OPTIONS`` requests, ``GafaelfawrIngress`` supports a configuration option, ``config.allowOptions``.
When set to true, ``OPTIONS`` requests that are not CORS preflight requests and do not contain an ``Origin`` header are allowed through to the underlying service.

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
If it is coming from GitHub, it is stored in Redis.
The precedence logic will then use the right data sources for subsequent requests.

An advantage of this approach, in addition to simplicity, is that it allows administrators creating tokens via the token API to choose whether they want to override external data sources.
If they specify identity information for the token, it's stored in Redis and overrides external sources.
Otherwise, external sources will be used as configured.

Cookies
-------

Authentication cookies are stored as session cookies, rather than as cookies with an expiration tied to the lifetime of the user's credentials.
This is a somewhat arbitrary choice given that, in practice, session cookies are now always persisted by the browser so that sessions do not end when the user closes the browser.

We could switch to cookies with an expiration date.
However, the identity management system needs to use the cookie to store various other information, such as login state, that does not have an obvious expiration time.
The token and the other information could be divided into separate cookies, but that adds complexity with little benefit.

Cookies are encrypted primarily to prevent easy tampering or snooping, and because it's easy to do and has no drawbacks.
The encryption does not protect against theft of the entire cookie.
The cookie still represents a bearer token, and an attacker who gains access to the cookie can reuse that cookie from another web browser and gain access as the user.

The current design uses hostname-scoped cookies and assumes the entire Science Platform deployment runs within a single hostname.
This is not a good long-term assumption, since there are serious web security drawbacks to using a single hostname and a single web security context.
See :dmtn:`193` for more details about this problem, including a new proposed design that will likely be adopted in the future.

Database schema migrations
--------------------------

Gafaelfawr supports database schema migrations using Alembic_, chosen because we were already using SQLAlchemy for the storage layer.
The database is marked with its schema version (using the normal Alembic table), and all components of Gafaelfawr refuse to start if the schema version is too old.

.. _Alembic: https://alembic.sqlalchemy.org/en/latest/

Schema migrations are performed by setting the ``config.updateSchema`` setting in Phalanx_ and then syncing the Gafaelfawr chart.
This uses a Helm pre-install and pre-upgrade hook to update the schema before syncing any other Kubernetes resources.
The same approach should be used for bootstrapping a new cluster; if the schema does not already exist, that same hook will create the schema in an empty database.

.. _Phalanx: https://phalanx.lsst.io/

Performing a schema migration while Gafaelfawr is running is not, in general, safe, but Helm doesn't provide a simple mechanism to stop the service, perform the migration, and then restart the service.
Stopping the service before a migration must be done manually.
For this reason, we recommend against leaving ``config.updateSchema`` enabled, except in dev environments where the database is frequently reinitialized.

Alembic works somewhat poorly with async database code, which requires some complex workarounds in the Gafaelfawr source code.

Redis pools
-----------

Originally, Gafaelfawr had only one Redis server and connection pool.
Since this stored all Gafaelfawr tokens, it requires underlying persistent storage for all non-trivial environments and backups for production environments.
This Redis server was used for everything, including ephemeral data such as the temporary OpenID Connect authorization codes.

API rate limiting support greatly increased the number of required Redis writes, since every successful authenticated request for a service with a quota required a write to Redis.
Redis with persistent storage is potentially slower than Redis using only in-memory storage since it has to write to disk.
The Redis servers and internal connection pools were therefore split into a persistent Redis server, which uses persistent storage, and an ephemeral Redis server, which uses only in-memory storage and clears its database on restart.

Most data stayed in the persistent Redis.
The only data that moved to the ephemeral Redis were the temporary OpenID Connect authorization codes and the rate limiting data.
Quota overrides are stored in and read from the persistent Redis, since there was some chance they would need to survive a Redis restart.

Rate limiting is done via the limits_ library, which uses coredis_ rather than the more standard `Python Redis library <https://redis.readthedocs.io/en/stable/>`__.
This means there are unfortunately two internal Redis connection pools for the ephemeral Redis, one used for OpenID Connect authentication codes and one underlying the limits library for rate limiting.

.. _limits: https://limits.readthedocs.io/en/stable/
.. _coredis: https://coredis.readthedocs.io/en/stable/

Gafaelfawr still uses Redis rather than Valkey or some other replacement.
Its usage continues to fall within the acceptable use under the new non-open-source `Redis license <https://redis.io/legal/licenses/>`__ since Gafaelfawr only uses Redis internally and does not expose it as a service.

Kubernetes resources
====================

The initial implementation of the Kubernetes operator to create ``Secret`` resources from ``GafaelfawrServiceToken`` custom resources was hand-written using only kubernetes_asyncio_.
This worked, but it required carefully managing the watch and object generations, and didn't easily support periodically rechecking all generated tokens for validity.

.. _kubernetes_asyncio: https://github.com/tomplus/kubernetes_asyncio

After positive experiences using it in other projects, we decided to switch to Kopf_ to do the heavy lifting for the Kubernetes operator.
The Kopf framework does all the work of managing the Kubernetes watch and storing state in Kubernetes, and supports triggering code both on creation and modification of resources and on a timer, making it easy to implement the periodic recheck.

.. _Kopf: https://kopf.readthedocs.io/en/stable/

There are a few drawbacks to Kopf, unfortunately:

- Kopf doesn't easily support testing with a mock Kubernetes layer and wants to be tested against a real cluster.
  We use Minikube_ for testing inside GitHub Actions, but this unfortunately means a long test/update cycle when debugging since finishing the testing action can take about 10-15 minutes.
  Writing tests with Kopf also requires pausing the foreground test for an indeterminate amount of time until the background operator finishes, since there is no way for the operator to signal that it's done.
  That means tests have to be littered with arbitrary delays and take longer to run than they would otherwise.

  .. _Minikube: https://minikube.sigs.k8s.io/docs/

- Kopf allows handlers to return information that should be stored in the ``status`` field of the Kubernetes object, but the key under which that information is stored is forced to match the name of the handler function.
  Object create and update handlers take the same signature, but timer handlers do not, so there is no way to store the state from the last modification and the state from a periodic recheck in the same fields in the object.

- Timers and create/update handlers don't appear to be protected against each other and can be invoked at the same time and race.
  We are working around this by using the ``idle`` parameter to the timer, which tells it to avoid acting on objects that have changed in the recent past.
  This hopefully gives the create or update handler long enough to complete.

- Kopf does not currently support conversion webhooks, making it practically impossible to introduce new versions of the schemas for the custom resource definitions.
  This is discussed further in :ref:`crd-updates`.

We've chosen to live with these drawbacks since using Kopf makes it easier to add more operators.
We've now also added a ``GafaelfawrIngress`` custom resource, which is used as a template to generate an ``Ingress`` resource with the correct annotations.

The initial implementation of the Kubernetes custom resource support extracted information directly from the dictionary returned by the Kubernetes API.
In implementing ``GafaelfawrIngress`` support, it became obvious that using Pydantic to do the parsing of the custom object saves a lot of work and tedium.
This approach is now used for all custom resources.

Originally, Gafaelfawr didn't do anything special to tag the ingresses that it manages.
However, we encountered one use case where locating all Gafaelfawr-managed ingresses would be useful (rewriting them with Kyverno_ to allow deployment of Gafaelfawr in an unsupported vCluster_ configuration with ingress-nginx in the parent cluster).
We therefore added labelling all Gafaelfawr-managed ingresses with the ``app.kubernetes.io/managed-by`` label with value ``Gafaelfawr``.

.. _Kyverno: https://kyverno.io/

.. _crd-updates:

Updating custom resource definitions
------------------------------------

We have accumulated enough changes and bugs in the initial custom resource definitions for ``GafaelfawrIngress`` and ``GafaelfawrServiceToken`` that we would like to introduce new backward-incompatible versions of their schemas and convert all old resources to the new schema.

The method Kubernetes uses to do this is a conversion webhook.
Once one of those is in place and able to convert from the old schema to the new schema, the new schema can be introduced and set as the storage format, all stored resources converted, all Helm charts updated, and then the old schema version retired.

Unfortunately, Kopf does not currently support conversion webhooks (see the `relevant GitHub issue <https://github.com/nolar/kopf/issues/956>`__).
We therefore have held off on breaking schema changes and are considering contributing conversion webhook support to Kopf.

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

Token editing
-------------

The original implementation of the API allowed the user to edit their user tokens to change the name, scopes, and expiration.
The thought was that this would allow the user to correct errors without invalidating a token that may be in use in various places.

There were several bugs with this initially, including updating only the database and not Redis with new scope information and not updating expirations of child tokens if the expiration was shortened.
Those were discovered and fixed over time.

However, user token editing is not commonly supported.
(GitHub doesn't support it, for example.)
It may therefore pose unexpected security concerns, and it's additional UI surface to maintain.
We would also like to encourage token rotation and use of tokens in only one place, so forcing users to invalidate a token and create a new one is not the worst outcome.

We therefore dropped support for allowing users to edit their own tokens.
That support has been kept for administrators, since it's useful for fixing bugs and may be useful in some emergency response situations, and so that we can retain and continue testing the debugged code in case we change our minds later.

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

Quotas and rate limiting
========================

For the design and implementation of quotas and rate limiting, including implementation decisions made along the way, see :sqr:`073`.

.. _remaining:

Remaining work
==============

The following requirements should be satisfied by the Science Platform identity management system, but are not yet part of the design.
The **IDM-XXXX** references are to requirements listed in :sqr:`044`, which may provide additional details.

.. rst-class:: compact

- Restrict OpenID Connect authentication by scope
- Improved OpenID Connect protocol support
- Force two-factor authentication for administrators (IDM-0007)
- Force reauthentication to provide an affiliation (IDM-0009)
- Changing usernames (IDM-0012)
- Handling duplicate email addresses (IDM-0013)
- Disallow authentication from pending or frozen accounts (IDM-0107)
- Logging of user authentications (IDM-0200)
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
- Users viewing their quotas and quota history (IDM-1201, IDM-1402, IDM-2101)
- Users requesting new quota grants (IDM-1202, IDM-2102, IDM-3003)
- Quota grant expiration (IDM-1203, IDM-1303, IDM-2103)
- Group storage quotas for the group itself (IDM-2100)
- Administrator verification of email addresses (IDM-1302)
- User impersonation (see :sqr:`071`) (IDM-1304, IDM-1305, IDM-2202)
- Review newly-created accounts (IDM-1309)
- Merging accounts (IDM-1311)
- Logging of administrative actions tagged appropriately (IDM-1400, IDM-1403, IDM-1404)
- Affiliation-based groups (IDM-2001)
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

`RFC 8288: Web Linking`__
    The standard for the ``Link`` HTTP header and its relation types.

__ https://datatracker.ietf.org/doc/html/rfc8288

`RFC 9457: Problem Details for HTTP APIs`__
    Defines a "problem detail" as a way to carry machine-readable details of errors in a HTTP response.
    This avoids the need to define new error response formats for HTTP APIs.

__ https://datatracker.ietf.org/doc/html/rfc9457

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
