---
type: Skill
name: arXiv Daily Papers
category: research
description: Fetches the latest AI / autonomous-agent paper submissions from arXiv and prints a sparkle-themed digest of the top 10
var: ""
tags: [research, arxiv, papers, sparkleware]
---

> **${var}** — Optional. Pass a topic string (e.g. `"reinforcement learning"`) to narrow the digest to papers whose title / abstract / authors also match it. If empty, digests the 10 newest submissions across `cs.AI` / `cs.LG` / `cs.MA`.

# arXiv Daily Papers ✦

Today is ${today}. Each morning, scan the freshest submissions across the three arXiv subject classes most relevant to AI agents — `cs.AI` (artificial intelligence), `cs.LG` (machine learning), and `cs.MA` (multi-agent systems) — and build a one-screen digest with title + authors + one-line abstract.

## Goal

Surface the 10 most-recent papers in the union of `cs.AI`, `cs.LG`, `cs.MA` (de-duped, sorted newest first), write them to a dated file under `output/`, and send a short notification. Same arXiv source and query as always — this run just persists the result and logs its status.

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

### 2. Fetch the feed (primary: WebFetch)

arXiv's query API returns an Atom XML feed over the open web, so read it with the built-in **WebFetch** tool (see the Sandbox note below). Call `WebFetch` on the `$API_URL` from step 1 with a prompt like:

> "This is an arXiv Atom feed. List up to 10 entries, newest first. For each, give: title, comma-separated author names, a one-line (≤160 char) abstract summary, and the arXiv id/link."

WebFetch returns the parsed entries directly — use those as the digest items in step 4.

arXiv's API is rate-limited at ~1 request per 3 seconds per IP. We only fire once, so we're well under the limit.

**Fallback — `curl` + local XML parse.** If WebFetch is unavailable, fetch the same URL with `curl` and parse the Atom XML yourself:

```bash
XML="$(curl -fsSL -A 'sparkleware-arxiv-digest/0.1' "$API_URL" 2>/dev/null || echo '')"
if [ -z "$XML" ]; then
  # curl returned empty in the sandbox — retry this same URL with WebFetch (see Sandbox note).
  echo "(Couldn't reach arXiv via curl — falling back to WebFetch.)"
fi
```

Parse options, in order of preference:

**`xmllint` (Linux/macOS, often in git-bash on Windows):**

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
fi
```

**Primitive `sed`/`awk` extraction (only when no `xmllint` is available):**

```bash
if ! command -v xmllint >/dev/null 2>&1; then
  echo "$XML" | awk '
/<entry>/      {entry=1; title=""; authors=""; link=""; next}
/<\/entry>/    {entry=0; printf("%s\n   %s\n   %s\n\n", title, authors, link); next}
entry && /<title>/    {gsub(/.*<title>|<\/title>.*/,"",$0); title=$0}
entry && /<name>/     {gsub(/.*<name>|<\/name>.*/,"",$0); authors = (authors ? authors ", " : "") $0}
entry && /<id>/       {gsub(/.*<id>|<\/id>.*/,"",$0); link=$0}
'
fi
```

### 3. Build the digest banner

```bash
echo "       ✦"
echo "     ✧   ✧"
echo "   ✦  ●  ✦       arXiv digest — ${today}"
echo "     ✧   ✧"
echo "       ✦"
echo ""
echo "Top ${MAX_RESULTS} recent submissions across cs.AI + cs.LG + cs.MA${var:+ (filtered: ${var})}"
echo ""
```

### 4. Write the digest to `output/`

Persist the run so the file — not stdout — is the full record. Write `output/daily-papers-${today}.md`, using the entries from step 2:

```bash
mkdir -p output
OUT="output/daily-papers-${today}.md"
{
  echo "# arXiv Daily Papers — ${today} ✦"
  echo ""
  echo "Top ${MAX_RESULTS} recent submissions across cs.AI + cs.LG + cs.MA${var:+ (filtered: ${var})}"
  echo ""
} > "$OUT"
```

Append one entry per paper, in this markdown shape:

```markdown
### [Title](arxiv-link)
*Author One, Author Two, …*
One-line (≤160 char) abstract summary.
```

If zero entries came back (feed unreachable or empty), still write the file with a single `_No papers retrieved this run._` line so the archive stays continuous.

### 5. Notify

Send a SHORT summary via `./notify` — the file is the full record, this is what reaches the operator:

```
*arXiv Daily Papers — ${today}*

[N] papers across cs.AI / cs.LG / cs.MA${var:+ · filter: ${var}}. Top: [shortened title].

Full digest: https://github.com/${GITHUB_REPOSITORY}/blob/main/output/daily-papers-${today}.md
```

### 6. Log status

Append one status block to `memory/logs/${today}.md`:

```bash
mkdir -p memory/logs
cat >> "memory/logs/${today}.md" <<EOF

## daily-papers
- **Query**: cs.AI + cs.LG + cs.MA${var:+ (filter: ${var})}
- **Source**: export.arxiv.org/api/query
- **Papers written**: N (of ${MAX_RESULTS} requested)
- **Output**: output/daily-papers-${today}.md
- **Status**: STATUS_OK | STATUS_DEGRADED (feed unreachable / partial parse)
EOF
```

Use `STATUS_OK` when the feed returned papers and the file was written; `STATUS_DEGRADED` when the feed was unreachable, returned nothing, or only partially parsed.

## Sandbox note

WebFetch and WebSearch are built-in Claude tools that bypass the GitHub Actions sandbox network gate. Use those for external reads; if a curl call returns empty in the sandbox, retry the same URL with WebFetch. This skill reads arXiv's Atom feed (web content), so **WebFetch is the primary read** (step 2) and `curl` + local XML parsing is only the fallback.

## Notes

- Primary read is WebFetch; the `curl` fallback additionally benefits from `xmllint` (libxml2-utils on apt, libxml2 on brew), with a graceful `awk` fallback for weirdly-escaped titles.
- arXiv's terms: be polite, throttle to ~1 request per 3 seconds (we send 1 per day, so fine).
- No API key required.
- Safe to schedule daily — pure read.
- If `${var}` is set, the digest is filtered to papers also matching that string (regardless of where in the abstract / title / authors).

## Constraints

- **Same source, always.** The digest comes from `export.arxiv.org/api/query` over the exact category union above — do not swap in other feeds or endpoints.
- **No filler.** If fewer than `${MAX_RESULTS}` papers come back, write fewer entries — never pad with unrelated papers.
- **File is the record.** Always write `output/daily-papers-${today}.md` and log a status block, even on a degraded/empty run.
