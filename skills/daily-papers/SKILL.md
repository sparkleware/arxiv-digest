---
name: arXiv Daily Papers
description: Fetches the latest AI / autonomous-agent paper submissions from arXiv and prints a sparkle-themed digest of the top 10
var: ""
tags: [research, arxiv, papers, sparkleware]
---

# arXiv Daily Papers ✦

Each morning, scan the freshest submissions across the three arXiv subject classes most relevant to AI agents — `cs.AI` (artificial intelligence), `cs.LG` (machine learning), and `cs.MA` (multi-agent systems) — and print a one-screen digest with title + authors + one-line abstract.

## Goal

Surface the 10 most-recent papers in the union of `cs.AI`, `cs.LG`, `cs.MA` (de-duped, sorted newest first), formatted so you can scan and decide what to actually read.

## Steps

### 1. Configure constants

```bash
CATS="cat:cs.AI+OR+cat:cs.LG+OR+cat:cs.MA"
MAX_RESULTS=10
API_URL="http://export.arxiv.org/api/query?search_query=${CATS}&start=0&max_results=${MAX_RESULTS}&sortBy=submittedDate&sortOrder=descending"
```

If `${var}` is set, the user wants to narrow further (e.g. `${var}="reinforcement learning"`). In that case prepend an `AND all:` clause:

```bash
if [ -n "${var:-}" ]; then
  ENCODED=$(printf '%s' "$var" | sed 's/ /+/g')
  API_URL="http://export.arxiv.org/api/query?search_query=(${CATS})+AND+all:${ENCODED}&start=0&max_results=${MAX_RESULTS}&sortBy=submittedDate&sortOrder=descending"
fi
```

### 2. Fetch the Atom feed

```bash
XML="$(curl -fsSL -A 'sparkleware-arxiv-digest/0.1' "$API_URL" 2>/dev/null || echo '')"
if [ -z "$XML" ]; then
  echo "(Couldn't reach arXiv — skipping digest today.)"
  exit 0
fi
```

arXiv's API is rate-limited at ~1 request per 3 seconds per IP. We only fire once, so we're well under the limit.

### 3. Parse the Atom XML

arXiv returns Atom XML. Parsing options in order of preference:

**Primary — `xmllint` (Linux/macOS, often in git-bash on Windows):**

```bash
if command -v xmllint >/dev/null 2>&1; then
  ENTRIES=$(echo "$XML" | xmllint --xpath 'count(//*[local-name()="entry"])' - 2>/dev/null || echo 0)
  for i in $(seq 1 "$ENTRIES"); do
    TITLE=$(echo "$XML" | xmllint --xpath "string(//*[local-name()='entry'][$i]/*[local-name()='title'])" - 2>/dev/null | tr '\n' ' ' | sed 's/  */ /g')
    AUTHORS=$(echo "$XML" | xmllint --xpath "//*[local-name()='entry'][$i]/*[local-name()='author']/*[local-name()='name']/text()" - 2>/dev/null | tr '\n' ',' | sed 's/,/, /g; s/, $//')
    SUMMARY=$(echo "$XML" | xmllint --xpath "string(//*[local-name()='entry'][$i]/*[local-name()='summary'])" - 2>/dev/null | tr '\n' ' ' | sed 's/  */ /g' | cut -c 1-160)
    LINK=$(echo "$XML" | xmllint --xpath "string(//*[local-name()='entry'][$i]/*[local-name()='id'])" - 2>/dev/null)
    printf '%d. %s\n   %s\n   %s...\n   %s\n\n' "$i" "$TITLE" "$AUTHORS" "$SUMMARY" "$LINK"
  done
  exit 0
fi
```

**Fallback — primitive `sed`/`awk` extraction (when no xmllint available):**

```bash
echo "$XML" | awk '
/<entry>/      {entry=1; title=""; authors=""; link=""; next}
/<\/entry>/    {entry=0; printf("%s\n   %s\n   %s\n\n", title, authors, link); next}
entry && /<title>/    {gsub(/.*<title>|<\/title>.*/,"",$0); title=$0}
entry && /<name>/     {gsub(/.*<name>|<\/name>.*/,"",$0); authors = (authors ? authors ", " : "") $0}
entry && /<id>/       {gsub(/.*<id>|<\/id>.*/,"",$0); link=$0}
'
```

### 4. Print the digest banner

```bash
echo "       ✦"
echo "     ✧   ✧"
echo "   ✦  ●  ✦       arXiv digest — $(date -u +%Y-%m-%d)"
echo "     ✧   ✧"
echo "       ✦"
echo ""
echo "Top ${MAX_RESULTS} recent submissions across cs.AI + cs.LG + cs.MA${var:+ (filtered: ${var})}"
echo ""
```

Then run the parser block from step 3 to emit the per-paper lines.

### 5. Optional — append to memory log

```bash
LOG_FILE="${AEON_ROOT:-.}/memory/logs/arxiv-digest-$(date -u +%Y-%m-%d).md"
if [ -d "${AEON_ROOT:-.}/memory" ]; then
  mkdir -p "$(dirname "$LOG_FILE")"
  {
    echo "# arXiv digest — $(date -u +%Y-%m-%d)"
    echo ""
    # re-run the parser, redirecting the same output into the log
  } >> "$LOG_FILE"
fi
```

Useful when you skim the digest in the terminal but want to revisit papers later — the log file accumulates a searchable archive.

## Notes

- Requires `curl`. Strongly recommended: `xmllint` (libxml2-utils on apt, libxml2 on brew). Graceful fallback to awk works but is less reliable on weirdly-escaped titles.
- arXiv's terms: be polite, throttle to ~1 request per 3 seconds (we send 1 per day, so fine).
- No API key required.
- Safe to schedule daily — pure read.
- If `${var}` set, the digest is filtered to papers also matching that string (regardless of where in the abstract / title / authors).
