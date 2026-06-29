# Writeup Conventions

## Directory Structure
```
content/posts/<room-name>-<platform>/
├── index.md
├── feature.png
├── image1.png
├── image2.png
└── ...
```

- Room name should be lowercase, hyphenated (e.g. `brooklyn-nine-nine-tryhackme`, `dc03-hackmyvm`)
- Platform suffix: `-tryhackme`, `-hackmyvm`, `-htb`, etc.

## index.md Frontmatter
```yaml
+++
date = '<ISO-8601 timestamp>'
draft = false
title = '<Room Name> — <Platform>'
tags = ["tag1", "tag2", ...]
feature = 'feature.png'
showTableOfContents = true
+++
```

- Tags should include: the platform name, OS, difficulty, key techniques/tools used
- `feature` should point to `feature.png` in the same directory
- `showTableOfContents` should always be `true`

## Content Structure
Follow the DC03 writeup pattern (`content/posts/dc03-hackmyvm/index.md`):

### Sections
- `## Overview` — 1-2 paragraph summary of the room/box
- `## Step 1: Reconnaissance and Enumeration` — nmap output in code blocks, key findings as bullet points
- `## Step 2: Initial Access` — how foothold was gained
- `## Step 3: Privilege Escalation` — priv esc chain to root/admin
- `## Conclusion` — summary and remediations

### Images
- All images stored as PNG in the same directory as `index.md`
- Referenced as `![](filename.png)` (no path prefix)
- Images should be placed after the paragraph that describes them

### Code Blocks
- Use triple backticks with no language specifier for terminal output
- Inline commands use backticks like `command`
- Tool names in bold like **Hydra**

### Read Existing Writeup
Before creating a new writeup, read `content/posts/dc03-hackmyvm/index.md` as the reference template for structure and formatting.
