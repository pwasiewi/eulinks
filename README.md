# eulinks

Personal link catalog organized by topic. Each subdirectory contains one or more
`*links.txt` / `*linki.txt` files in a simple Markdown bullet format:

```
### Section Title
 - https://example.com/some-article
 - https://other.org/reference
```

Original mirrors: https://git.io/vATMf · http://bit.ly/eulinks

---

## Tools

All three scripts run from inside `eulinks/` and share the same file conventions:

- Only lines inside `### Header` sections that have at least one URL bullet are touched.
- Lines inside ```` ``` ```` fenced code blocks are **never** modified.
- Bullet style (`-` vs `*`) is auto-detected from the section and preserved on insert.
- A trailing `# 2026` marker (configurable) tags every automatically inserted line.

---

### `checklinks` — verify URLs and remove dead ones

```
./checklinks                         # dry-run: report dead links across all files
./checklinks --only ai,bd_sql        # limit scan to specific subdirectories
./checklinks -a                      # apply: remove dead lines (asks for confirmation)
./checklinks -a -y                   # apply without confirmation
./checklinks --aggressive            # also treat 403/429/5xx/timeout as dead
./checklinks --report dead.txt       # write report to file instead of stdout
```

A line is removed only if **every** URL it contains is dead. Non-URL lines
(headers, prose, markdown links with valid text) are never touched.

Options: `--workers 24`, `--timeout 8.0`, `--retries 1`.

---

### `insertlinks` — batch-insert curated links from a JSON mapping

```
./insertlinks mapping.json
./insertlinks mapping.json --marker "# 2026" --dry-run
```

**mapping.json format:**

```json
{
  "relative/path/to/file.txt": {
    "Section Header Title": [
      "https://example.com/article-1",
      "https://example.com/article-2"
    ]
  }
}
```

Each URL is inserted immediately after the matching header, before existing bullets.
Header titles must match exactly (case-sensitive, stripped of leading/trailing whitespace).
Headers not found in their file are reported as `HEADERS NOT MATCHED` to stderr and
the script exits 1 — fix the title and re-run.

The script is **not idempotent**: running it twice on the same mapping inserts the URLs
twice. Always run `checklinks --only <dir>` after inserting to verify new links are live.

---

### `enrichlinks` — automatically find and insert new links via SearXNG

Requires the SearXNG instance from `aistack` to be running (`docker ps | grep searxng`).
Check the actual host port: `docker ps | grep searxng` — it may be `8081`, not `8080`.

```
./enrichlinks DIR [DIR ...]                       enrich all unenriched sections
./enrichlinks DIR --dry-run                       show what would be inserted, don't write
./enrichlinks DIR --max 3                         up to 3 links per section (default: 2)
./enrichlinks DIR --context "Android Kotlin"      extra keywords appended to every query
./enrichlinks DIR --sections "Redis,Docker swarm" target specific sections (substring)
./enrichlinks DIR --no-skip-marker                re-enrich sections that already have the marker
./enrichlinks DIR --time-range none               broaden search (default: year)
./enrichlinks ai bd_sql prog_java                 loop over multiple dirs in one call
```

For each section that has URL bullets and no existing `# 2026` marker, the script:

1. Builds a query `"<section title> 2026 [--context]"` and calls the SearXNG REST API.
2. Filters candidates: drops blacklisted domains (Instagram, TikTok, Facebook, Twitter/X,
   LinkedIn, Pinterest) and non-reference URL patterns (`/shorts/`, `/reel/`, IG post paths).
3. Checks up to 8 remaining candidates concurrently (6 workers) for HTTP 200.
4. Skips URLs already present anywhere in the file.
5. Inserts the top `--max` live results immediately after the header.

**`--context` is important for directories with Polish or vague section titles** — without
it, SearXNG may return unrelated results. Use topic keywords that narrow the domain:

```
sw_android  →  --context "Android Kotlin development"
prog_python →  --context "Python programming tutorial"
ai          →  --context "AI machine learning deep learning"
sieci       →  --context "Linux networking sysadmin"
```

Options: `--searxng URL`, `--timeout 8.0`, `--delay 1.2`,
`--time-range year|month|week|day|none`, `--marker "# 2026"`.

**Typical workflow for a new directory:**

```bash
cd ~/Claude/eulinks
SEARXNG=http://localhost:8081   # adjust to actual port

# 1. Preview
./enrichlinks prog_python --searxng $SEARXNG --context "Python tutorial" --dry-run

# 2. Write
./enrichlinks prog_python --searxng $SEARXNG --context "Python tutorial"

# 3. Verify inserted links are live, remove dead ones
./checklinks --only prog_python
./checklinks --only prog_python -a -y
```

**Manual workflow (insertlinks):**

```bash
# 1. Create mapping.json with hand-picked URLs for specific sections
./insertlinks /tmp/my_mapping.json --dry-run
./insertlinks /tmp/my_mapping.json

# 2. Verify
./checklinks --only prog_java
```
