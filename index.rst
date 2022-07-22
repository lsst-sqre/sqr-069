Abstract
========

The identity management, authentication, and authorization component of the Rubin Science Platform is responsible for maintaining a list of authorized users and their associated identity information, authenticating their access to the Science Platform, and determining which services they are permitted to use.
This tech note collects decisions, analysis, and trade-offs made in the implementation of the system that may be of future interest.
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

Authentication flows
====================

The protocol for HTTP Basic Authentication, using ``x-oauth-basic`` as either the username or password along with the token, is reportedly based on GitHub support for HTTP Basic Authentication.
GitHub currently appears to recognize tokens wherever they're put and does not require the ``x-oauth-basic`` string.
(This would likely be wise for Gafaelfawr to do as well, but it has not yet been implemented.)

The password is probably the better place to put the token in HTTP Basic Authentication, since software will know to protect or obscure it, but common practice in other APIs that support using tokens for HTTP Basic Authentication is to use the username.
Gafaelfawr therefore supports both.
As a fallback, if neither username nor password is ``x-oauth-basic``, it assumes the username is the token, but this is not documented (except here) since we'd prefer users not use it.

Storage
=======

Gafaelfawr stores data in both a SQL database and in Redis.
Use of two separate storage systems is unfortunate extra complexity, but Redis is poorly suited to store relational data about tokens or long-term history, while PostgreSQL is poorly suited for quickly handling a high volume of checks for token validity.

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
