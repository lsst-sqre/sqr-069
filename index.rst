Abstract
========

The identity management, authentication, and authorization component of the Rubin Science Platform is responsible for maintaining a list of authorized users and their associated identity information, authenticating their access to the Science Platform, and determining which services they are permitted to use.
This tech note collects the decisions, analysis, and trade-offs made in the implementation of the system.
It collects historical background useful for understanding design and implementation decisions, but which may clutter other documents and distract from the details of the system as implemented.

.. note::

   This is part of a tech note series on identity management for the Rubin Science Platform.
   The other two primary documents are DMTN-234_, which describes the high-level design; and DMTN-224_, which describes the implementation.
   See the `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ for a complete list of related documents.

.. _DMTN-234: https://dmtn-234.lsst.io/
.. _DMTN-224: https://dmtn-224.lsst.io/

Federated identity
==================

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

Authentication
==============

GitHub
------

Organizational membership
^^^^^^^^^^^^^^^^^^^^^^^^^

In the original GitHub integration, we used the token returned by GitHub from the OAuth 2.0 authorization flow to retrieve metadata from the user's account and then discarded it.

However, only the organization and team membership at the time the user authorizes the OAuth App are passed to that app.
The authorization is then remembered, and if the user authenticates to the same Science Platform again, they are not prompted again for what organization data to release.
Instead, the prior authorization is reused, including the old organization data, and thus new team memberships are not picked up by the Science Platform.
This means that even if the user logs out (destroying their Gafaelfawr session token and cookie) and logs back in, Gafaelfawr will not see any new organization information because the user will not have released it from the authorization screen.

We solved this problem by retaining the GitHub token for the user's authentication by storing it in the user's encrypted Gafaelfawr session cookie.
If the user explicitly logs out, that token is retrieved and used to revoke the user's OAuth App authorization.
This forces the user back to the OAuth App authorization screen the next time they log in, which in turn causes GitHub to release updated organization information (subject to organization data visibility).

Prior authorizations with incomplete information may still be reused if the user never explicitly logs out, only lets their cookies expire, but at least we can document that explicitly logging out and logging back in will fix any missing organizational data.
