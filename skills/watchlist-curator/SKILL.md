---
name: watchlist-curator
description: Resolve loose user requests into confirmed KRX/US Hermes watchlist additions, then update watchlist JSON only after explicit approval.
---

# watchlist-curator

Use this skill when a user asks to add companies or tickers to the Hermes KRX or US daily chart pulse watchlists, especially from loose text such as "삼성전자랑 Circle 추가해줘".

## Safety Rule

Never edit a real watchlist on the first pass. Do not infer tickers or companies yourself. Run the CLI, show the CLI's `humanSummary` and `confirmationPrompt` exactly, then wait for explicit user confirmation. Only run `apply` after the user clearly confirms the exact entries.

## Workflow

1. Do not parse or guess the requested companies yourself.
2. Run `propose`:

```bash
node "$HERMES_HOME/skills/watchlist-curator/bin/watchlist-curator.js" propose --input "<user text>"
```

From this repository during development:

```bash
node skills/watchlist-curator/bin/watchlist-curator.js propose --input "<user text>" --offline \
  --watchlist-krx examples/watchlist.example.json \
  --watchlist-us examples/us-watchlist.example.json
```

3. Show the CLI output to the user without rewriting the proposed entries:
   - Copy the complete `humanSummary` block.
   - Copy the complete `confirmationPrompt` block.
   - If the output includes an apply command, do not edit it.
4. If the user confirms the exact entries, run the `applyCommand` from the CLI output exactly as printed.
5. If the CLI reports `ambiguous` or `unresolved`, do not run `apply`.
   - For `ambiguous`, ask the user to choose one of the printed candidates.
   - For `unresolved`, perform web search only to present candidates with sources. Do not write candidates directly to a watchlist.

Use `propose --json` or `resolve` only when you need structured data for debugging. The JSON fields are:

   - `additions`: proposed normalized `{ ticker, name, market }` entries, preserving `memo` when supplied.
   - `duplicates`: requested names already present in the relevant watchlist.
   - `ambiguous`: multiple plausible matches; ask the user to choose.
   - `unresolved`: no deterministic match; perform web search and return candidates instead of guessing.
   - `humanSummary`: Korean text to show the user as-is.
   - `confirmationPrompt`: Korean confirmation prompt to show the user as-is.
   - `applyCommand`: exact command to run after confirmation. Empty means writing is forbidden.

## Normalization

- KRX tickers are six digits.
- KRX `.KS` or KOSPI symbols map to `KOSPI`.
- KRX `.KQ` or KOSDAQ symbols map to `KOSDAQ`.
- US tickers are uppercased and deduplicated case-insensitively.
- US markets normalize to `NASDAQ`, `NYSE`, or `AMEX`.
- The watchlist item shape is `{ "ticker": "...", "name": "...", "market": "..." }`; include `memo` only when the user supplied it.

## Source Strategy

The CLI includes a small bundled fixture set for deterministic common requests and local tests. When `--offline` is not set, unresolved entries may use live lookups:

- Yahoo Finance search for US exact symbols and company-name candidates.
- Naver Finance autocomplete for KRX name candidates.

Live lookup can fail or rate-limit. If a name cannot be verified deterministically, do a normal web search, cite the source to the user, and present candidates rather than writing.

## Default Watchlists

Unless overridden, the CLI reads and writes:

- `$HERMES_HOME/config/krx-daily-chart-pulse/watchlist.json`
- `$HERMES_HOME/config/us-daily-chart-pulse/watchlist.json`

Use `--watchlist-krx` and `--watchlist-us` for tests or temporary copies.
