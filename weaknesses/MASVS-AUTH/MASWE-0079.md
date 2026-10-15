---
title: Abusable Shared Keys in the App
id: MASWE-0079
alias: abusable-shared-keys
requirement: "The app only holds shared keys that cannot be used by others to incur costs, corrupt data, disrupt service, or access data."
platform: [android, ios]
profiles: [L1, L2]
threat: MAS-THREAT-0079
attacks: [MAS-ATTACK-0001, MAS-ATTACK-0003, MAS-ATTACK-0068, MAS-ATTACK-0091]
mappings:
  masvs-v1: [MSTG-AUTH-1]
  masvs-v2: [MASVS-AUTH-1]
  cwe: [269, 290, 798]
refs:
- https://cloud.google.com/docs/authentication/api-keys#api_key_restrictions
- https://github.com/firebase/firebase-android-sdk/issues/1583
- https://blog.dixitaditya.com/bypassing-google-maps-api-key-restrictions
- https://www.cloudsek.com/blog/hardcoded-google-api-keys-in-top-android-apps-now-expose-gemini-ai
- https://firebase.google.com/docs/app-check
status: new
---

## Overview

This weakness occurs when the app holds a key that every installation shares and that allows operations which cause harm when performed by anyone, such as billed calls, writes to the app owner's data, or reads of protected data.

Many services issue keys that are meant to be used from apps, such as API keys for maps, AI/ML, analytics, or backend-as-a-service platforms, often placed in SDK configuration files. Because the same key is present in every installation, it is only as protected as the least protected device that holds it. Keys in the app package can be extracted from it (see @MASWE-0004), and keys delivered at runtime, for example through remote configuration or from a backend after attestation, can be read from memory or traffic on a device the attacker controls (see @MASWE-0056). Services often let the key be restricted to identifiers the app sends with the request, such as package name, signing-certificate digest, or bundle identifier. These identifiers are public and any client can send them, so such restrictions do not prevent other clients from using the key.

The weakness is therefore not that the key can be obtained, which is covered by @MASWE-0004, but that the key allows harmful operations once obtained. It does not cover shared keys that only serve generic identification, for example telling Android and iOS builds apart, routing requests, or attributing usage per platform, as long as no security decision depends on them.

## Modes of Introduction

- **Over-Privileged Shared Keys**: Giving the app a key that allows billed, write, or read operations beyond what any anonymous client of the service may perform.
- **Broad Key Scope**: Issuing the app's key from a project or account that also has other APIs enabled, so that the key extends to them.
- **Runtime-Delivered Shared Keys**: Delivering the same key to every installation at runtime, for example through remote configuration or after attestation, instead of issuing per-installation credentials.
- **Shared Keys in Third-Party SDKs**: Integrating SDKs whose service calls are authorized only by a static key from the app's configuration files, without enabling the server-verified app checks the service offers.
- **Security Decisions Based on a Shared Key**: Granting access, trust, or higher limits to requests because they carry the shared key or pass restrictions based on identifiers the app declares.

## Impact

- **Financial Loss**: Attackers can send billed requests (e.g., maps or AI/ML services) under the app's key, resulting in unexpected charges to the app owner.
- **Compromise of System Integrity and Business Operations**: Attackers can inject fake events, records, or reports, send messages to the app's users, or exhaust quotas and trigger abuse suspensions on the key, resulting in polluted data, unwanted content reaching users, and service disruption for every installation at once.
- **Compromise of Sensitive Data**: Attackers can read the data the key unlocks, such as database records, stored files, or search indexes, resulting in unauthorized disclosure of user or app data.

## Mitigations

- **Keep Privileged Keys on a Backend**: Ensure every shared key the app holds, whether packaged or delivered at runtime, allows only operations that are harmless when anyone performs them, and route any other operation through a backend that holds the key.
- **Restrict Key Scope**: Limit each key to the APIs the app needs, and issue keys the app holds from a separate project or account so that enabling new APIs does not extend them.
- **Issue Per-Installation Credentials**: Replace shared keys with short-lived credentials bound to a single installation, for example to a key generated in the platform keystore (see @MASWE-0003), so that a leaked credential can be revoked for that installation alone.
- **Do Not Trust Requests Because They Carry the Key**: Use server-verified app attestation (see @MASWE-0056) rather than the shared key or app-declared identifiers to decide which clients receive access, trust, or higher limits, and enable the equivalent app checks offered by third-party services.
