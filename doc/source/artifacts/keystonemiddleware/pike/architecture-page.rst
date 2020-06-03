=================
Architecture page
=================

keystonemiddleware architecture - 4.17.1/pike
---------------------------------------------
**Status**: Draft/Ready for Review/Reviewed

**Release**: Pike

**Version**: 4.17.1

**Contacts**:

- PTL: Lance Bragstad - lbragstad

- Architect: Gage Hugo - gagehugo

- Security Reviewer: Luke Hinds - lhinds
- Security Reviewer: Jeremy Stanley - fungi

Project description and purpose
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
keystonemiddleware [0]_ is primarily used for integrating with the OpenStack
Identity API [2]_ and handling authorization enforcement based upon the data
within the OpenStack Identity tokens. Also included is middleware that
provides the ability to create audit events based on API requests.


Primary users and use-cases
~~~~~~~~~~~~~~~~~~~~~~~~~~~
The primary users of keystonemiddleware are other services within an OpenStack
deployment that require identity information supplied from OpenStack
Identity (keystone).


External dependencies & associated security assumptions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
keystonemiddleware depends on having an OpenStack Identity (keystone) [2]_
endpoint. Without an Identity endpoint, there is not much use for
keystonemiddleware. It also depends on having a service configuration
for the service that it is protecting.


Components
~~~~~~~~~~

- OpenStack Identity - keystone (Python)
- memcache (optional)


Service architecture diagram
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. image:: figures/keystonemiddleware_architecture-diagram.png

Architecture Page [1]_

Data assets
~~~~~~~~~~~

- *Authorization Tokens* - persisted in memcache
- *memcache encryption keys* - persisted in keystonemiddleware.conf


Data asset impact analysis
~~~~~~~~~~~~~~~~~~~~~~~~~~

Data Assets:
- *Authorization Token*:

  - Integrity Failure Impact: Attacker that can capture and hijack a valid
    auth token can get access to anything scoped to the token.

- *keystonemiddleware.conf*:

  - Integrity Failure Impact: Attacker who can read the config file can gain
    access to the memcache encryption key, which can allow them to access and
    modify all cached tokens.


Interfaces
~~~~~~~~~~

1. User -> KeystoneMiddleware *[TLS]*:

   - Assets in flight: keystone Token
   - An attacker who can successfully intercept the token can modify anything
     that the token is scoped to. This has potential availability impact.


Resources
~~~~~~~~~

.. [0] `<https://docs.openstack.org/developer/keystonemiddleware/#python-middleware-for-openstack-identity-api-keystone>`_
.. [1] `<https://docs.openstack.org/developer/keystonemiddleware/middlewarearchitecture.html>`
.. [2] `<https://docs.openstack.org/developer/keystone/index.html>`
