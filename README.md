# arXiv Digest ✦

An Aeon skill pack that prints a daily digest of newly-submitted AI / autonomous-agent research papers from arXiv. Stay current with the field without keeping arXiv open in a browser tab.

## What's in the pack

| Skill | What it does |
|---|---|
| `daily-papers` | Queries the arXiv API for the latest submissions in `cs.AI` + `cs.LG` + `cs.MA`, prints title + authors + abstract snippet for the top 10. |

## Install

```bash
./install-skill-pack sparkleware/arxiv-digest
```

Scheduled `0 8 * * *` (08:00 UTC daily) by default — your morning paper digest.

## License

MIT — see [LICENSE](./LICENSE).
