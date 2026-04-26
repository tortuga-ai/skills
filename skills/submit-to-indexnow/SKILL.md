---
name: tortuga:submit-to-indexnow
description: Submit one or more URLs to IndexNow so Bing (and other IndexNow-compatible engines) index them immediately.
license: MIT
metadata:
  author: tortuga-ai
  version: "2.0.0"
  organization: Tortuga AI
  public: true
  repo: https://github.com/tortuga-ai/skills
  date: March 2026
  abstract: Submits URLs to the IndexNow API for immediate Bing indexing. Auto-discovers the key from the live domain. Supports any domain.
---

Submit one or more URLs to [IndexNow](https://www.bing.com/indexnow/getstarted) so Bing (and other IndexNow-compatible engines) index them immediately.

## Prerequisites

Each domain you submit must have an IndexNow key file deployed. To set one up for a new domain:

1. Generate a key: any 32-character hex string (e.g. `python3 -c "import uuid; print(uuid.uuid4().hex)"`)
2. Create `public/<key>.txt` containing the key value
3. Deploy so it's accessible at `https://<domain>/<key>.txt`

See https://www.bing.com/indexnow/getstarted for full documentation.

## Usage

```
/submit-to-indexnow <url1> [url2] [url3] ...
```

URLs must be full `https://` URLs. One or more, space-separated.

If no URLs are provided, ask: "Which URL(s) do you want to submit to IndexNow?"

## Step 1 — Extract domain and discover key

Extract the domain from the provided URL(s).

For each unique domain, discover its IndexNow key by fetching the live sitemap or a known path. The IndexNow key file is a `.txt` file at the domain root with a 32-character hex filename. Fetch the domain's robots.txt first, then look for the key:

```bash
# Fetch robots.txt and look for any .txt file references that could be the key
curl -s "https://<domain>/robots.txt"
```

If robots.txt doesn't reveal the key, try listing known key patterns by fetching the sitemap or ask the user. Alternatively, the user may provide the key directly.

The most reliable method: fetch all `.txt` files linked from the domain. The key file contains only the 32-char hex key as its content. Use `curl` to try fetching common key file patterns from the live domain, or ask the user for the key.

## Step 2 — Verify key is live

Before submitting, verify the key file is accessible on the live domain:

```bash
curl -s -o /dev/null -w "%{http_code}" "https://<domain>/<key>.txt"
```

If it returns 404, tell the user: "The IndexNow key file is not live yet at `https://<domain>/<key>.txt`. Wait for the deploy to finish and try again."

## Step 3 — Submit

Group URLs by domain. For each domain, POST to `https://api.indexnow.org/indexnow`:

```bash
curl -s -o /dev/null -w "%{http_code}" -X POST https://api.indexnow.org/indexnow \
  -H "Content-Type: application/json; charset=utf-8" \
  -d '{"host":"<domain>","key":"<key>","keyLocation":"https://<domain>/<key>.txt","urlList":["<url1>","<url2>"]}'
```

## Step 4 — Report result

- HTTP 200 or 202 → success: "Submitted N URL(s) to IndexNow for `<domain>` (Bing will crawl shortly)"
- HTTP 400 → bad request: report the URL and suggest checking the format
- HTTP 422 → key not verified: the key file is not deployed or doesn't match. Tell the user to check the key file on the domain
- Any other error → report status code and response body
