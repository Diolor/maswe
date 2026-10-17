---
title: Sensitive Data Hardcoded in the App Package
id: MASWE-0004
alias: data-hardcoded-app-package
requirement: "The app does not hardcode sensitive data in the application package."
platform: [android, ios]
profiles: [L1, L2, EUDIW]
threat: MAS-THREAT-0004
attacks: [MAS-ATTACK-0001]
mappings:
  masvs-v1: [MSTG-STORAGE-1, MSTG-CODE-2]
  masvs-v2: [MASVS-STORAGE-1]
  cwe: [312, 321, 540, 798]
  android-risks:
  - insecure-api-usage
  maswe-beta: [MASWE-0005, MASWE-0013, MASWE-0036]
refs:
- https://cloud.google.com/docs/authentication/api-keys#securing
- https://cloud.google.com/docs/authentication/api-keys#api_key_restrictions
- https://firebase.google.com/docs/projects/api-keys
- https://github.com/OWASP/ASVS/blob/master/5.0/en/0x16-V7-Session-Management.md
- https://github.com/OWASP/ASVS/blob/master/5.0/en/0x17-V8-Authorization.md
- https://datatracker.ietf.org/doc/html/rfc8252#section-8.5
- https://api-security.owasp.org/editions/2023/en/0x11-t10/
- https://github.com/gitleaks/gitleaks
---

## Overview

This weakness occurs when sensitive data is embedded in the app package (APK/IPA) and shipped with the app, where anyone who downloads the package can recover it.

The hardcoded sensitive data may include API keys and secrets for first- or third-party services, credentials such as passwords or session tokens, and cryptographic material such as symmetric or private keys embedded directly in the package (as opposed to being generated and stored in a platform keystore, see @MASWE-0003). It also includes developer leftover artifacts, such as staging or integration URLs, developer emails and usernames, and source code files left in the package (e.g. `.swift`, `.cpp`, map files, or other build artifacts) that leak internal information.

Not all API keys embedded in an app package are sensitive. Some keys are public identifiers that a service issues for use in distributed clients, for example to attribute requests to a project or to tell platforms apart, and the service does not treat them as secrets. Such a key cannot be kept confidential, as [RFC 8252](https://datatracker.ietf.org/doc/html/rfc8252#section-8.5) notes for secrets statically included in distributed apps, so what matters is what it allows. A key that lets anyone holding it reach protected resources or privileged functionality, such as billed APIs, writes to the app owner's data, or reads of protected data, is sensitive, and embedding it is a weakness. Whether a given key is public or sensitive therefore depends on the permissions and restrictions the remote service associates with it, not on its name or intended use. Authorization and access control weaknesses in the remote service itself should be assessed using standards such as the [OWASP ASVS](https://github.com/OWASP/ASVS/blob/master/5.0/en/0x17-V8-Authorization.md) and the [OWASP API Security Top 10](https://api-security.owasp.org/editions/2023/en/0x11-t10/).

Note that developer _debug_ artifacts (verbose logging, backdoors, testing utilities, hidden switches) are covered separately under resilience in @MASWE-0061. The focus here is on hardcoded sensitive data that leaks confidentiality regardless of any anti-tampering considerations.

## Modes of Introduction

- **App Source Code**: Embedding secrets directly in the code that is compiled into the app.
- **App Assets and Resources**: Including secrets in configuration files, manifests, property lists, string resources, and other bundled files.
- **Libraries**: Including secrets in the configuration files or source code of first-party, third-party, or transitive dependencies.
- **Build and Developer Leftovers**: Inadvertently packaging staging/integration endpoints, developer identities, and source files with the app.

## Impact

- **Financial Loss**: Attackers can abuse compromised API keys to make unauthorized billed API calls (e.g., AI/ML services), resulting in unexpected charges to the app owner.
- **Compromise of System Integrity and Business Operations**: Attackers can use extracted credentials to access backend services, resulting in service disruption, policy-violation suspensions, or denial of service.
- **Compromise of Sensitive Data**: Attackers can use extracted cryptographic material to decrypt protected data, resulting in unauthorized disclosure of user or app data.
- **Bypass of Protection Mechanisms**: Attackers can use hardcoded keys to unlock paid features or access restricted content, resulting in circumvention of the protections the app enforces.

## Mitigations

- **Proxy Static Secrets Through a Middleware**: If a stateful API service is not viable, front the stateless API with a middleware solution (API proxy or gateway) that proxies requests between the app and the API endpoint, keeping the static secret server-side rather than in the client. Use JSON Web Tokens (JWT) and JSON Web Signature (JWS) as appropriate.
- **Use Stateful API Services**: Prefer API services that provide secure authentication, client validation, and session controls. Implement dynamic tokens that expire after a reasonably short time (e.g., 1 hour) to reduce the impact of key exposure, and ensure proper error handling and logging to detect unauthorized access attempts. Consider OAuth 2.0 and libraries such as AppAuth to simplify secure OAuth flows.
- **Issue Per-Installation Credentials at Runtime**: Do not move a shared secret out of the package only to deliver the same value to every installation at runtime, for example through remote configuration or a key management service. Such a secret is still shared, so it can be extracted from any single device and used against every installation. Instead, have the backend issue dynamically generated, short-lived credentials to each installation rather than static API keys (see [ASVS 7.2.2](https://github.com/OWASP/ASVS/blob/master/5.0/en/0x16-V7-Session-Management.md)), sender-constrained to a key in the platform keystore where possible (see [ASVS 10.4.5](https://github.com/OWASP/ASVS/blob/master/5.0/en/0x19-V10-OAuth-and-OIDC.md) and @MASWE-0003), and delivered over a pinned channel (see @MASWE-0028). Any client, including an attacker's, can obtain such a credential, so the protection comes from the backend enforcing per-installation quotas and revoking abused credentials, not from keeping credentials away from attackers.
- **Restrict Unavoidable Hardcoded Keys**: If a key must ship with the app, treat it as public. Configure the service so that the key grants only operations that any anonymous client may perform, limit it to the APIs the app uses, and be prepared to revoke and reissue it, knowing that existing installations keep the old value until they update. Restrictions based on identifiers the app declares, such as its package name, signing certificate, or bundle identifier, limit casual misuse but do not authenticate the client, because any client can send them. Enforce authorization for every privileged operation on the remote endpoint instead, and verify that configuration using the OWASP ASVS and the OWASP API Security Top 10.
- **Use Platform Keystores**: Store cryptographic keys and authentication material using the platform's hardware-backed keystore (Android Keystore, iOS Keychain) instead of embedding them in the package. See @MASWE-0003.
- **Audit for Leftover Secrets**: Regularly audit the codebase and dependencies for hardcoded sensitive data and developer leftovers (e.g., using tools such as [gitleaks](https://github.com/gitleaks/gitleaks)) and strip build artifacts and source files from release packages.
- **Harden Only as a Last Resort**: When no other secure option is available, use white-box cryptography, code/resource obfuscation, and RASP to raise the effort required to extract secrets, ensuring keys are only assembled in memory when needed. These techniques deter but do not prevent extraction and must not replace the mitigations above.
