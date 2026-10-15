---
title: Abusable Shared Keys in the App
id: MASWE-0079
alias: abusable-shared-keys
requirement: "Keys that all installations of the app share cannot be used by others to incur costs, corrupt data, disrupt service, or access data."
platform: [android, ios]
profiles: [L1, L2]
threat: MAS-THREAT-0079
attacks: [MAS-ATTACK-0001, MAS-ATTACK-0003, MAS-ATTACK-0064, MAS-ATTACK-0065, MAS-ATTACK-0068, MAS-ATTACK-0091]
mappings:
  masvs-v1: [MSTG-AUTH-1]
  masvs-v2: [MASVS-AUTH-1]
  cwe: [269, 290, 770, 798]
refs:
- https://cloud.google.com/docs/authentication/api-keys#api_key_restrictions
- https://github.com/firebase/firebase-android-sdk/issues/1583
- https://blog.dixitaditya.com/bypassing-google-maps-api-key-restrictions
- https://www.cloudsek.com/blog/hardcoded-google-api-keys-in-top-android-apps-now-expose-gemini-ai
- https://developer.android.com/privacy-and-security/security-key-attestation#certificate_status
- https://firebase.google.com/docs/app-check
status: new
---

## Overview

This weakness occurs when the app holds a key that every installation shares, and anyone who uses that key can incur costs, corrupt data, disrupt the service, or access data.

Many services issue keys that are meant to be used from apps, such as API keys for maps, AI/ML, analytics, or backend-as-a-service platforms, and keys in SDK configuration files. A shared key leaks wherever the app keeps it. Keys in the app package can be extracted from it (see @MASWE-0004). Keys the app receives at runtime, for example through remote configuration or from a backend after attestation, can be read from memory or traffic on a device the attacker controls. Attestation decides which devices receive the key, but it can be passed on a compromised device, for example with leaked factory attestation keys, and a device can be compromised after it was attested. Once one installation gives the key away, every installation is exposed, so every installation is only as safe as the weakest device that holds the key.

Restricting a key to an app by identifiers the app sends with the request, such as package name, signing-certificate digest, or bundle identifier, does not stop others from using it, because those identifiers are public and any client can send them. A shared key therefore does not authenticate the app. What decides the harm is only what the key allows.

A shared key is acceptable in the app only if its use by anyone is harmless. Such a key can still serve generic identification, for example telling Android and iOS builds apart, routing requests, or attributing usage per platform, as long as no security decision depends on it. It becomes a weakness when it can be used to:

- incur costs, through billed or metered calls,
- corrupt data, by writing records, events, or reports the app owner relies on, or by sending messages to the app's users,
- disrupt the service, by exhausting quotas or triggering abuse suspensions that affect every installation at once,
- access data, by reading records, files, or indexes the key unlocks.

## Modes of Introduction

- **Over-Privileged Shared Keys**: Giving the app a key that allows billed, write, or read operations beyond what any anonymous client of the service may safely perform.
- **Runtime-Delivered Shared Keys**: Delivering the same key to every installation at runtime, for example through remote configuration or after attestation, instead of issuing per-installation credentials.
- **Broad Key Scope**: Not limiting the key to the APIs the app needs, or enabling additional APIs later on a project or account whose key the app holds.
- **Security Decisions Based on a Shared Key**: Granting access, trust, or higher limits to requests because they carry a shared key, instead of using the key only for generic identification.
- **Client-Declared Restrictions**: Relying on key restrictions that the service evaluates against identifiers the app declares in the request to limit who can use the key.
- **Shared Keys in Third-Party SDKs**: Integrating SDKs whose service calls are authorized only by a static key from the app's configuration, without enabling the server-verified app checks the service offers.

## Impact

- **Financial Loss**: Attackers can send billed requests (e.g., maps or AI/ML services) under the app's key, resulting in unexpected charges to the app owner.
- **Compromise of System Integrity and Business Operations**: Attackers can inject fake events, records, or reports, or send messages to the app's users, resulting in decisions based on polluted data and users receiving content the app owner did not send.
- **Compromise of System Integrity and Business Operations**: Attackers can exhaust quotas or trigger abuse suspensions on the shared key, resulting in service disruption for every installation of the app at once.
- **Compromise of Sensitive Data**: Attackers can read the data the key unlocks, such as database records, stored files, or search indexes, resulting in unauthorized disclosure of user or app data.

## Mitigations

- **Give the App Only Keys That Are Safe to Be Public**: Ensure every shared key the app holds, whether packaged or delivered at runtime, allows only operations that are harmless when anyone performs them, and keep any other key on a backend.
- **Issue Per-Installation Credentials**: Replace shared keys with short-lived credentials bound to one installation, for example to a key generated in the platform keystore, so that a leak can be revoked for that installation alone.
- **Restrict Key Scope**: Limit each key to the APIs the app needs, and keep keys the app holds in a separate project or account so that enabling new APIs does not extend them.
