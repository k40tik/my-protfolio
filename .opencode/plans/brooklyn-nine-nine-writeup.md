# Plan: Brooklyn Nine Nine — TryHackMe Writeup

## Steps

### 1. Clean up and prepare directory
- Delete old `content/posts/skynet-tryhackme/` (already done)
- Create `content/posts/brooklyn-nine-nine-tryhackme/` (already done)
- Download all 8 images from AppFlowy URLs (already done — verified as valid PNGs):
  - `feature.png`, `ftp-anon.png`, `ssh-pass-auth.png`, `note-to-jake.png`
  - `hydra-success.png`, `sudo-l.png`, `gtfo-bins.png`, `root-shell.png`

### 2. Write `content/posts/brooklyn-nine-nine-tryhackme/index.md`
Frontmatter:
```
+++
date = '2026-06-29T11:43:00+03:00'
draft = false
title = 'Brooklyn Nine Nine — TryHackMe'
tags = ["TryHackMe", "Linux", "Boot2Root", "Easy", "FTP", "Privilege Escalation", "CTF Writeup"]
feature = 'feature.png'
showTableOfContents = true
+++
```

Content sections (following DC03 pattern):
- **Overview** — room description
- **Step 1: Reconnaissance and Enumeration** — nmap output (21 FTP, 22 SSH, 80 HTTP), FTP anonymous access + SSH password auth screenshots
- **Step 2: Initial Access** — note_to_jake.txt content, hydra brute force for user jake
- **Step 3: Privilege Escalation** — sudo -l showing less, GTFOBins technique, root shell via `sudo less /etc/profile` + `!/bin/bash`
- **Conclusion** — summary and remediations

### 3. Create AGENTS.md at repo root
Instructions for future writeup creation:
- How to structure posts (dir pattern: `content/posts/<room-name>/index.md`)
- Frontmatter format (date, draft, title, tags, feature, showTableOfContents)
- Image handling (PNG images stored locally in post dir)
- Markdown conventions (## sections, code blocks, ![](img.png) references)

### 4. Commit and push
```
git add content/posts/brooklyn-nine-nine-tryhackme/ AGENTS.md
git commit -m "Add Brooklyn Nine Nine — TryHackMe writeup"
git push
```
