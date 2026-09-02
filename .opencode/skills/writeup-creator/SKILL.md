---
name: writeup-creator
description: Create CTF writeups for the portfolio following the established conventions
metadata:
  audience: self
  workflow: content
---

## What I do

I create CTF writeup markdown files in `content/posts/` following the project's established conventions. I read the reference template first, then create the writeup in the correct directory with all images downloaded.

## When to use me

Use this when the user says "new writeup for <room-name> <platform>" or anything about creating a CTF writeup.

## Instructions

1. Read `content/posts/dc03-hackmyvm/index.md` as the reference template for structure and formatting. Read `content/posts/ascension-hacksmarter/index.md` as the reference for the first-person writing style.
2. Create directory `content/posts/<room-name>-<platform>/` where room name is lowercase hyphenated.
3. Download the first image from the user's message as `feature.png`.
4. Download all other screenshots with descriptive lowercase hyphenated names (e.g. `nmap-scan.png`, `web-login.png`).
5. Create `index.md` with:
   - Frontmatter: `date` (ISO-8601), `draft = false`, `title = '<Room Name> — <Platform>'`, tags including platform, OS, difficulty, key techniques, `feature = 'feature.png'`, `showTableOfContents = true`
   - Sections: `## Overview`, `## Step 1: Reconnaissance and Enumeration`, `## Step 2: Initial Access`, further steps as needed, `## Conclusion`
   - `**Key Findings:**` bullet list after nmap output
   - Code blocks with commands and output separated
   - Images referenced as `![](name.png)` with no path prefix, placed after the paragraph describing them
   - Conclusion with numbered attack chain summary and bullet-point remediations
6. **Write in first person, active voice.** The writeup is told from the author's perspective ("I scanned", "I exploited", "I dumped"). Never refer to the author as "the attacker" or use third-party framing, and avoid passive constructions such as "was performed", "was crafted", "was dumped". Examples:
   - Write "I crafted a `.lnk` file to trigger an SMB connection back to me" — NOT "a `.lnk` file was crafted to trigger an SMB connection back to the attacker".
   - Write "I performed a BloodHound collection" — NOT "a BloodHound collection was performed".
   - Write "A machine account I controlled was used..." — NOT "an attacker-controlled machine account was used".
   - Use `<my-ip>` placeholders in commands — NOT `<attacker-ip>`.
   - Keep `## Conclusion` attack-chain bullets as terse noun-phrase summaries (e.g. "SMB guest enumeration revealed a non-standard share"), matching the reference template.
7. Tags should use full names like "TryHackMe", "Linux", "Privilege Escalation", "FTP", etc.
8. **NEVER include actual passwords, credentials, hashes, or secret keys in the writeup.** Replace all sensitive values with `<REDACTED>`. This includes:
   - Plaintext passwords in commands (e.g., `-p 'password'`, `-newpass 'password'`)
   - Cracked passwords shown in output
   - NTLM hashes in DCSync/secretsdump output
   - Hardcoded credentials found in binaries or config files
   - Any other sensitive authentication material
