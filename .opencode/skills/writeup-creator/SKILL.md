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

1. Read `content/posts/dc03-hackmyvm/index.md` as the reference template for structure and formatting.
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
6. Tags should use full names like "TryHackMe", "Linux", "Privilege Escalation", "FTP", etc.
