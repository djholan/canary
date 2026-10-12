# Canary: AI Credit-Stress Alert System

Canary watches credit-stress signals (credit default swaps, bond spreads, debt raises, SEC filings, price shocks) for a small watchlist of AI-infrastructure companies and pushes alerts to an iPhone via ntfy. Goal: demo-ready, reliable, zero babysitting.

## Architecture (two layers)

1. **Watcher** (this repo, Python, GitHub Actions cron)
   - Dumb, fast, free. No LLM calls.
   - Fetches Google News RSS, SEC EDGAR filings, and prices (yfinance).
   - Dedupes against `state/state.json`, applies keyword + threshold rules.
   - Sends obvious alerts straight to ntfy (new 8-K / 424B prospectus, price drop > 5%).
   - For ambiguous candidates, POSTs to the Claude Code routine `/fire` endpoint with a short text payload so the Analyst can investigate.
2. **Analyst** (Claude Code routine, runs in the cloud on the Max plan)
   - Scheduled pre-market and post-close digests, plus on-demand runs fired by the Watcher.
   - Uses web search to verify, scores severity 1-10, extracts CDS levels in bp, pushes to ntfy.
   - Its prompt lives in `routine-prompt.md`. The routine does not commit to this repo.

## Watchlist

Defined in `config/watchlist.yaml`. Core: SPCX (SpaceX), ORCL (Oracle), CRWV (CoreWeave), NBIS (Nebius), AVGO (Broadcom). Optional contagion watch: NVDA.
Never hardcode SEC CIKs. Resolve them at runtime from https://www.sec.gov/files/company_tickers.json and cache in state.

## Layout

```
canary/
  canary/
    __init__.py
    main.py          # entrypoint: python -m canary.main
    config.py        # loads watchlist.yaml + env
    sources/
      news.py        # Google News RSS
      edgar.py       # SEC EDGAR filings (8-K, 10-Q, 10-K, 424B*, 6-K, 20-F)
      prices.py      # yfinance quotes + earnings dates
    rules.py         # keyword filters, thresholds, severity hints
    notify.py        # ntfy push + routine fire
    state.py         # load/save state/state.json (seen ids, CIK cache, last prices)
  config/watchlist.yaml
  state/state.json
  tests/
  .github/workflows/watch.yml
  routine-prompt.md
  CLAUDE.md
```

## Conventions

- Python 3.12, type hints, small pure functions. Dependencies: `feedparser`, `httpx`, `yfinance`, `pyyaml`, `pytest`. Ask before adding anything else.
- Every network call: 10s timeout, one retry, never crash the whole run because one source failed. Log and continue.
- SEC requires a descriptive User-Agent with a contact email: read it from env `SEC_USER_AGENT`.
- Dedupe key: sha1 of (source + url or accession number). Keep at most 5,000 seen ids, trimming oldest.
- `--dry-run` flag prints alerts instead of sending. Default for local runs.
- Keep `state/state.json` small and human-readable (sorted keys, indent 2).

## Notification format (never change without asking)

Title: `{dot} {TICKER} · {headline metric}`, e.g. `🔴 SPCX · CDS 195bp record high`
Body: one line of why, then the source link.
Dots: 🔴 severity 8-10 (ntfy priority 5), 🟠 5-7 (priority 4), 🟢 digest / info (priority 3).
Tap action opens the source URL (ntfy `Click` header). No em dashes anywhere in alert text.

## Secrets / env

- `NTFY_TOPIC`: private ntfy topic name (GitHub secret + local `.env`, never committed)
- `SEC_USER_AGENT`: e.g. `Canary research you@example.com`
- `ROUTINE_FIRE_URL`, `ROUTINE_FIRE_TOKEN`: Claude Code routine API trigger (GitHub secrets)
- `.env` is gitignored. Load it locally with a tiny loader, no extra dependency.

## Commands

- Run once locally: `python -m canary.main --dry-run`
- Send a real test push: `python -m canary.main --test-push`
- Tests: `pytest -q`

## Definition of done for any change

1. `pytest -q` passes.
2. `python -m canary.main --dry-run` runs end to end with no exceptions.
3. If notification code changed, `--test-push` delivered to the phone (ask me to confirm).
