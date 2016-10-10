==========================
barbican architecture page
==========================

barbican - 3.0.0.0b2/newton
---------------------------
**Status**: Draft

**Release**: Newton

**Version**: 3.0.0.0b2

**Contacts**:

- PTL: Douglas Mendizábal - redrobot

- Architect: Douglas Mendizábal - redrobot

- Security Reviewer: Robert Clark - hyakuhei
- Security Reviewer: Doug Chivers - capnoday

Project description and purpose
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Barbican is a RESTful API for the secure storage, provisioning and management
of secret material such as passphrases, encryption keys and X.509 certificates.


Primary users and use-cases
~~~~~~~~~~~~~~~~~~~~~~~~~~~
1. End users will use the system to store sensitive data, such as passphrases
   encryption keys, etc.
2. Cloud administrators will use the administrative APIs to manage resource
   quotas.
3. Other cloud services will use the system to store/retrieve sensitive data on
   behalf of end users.


External dependencies & associated security assumptions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Barbican depends on an external Authentication/Authorization service.  In
a typical deployment this dependency will be fulfilled by the keystone service.

This review is of a barbican deployment with a certificate management system
(CMS) and PKCS#11.


Components
~~~~~~~~~~
- API Process (Python): Python web service listening for web requests.
- Worker Process (Python): Python process that executes tasks
  retrieved from the worker queue.
- keystone Listener Process (Python): Python process that consumes keystone
  events published by the keystone service.
- Database (MySQL): MySQL database to store barbican state data related to its
  managed entities and their metadata.
- Worker Queue (RabbitMQ): This queue is used to process new order
  requests.
- Configurable secure backend.  One of the following:

  - DogTag Service: An instance of the RedHat DogTag service.
  - Hardware Security Module (HSM): Dedicated hardware security module with
    either a PKCS#11 or KMIP interface

- Administrative CLI (Python): Command line interface tool that performs
  administrative tasks.


Service architecture diagram
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
.. image:: figures/barbican_architecture-diagram.png

Generated using draw.io, editable version:
https://www.draw.io/#G0Bz5iXPX51Qd9SW1IZXZ2UjljWEU


