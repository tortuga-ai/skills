---
name: tortuga:image-check
description: Scan a website codebase for oversized images, report sizes, and convert to WebP.
license: MIT
metadata:
  author: tortuga-ai
  version: "1.0.0"
  organization: Tortuga AI
  public: true
  repo: https://github.com/tortuga-ai/skills
  date: March 2026
  abstract: Finds all images in the current project, reports file sizes, flags anything over a configurable threshold, and offers to convert flagged images to WebP using cwebp or ffmpeg.
---

Scan the current codebase for images, report sizes, and convert oversized ones to WebP.

## Step 1 — Find all images

```bash
find . -type f \( -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" -o -iname "*.gif" -o -iname "*.webp" -o -iname "*.avif" \) \
  -not -path "*/node_modules/*" \
  -not -path "*/.git/*" \
  -not -path "*/dist/*" \
  -not -path "*/.next/*" \
  | sort | xargs ls -lh 2>/dev/null | awk '{print $5, $9}' | sort -rh
```

## Step 2 — Report

Print a table:

| File | Size | Status |
|------|------|--------|
| path/to/image.jpg | 1.2 MB | Oversized |
| path/to/image.webp | 45 KB | OK |

Flag threshold:
- **Images > 200 KB** → Oversized (recommend convert/compress)
- **Already WebP/AVIF** → note size but no conversion needed unless still oversized
- **< 200 KB** → OK

At the end, print a summary:
- Total images found
- Total size
- Count oversized

## Step 3 — Offer conversion

If any oversized non-WebP images were found, ask:

> "Found X oversized images. Convert them all to WebP? (yes / no / select)"

If yes or select, proceed to Step 4.

## Step 4 — Convert to WebP

Check which tool is available:

```bash
which cwebp || which ffmpeg
```

**If cwebp available** (preferred):
```bash
cwebp -q 80 "input.jpg" -o "input.webp"
```

**If only ffmpeg available:**
```bash
ffmpeg -i "input.jpg" -q:v 80 "input.webp"
```

For each converted image:
- Create the `.webp` alongside the original (same directory, same name, new extension)
- Report: `converted/path/to/image.jpg -> image.webp (1.2 MB -> 45 KB, -96%)`

After converting, ask: "Delete the originals? (yes / no)" — only delete if confirmed.

## Step 5 — Final report

Print updated table showing before/after sizes and total savings.
