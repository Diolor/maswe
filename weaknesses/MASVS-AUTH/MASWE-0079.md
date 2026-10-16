---
title: Abusable Client API Keys
id: MASWE-0079
alias: abusable-client-api-keys
requirement: "The app only holds client keys that cause no harm when used by anyone."
platform: [android, ios]
profiles: [L1, L2, EUDIW]
threat: MAS-THREAT-0079
attacks: [MAS-ATTACK-0001, MAS-ATTACK-0003, MAS-ATTACK-0064, MAS-ATTACK-0068, MAS-ATTACK-0091]
mappings:
  masvs-v1: [MSTG-AUTH-1, MSTG-AUTH-12]
  masvs-v2: [MASVS-AUTH-1]
  cwe: [285, 290, 306, 807]
  android-risks:
  - insecure-api-usage
  maswe-beta: [MASWE-0005]
refs:
- https://cloud.google.com/docs/authentication/api-keys#api_key_restrictions
- https://cloud.google.com/docs/authentication/api-keys-best-practices
- https://firebase.google.com/docs/projects/api-keys
- https://firebase.google.com/docs/app-check
- https://developer.apple.com/documentation/devicecheck/establishing-your-app-s-integrity
status: new
---

## Overview

This weakness occurs when the app holds a client key (an API key, access token, or similar credential shared by every installation) that, alone or combined with the service's access rules, allows operations that cause harm when anyone performs them, such as billed calls, writes to the app owner's data, or reads of protected data.

Many services issue client keys that are meant to be used from apps, such as API keys for maps, analytics, crash reporting, or backend-as-a-service platforms, often placed in SDK configuration files. Because the same key is present in every installation, it is only as protected as the least protected device that holds it. Keys in the app package can be extracted from it (see @MASWE-0004). Keys the app receives at runtime, for example through remote configuration or from a backend after attestation, can be read in a vulnerable phone. Services often let the key be restricted to identifiers the app sends with the request, such as package name, signing-certificate digest, or bundle identifier. These identifiers are public and any client can send them, so such restrictions do not prevent other clients from using the key.

The weakness is therefore not that the key can be obtained, but that the key allows harmful operations once obtained. It does not cover client keys that only serve generic identification, for example telling Android and iOS builds apart, routing requests, or attributing usage per platform, as long as no security decision depends on them.

## Modes of Introduction

- **Over-Privileged Client Keys**: Giving the app a key, or configuring the service behind it, so that anyone holding the key can perform billed, write, or read operations that the app owner intends only for authenticated users or its own backend.
- **Broad Key Scope**: Issuing the app's key without limiting it to the APIs the app needs, so that it reaches every API enabled on the same project or account, including APIs enabled after the key shipped.
- **Runtime-Delivered Client Keys**: Delivering the same key to every installation at runtime, for example through remote configuration or after attestation, instead of issuing per-installation credentials.
- **Client Keys in Third-Party SDKs**: Integrating SDKs whose service calls are authorized only by a static key from the app's configuration files.
- **Security Decisions Based on a Client Key**: Granting access, trust, or higher limits to requests because they carry the client key or pass restrictions based on identifiers the app declares.

## Impact

- **Financial Loss**: Attackers can send billed requests under the app's key, resulting in unexpected charges to the app owner or quotas reserved for it.
- **Compromise of System Integrity and Business Operations**: Attackers can inject fake events or records, write content that the app later shows to its users, or exhaust the key's quotas, resulting in polluted data, unwanted content reaching users, and service disruption for every installation at once.
- **Authentication or Authorization Bypass**: Attackers can present the key and the app's public identifiers from their own client to pass checks meant to admit only the genuine app, resulting in unauthorized access to functionality.

## Mitigations

- **Keep Privileged Keys on a Backend**: Ensure every client key the app holds, whether packaged or delivered at runtime, allows only operations that are harmless when anyone performs them, and route any other operation through a backend that holds the key.
- **Restrict Key Scope**: Limit each key to the APIs the app needs, and issue keys the app holds from a separate project or account so that enabling new APIs does not extend them.
- **Issue Per-Installation Credentials**: Replace shared client keys with short-lived credentials issued to one installation after server-verified attestation (see @MASWE-0056) and bound to a key in the platform keystore (see @MASWE-0003). Because the attacker's own installation still receives a valid credential, enforce per-installation quotas and rate limits on the backend and revoke the credential on abuse.
- **Do Not Trust Requests Because They Carry the Key**: Use server-verified app attestation (see @MASWE-0056) rather than the client key or app-declared identifiers to decide which clients receive access, trust, or higher limits, and enable the equivalent app checks offered by third-party services.
