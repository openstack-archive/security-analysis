=================================
barbican security review findings
=================================

barbican security review findings - 3.0.0.0b2/newton
----------------------------------------------------
**Status**: Draft

**Release**: Newton

**Version**: 3.0.0.0b2

**Review Date**: 08/18/2016

**Review Body**: OpenStack Security Project

**Contacts**:

- PTL: Douglas Mendizábal - redrobot

- Architect: Douglas Mendizábal - redrobot

- Security Reviewer: Robert Clark - hyakuhei
- Security Reviewer: Doug Chivers - capnoday


Findings:
---------

1. Modification of ACLs in barbian database could compromise all secrets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- Risk: barbican has a feature that allows a tenant to grant another tenant
  access to a secret. This is controlled via a tenant mapping table within the
  barbican database. The implied security model of the barbican database (when
  running with PCKS#11) is that all cryptographic operations are performed in
  the HSM, a confidentiality or integrity breach of the database will not
  directly result in secrets being compromised. However if an attacker was able
  to modify the ACL mapping, they could grant a tenant access to any/all
  secrets stored in the HSM. Once the mapping is manipulated the attacker could
  retrieve secrets using the normal barbican API.
- Impact: All secrets stored in barbican are exposed.
- Likelihood: Medium
- Impact: High
- Overall Risk Rating: High
- Bug: <link to launchpad bug for this finding>
- Recommendation: Provide deployment guidance requiring strong controls
  securing access to the barbican database.


2. Misconfigured HSM credential could cause DoS via HSM auto-purge
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- Risk: A misconfigured or tampered barbican hardware security module (HSM)
  credential could cause a denial-of-service of barbican (and potentially other
  services using the HSM if it is shared), if the HSM is configured to purge
  after a number of failed connection attempts.
- Impact: Denial of service to barbican, potential loss of all secrets if there
  is inadequate backup, denial of service and potential loss of secrets for
  other services sharing the HSM.
- Likelihood: Low
- Impact: High
- Overall Risk Rating: Medium
- Bug: <link to launchpad bug for this finding>
- Recommendation: Deployment guidance recommending that HSMs should not be
  configured to auto-purge, unless this risk is actively managed via a security
  event monitoring system. In this later case, consider adding a delay
  period or auto backoff to barbican connection attempts to allow a SOC time
  to respond.


3. Compromised HSM credential could cause DoS and all secrets (PKCS#11 only)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- Risk: When using PKCS#11 to connect barbican to a HSM, a compromised HSM
  credential would allow an attacker to delete MKEK and HMAC keys, causing a
  denial of service. If these keys were not backed up, all secrets would be
  lost.
- Impact: Denial of service, loss of all secrets.
- Likelihood: Low
- Impact: High
- Overall Risk Rating: Medium
- Bug: <link to launchpad bug for this finding>
- Recommendation: Deployment guidance recommending that HSM credentials are
  protected.


4. Compromised HSM credential lets attacker access all secrets (KMIP only)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- Risk: Although this review focusses on PKCS#11 barbican deployments, the
  following KMIP finding was discovered during review and is included here for
  completeness. When using KMIP to connect barbican to a HSM, a compromised HSM
  credential allows an attacker to access all secrets stored in the HSM.
- Impact: Compromise of all secrets.
- Likelihood: Low
- Impact: High
- Overall Risk Rating: Medium
- Bug: <link to launchpad bug for this finding>
- Recommendation: Deployment guidance recommending that HSM credentials are
  protected.


5. Metadata should be sanitized before rendering to avoid XSS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- Risk: Lack of sanitization of metadata could lead to cross site scripting
  (XSS) vulnerabilities.
- Impact:
- Likelihood: Low
- Impact: Medium
- Overall Risk Rating: Medium
- Recommendation: Ensure future UI designers are aware of this risk and
  sanitize all metadata before rendering.


6. Weak keystone credentials could result in loss of barbican users/secrets.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- Risk: An integrity failure of the keystone event queue credentials could
  allow an attacker to point barbican at a keystone event queue controlled by
  the attacker, the attacker could then publish events triggering deletion of
  all users/projects/secrets in barbican.
- Impact: Soft deletion of all users/projects/secrets in the compromised
  barbican deployment. Limited impact as there is time to restore deleted
  data before the cleanup process runs.
- Likelihood: Low
- Impact: Medium
- Overall Risk Rating: Low
- Bug: <link to launchpad bug for this finding>
- Recommendation: Strong integrity controls for keystone credentials,
  monitoring to detect mass deletion.


7. Compromised keystone credentials could lead to barbican admin compromise
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- Risk: If the keystone credentials for the barbican service account (for
  token validation) have barbican admin privileges then a confidentiality
  failure could allow an attacker to manipulate the barbican administration
  functions.
- Impact: Compromise of secrets, DOS.
- Likelihood: Medium
- Impact: High
- Overall Risk Rating: Medium
- Bug: <link to launchpad bug for this finding>
- Recommendation: Do not grant barbican service account admin privileges


8. Compromise of PKCS#11 MKEK/HMAC backup could cause compromise of all secrets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- Risk: Loss of confidentiality of the PKCS#11 MKEK/HMAC backup could allow an
  attacker to decrypt all secrets in the barbican database.
- Impact: Compromise of all secrets
- Likelihood: Low
- Impact: High
- Overall Risk Rating: Medium
- Recommendation: Provide handling and encryption recommendations for MKEK/HMAC
  backups.


Recommendations:
----------------

1. Provide best practice recommendations for HSM usage and operations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- Recommendation: HSM security is outside the scope of this review (because it
  is an external entity), but it is critical to the security of a barbican
  deployment, so best practice recommendations should be provided for HSM usage
  and security.

2. Document metadata useage
~~~~~~~~~~~~~~~~~~~~~~~~~~~
- Recommendation: barbican metadata is not encrpyted, but users could store
  confidential data there. barbican documentation should highlight this to
  users.