Data assets
~~~~~~~~~~~
- *Secret data* : Passphrases, Encryption Keys, RSA Keys - persisted in
  Database [PKCS#11] or HSM [KMIP] or [KMIP, Dogtag]
- *Secret metadata*  - persisted in database
- *RBAC rulesets* - persisted in policy.json
- *ACL rules* - persisted in database
- *DB Credentials* - persisted in barbican.conf
- *HSM Credentials* - persisted in barbican.conf, clients are also paired with
  the HSM (for Safenet anyway) via PKI
- *RabbitMQ Credentials* - persisted in barbican.conf
- *keystone Event Queue Credentials* - persisted in barbican.conf
- *Middleware configuration* - persisted in paste.ini
- *[PKCS#11] HSM HMAC Key* - persisted in HSM
- *[PKCS#11] HSM Master Key Encryption Key (MKEK)* - persisted in HSM
- *Per-project KEKs wrapped by MKEK* - stored in DB
- *CA (dogtag) credentials* - persisted in worker process barbican.conf
- *keystone Credentials (barbican service account)* - should be configured to
  only allow token validation but in some configurations may have higher
  privileges
- *Client keystone token* - ephemeral token provided by client, validated
  against keystone by barbican service account, could be any level of
  permissions (e.g. service accounts for other services)
- *CADF Credentials* - write only access to rabbit server

Data asset impact analysis
~~~~~~~~~~~~~~~~~~~~~~~~~~
Data Assets:

- *Secret Data*:

  - Integrity Failure Impact: Differs between plugins. In the case of the KMIP
    plugin, secret data is stored directly in the HSM and direct compromise is
    out of scope (See HSM credentials) however the PKCS#11 plugin encrypts
    secret data in the HSM, using a key that is not retrievable from the HSM.
    If secret data was tampered with in the database, subsequent retrieval via
    the HSM would fail because of HMAC integrity checks.
  - Confidentially Failure Impact: Secret data is persisted in the DB only in
    its encrypted form. When the encryption keys are protected Exposure of this
    data has little impact.
  - Note: The simple-crypto plugin does not protect against these issues
  - Availability: Service breaks if secret data is not available

- *Secret metadata*:

  - Integrity Failure Impact: barbican ignores metadata, it is there as a
    facility for users. User applications could be affected by changes to this
    metadata. Future UI designers should ensure that metadata is sanitized
    before being rendered to avoid things like XSS in metadata.
  - Confidentiality Failure Impact: Metadata is not intended to be confidential
    and barbican does not encrpyt it, but end-users could use metadata to
    store confidential information.
    does not encrypt it.
  - Availability Failure: barbican unaffected

- *RBAC/ACL rulesets*:

  - Integrity Failure Impact: Attacker with valid AuthN could be granted access
    to any secret.
  - Confidentiality Failure Impact: Mappings between users and secrets
  - Availability Failure Impact: barbican will not start if the file is not
    readable.

- *DB Credentials*:

  - Integrity Failure Impact: barbican won't be able to access DB and fail to
    start.
  - Confidentiality Failure Impact: ACLs could be abused to grant access to all
    secrets for any Authenticated user (AuthZ failure). All Secrets could be
    deleted / mangled.
  - Availability Failure Impact: barbican won't be able to access DB and fail
    to start.

- *HSM Credentials*:

  - Integrity Failure Impact: barbican can no longer authenticate to HSM,
    there is potential here to cause the HSM to purge on multiple reconnects.
  - Confidentiality Failure Impact:

    - [PKCS#11] No keys are exposed, MKEK and HMAC keys could be deleted,
      causing a denial of service (DoS). If these were not backed up, all
      secrets would be lost.
    - [KMIP] Attacker is able to retrieve all secrets from the HSM

  - Availability: barbican won't be able to access HSM and will fail to CRUD
    secrets in KMIP, unable to decrypt or encrypt secrets in PKCS#11.

- *RabbitMQ Credentials*:

  - Integrity Failure Impact: barbican and Workers can no longer access the
    queue. Denial of service.
  - Confidentiality Failure Impact: An attacker could add new tasks to the
    queue which would be executed by workers. User quotas could be exhausted by
    an attacker. DoS. User would be unable to create genuine secrets.
  - Availability Failure Impact: barbican could no longer create new secrets
    without access to the queue.

- *Identity Service (keystone) Event Queue Credentials [Including endpoint
address]*:

  - Integrity Failure Impact: An attacker could setup their own queue, point
    barbican to this rogue queue and by publishing events, delete all
    users/projects/secrets in barbican. Additionally, typical DoS scenario
    using incorrect credentials for the legitimate queue.
  - Confidentiality Failure Impact: barbican should only be able to subscribe
    to the event queue. An attacker with the credentials could create their own
    subscriber meaning that legitimate events don't get consumed by barbican.
    Race condition?
  - Availability Failure Impact: Projects might not get deleted when they
    should be but the overall run state of barbican is unaffected.

- *Middleware Configuration*:

  - Integrity Failure Impact: Can remove/replace keystone auth middleware -
    allows you to capture tokens and also could cause barbican to fail open
    (everything authorized). If keystone auth middleware is deleted from
    paste.ini, an attacker could add their own keystone headers to REST
    requests and barbican would interpret them as valid. If an attacker had
    access to the filesystem they could inject their own middleware by dropping
    a class on the FS (in the Python path) and directing paste.ini to use that.
  - Confidentiality Failure Impact: Minimal - an attacker can enumerate the
    middleware.
  - Availability Failure Impact: barbican breaks

- *PKCS#11 MKEK/HMAC*: - Stored un-retrievable in HSM

  - Integrity Failure Impact: PKCS#11 create, read, update, delete (CRUD)
    operations will fail.
  - Confidentiality Failure Impact: All secrets in DB could be decrypted.
    However this failure mode is out of scope for this TA.
  - Availability Failure Impact: PKCS#11 CRUD operations will fail.

- *PKCS#11 MKEK/HMAC*: - backed up *somewhere*

  - Integrity Failure Impact: HSM Disaster Recovery will fail : PKCS#11 CRUD
    operations will fail.
  - Confidentiality Failure Impact:  All secrets in DB could be decrypted by an
    attacker with knowledge of the DB contents.
  - Availability Failure Impact: HSM Disaster Recovery will fail : PKCS#11 CRUD
    operations will fail.

- *Dogtag Credentials*:

  - Integrity Failure Impact: Inability to submit or issue certificates (DoS)
  - Confidentially Failure Impact: A malicious user can create valid
    certificates for any service that trusts the Dogtag CA.
  - Availability Failure Impact: Unable to submit or retrieve certificates.=
    DoS.

- *Certificate Signing Request*:

  - As a cryptographically "public" asset, we do not model CIA for Certificate
    Signing Requests

- *keystone Credentials*:

  - Integrity Failure Impact: barbican will not be able to validate user
    credentials and fail. DoS.
  - Confidentially Failure Impact: A malicious user might be able to abuse
    other OpenStack services (depending on keystone role configurations) but
    barbican is unaffected. If the service account for token validation also
    has barbican admin privilages, then a malicious user could manipulate
    barbican admin functions.
  - Availability Failure Impact: barbican will not be able to validate user
    credentials and fail. DoS.

- *Client keystone Token*:

  - Integrity Failure Impact: barbican will not be able to validate user
    credentials and fail. DoS.
  - Confidentially Failure Impact: A malicious user might be able to abuse
    OpenStack services including barbican. If the token had an admin scope then
    an attacker may be able to subvert multiple cloud services. [Total fail].
  - Availability Failure Impact: Not a persistent asset, provided and used at
    the same time.


Interfaces
~~~~~~~~~~
Format:
From->To *[Transport]*

- Assets in flight
- Authentication?
- Description

1. Client->API Process *[TLS]*:

   - Assets in flight: User keystone Credentials, Plaintext Secrets, HTTP Verb,
     Secret ID, Path
   - Access to keystone credentials or plaintext secrets is considered a total
     security failure of the system - this interface must have robust
     confidentiality and integrity controls, i.e. TLS.

2. Administrator->API Process *[TLS]*:

   - Assets in flight: barbican admin keystone Credentials
   - An attacker with access to the admin credentials can modify quotas,
     expanding or reducing them for any user. This has potential availability
     impact. DoS.

3. Administrator->API Process Host *[SSH]*:

   - The actions of a malicious administrator are out of scope for the
     OpenStack Threat Analysis Process. However the OSSP suggests hosts for
     OpenStack services are configured following best practices such as
     **<TODO>.**

4. Administrator->Administrative CLI *[SSH]*:

   - The actions of a malicious administrator are out of scope for the
     OpenStack Threat Analysis Process. However the OSSP suggests hosts for
     OpenStack services are configured following best practices such as
     **<TODO>.**

5. API Process->PKCS#11 HSM *[NTL]*: - work to understand NTL is required

   - Assets in flight: HSM Credentials(Partition), HSM Commands, Plaintext
     Secrets, MKEK wrapped PKEKs (MKEK is never transmitted over this
     transport).
   - Note: Access to individual secrets resulting in a compromise of system
     integrity. Only secrets that were transmitted in view of an attacker are
     compromised.

6. Worker Process to HSM *[NTL]*: - work to understand NTL is required, how
   does it compare to TLS?

   - Assets in flight: HSM Credentials(Partition), HSM Commands, Plaintext
     Secrets, MKEK
   - Credentials/Authentication: Each HSM connection has different credentials.
     Credentials tied specifically to the FQDN of the worker process.
     Certificate pairs generated on the HSM and installed into worker
     processes. Site Crypto Officer / Crypto Officer creates certificates on
     the HSM.

7. Worker Process->Certificate Authority (Dogtag)*[TLS]*:

   - Assets in flight: CSR (uploaded by client or generated in Worker), Dogtag
     credentials
   - Note: All workers share the same set of credentials

8. API Process->keystone REST *[TLS]*: **TODO** - is this interface missing on
   the diagram?

   - Assets in flight: keystone credentials, Customer token


Resources
~~~~~~~~~
- https://wiki.openstack.org/wiki/barbican
