# Representment Workup Tool

Prepares chargeback cases for a disputes analyst. For each case it lays out the card scheme's evidence checklist, checks the merchant's PDFs and images against it, cites the exact passage behind every finding and recommends whether to fight the chargeback, accept the loss or ask the merchant for more.

**Who it's for:** payments disputes analysts, who today spend most of each case re-reading scheme rules, opening several PDFs to check one fact and writing up a rationale.

**Result:** a working end-to-end tool. It runs a 10-case sample batch across 15 Visa and Mastercard reason codes, with every satisfied requirement backed by a quote that is verified against the source and highlighted in the PDF or image. Time saved per case has not been measured with analysts.

[Watch the demo video](https://www.loom.com/share/ad42f216f27a499fbb32970c18ccfc3d)

## How it works

| Step | Where | What happens |
|---|---|---|
| 1. Upload | `/upload`, `POST /api/batches` | Cases JSON and evidence files are matched by file name. Missing files are shown but do not block analysis |
| 2. Parse | `lib/parse.ts` | LiteParse runs OCR locally and returns text regions with page coordinates. Results are cached by file hash. Images are classed as text, visual or mixed |
| 3. Analyse | `lib/workup.ts` | One forced tool call to Claude per case. It gets the reason code checklist, case data, parsed evidence and image pixels, and returns a status, citations and any gap per requirement |
| 4. Verify | `lib/resolver.ts` | Each quote is matched back to the parsed text and mapped to highlight rectangles. A requirement without a verified quote cannot count as satisfied |
| 5. Decide | `lib/rule-action.ts` | Code, not the model, computes `represent`, `request_more_evidence` or `accept_liability` from the verified statuses |
| 6. Review | `/queue`, `POST /api/cases/[id]/review` | Inbox, checklist and source viewer side by side. The analyst can override the action and rationale, then confirm |

## Key design decisions

- **Rules in code, not in the prompt.** The 15 reason codes, their thresholds (all requirements, or any one or two) and the two codes that cannot be represented live in `lib/rules.ts`. The model fills the checklist. It never picks the action. Trade-off: adding a reason code is a code change.
- **No verified quote, no credit.** The model's citations are re-checked against the parsed text before they can support a requirement. This blocks invented evidence at the cost of missing some true matches when OCR is poor.
- **Visual evidence kept separate.** For photos, the model may record a `visual_observation` of what is literally visible. The UI labels these apart from OCR quotes, so the analyst knows which findings are not text-verified.
- **Local parsing.** LiteParse needs no API key and returns coordinates for highlighting. Trade-off: OCR quality is lower than a hosted parser.
- **Single process, file storage.** Batches, parse results and workups are JSON under `.cache/`. Background processing is fire-and-forget and the UI polls. Fine for a demo; not durable across restarts and not for many users.

## Stack

Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS 4, Anthropic SDK (default model `claude-sonnet-4-6`), LiteParse (`@llamaindex/liteparse`), PDF.js.

## Repo layout

```
app/          pages (/upload, /queue) and API routes
components/   upload drop zone, inbox, workup pane, source viewer
lib/          parse, workup (Claude call), resolver, rules, rule-action, store, types
data/         synthetic sample batch: cases.json and 19 evidence files
docs/         design doc (PDR.md), reason code rules, dataset notes
```

## Setup and run

Requires Node.js 20 or later and an Anthropic API key.

```bash
npm install
cp .env.example .env    # set ANTHROPIC_API_KEY; ANTHROPIC_MODEL is optional
npm run dev
```

Open http://localhost:3000/upload and click **Load sample dataset (10 cases)**, or upload your own cases JSON with its evidence files. The first run parses files in the background. Parsed files and batch state are cached in `.cache/`.

Check the build with:

```bash
npm run build
```

There are no automated tests yet.

## What I'd do next

- Add tests for `rule-action.ts` and the quote resolver, using the 10 sample cases as fixtures.
- Measure: run the batch across models and score action agreement, verified citation rate and analyst edits.
- Log tokens, latency and analyst overrides per case to build an evaluation set.
- Add authentication, durable storage and wider reason code coverage.
