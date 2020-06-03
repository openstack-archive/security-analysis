========================
Security review findings
========================

keystonemiddleware security review findings - 4.17.1/pike
---------------------------------------------------------

**Status**: Draft/Completed

**Release**: Pike

**Version**: 4.17.1

**Review Date**: 02/26/2018

**Review Body**: OpenStack Security SIG

**Contacts**:

- PTL: Lance Bragstad - lbragstad

- Architect: Gage Hugo - gagehugo

- Security Reviewer: Luke Hinds - lhinds
- Security Reviewer: Jeremy Stanley - fungi


1. Security memcache with Pycrypto library
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Risk: Project documentation recommends use of the pycrypto library to secure
  memcache. Pycrypto is no longer maintained [0] with a last release made in
  2014. It also contains an unpatched CVE [1].
- Impact: Potential security flaw when using pycrypto due to lack of updates
  and security fixes.
- Likelihood: Medium
- Impact: Medium
- Overall Risk Rating: Medium
- Bug: https://bugs.launchpad.net/keystonemiddleware/+bug/1677308
- Recommendation: Correct docs to reference the cryptography libary.
- Investigation Results: Keystonemiddleware has since moved away from PyCrypto
  to a supported encryption library [2].

[0] https://github.com/dlitz/pycrypto/issues/173
[1] https://github.com/dlitz/pycrypto/issues/176
[2] https://github.com/openstack/keystonemiddleware/commit/e23cb36ac03c5e3a368cb8c493927cf8babc8dbc
