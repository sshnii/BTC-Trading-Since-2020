# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository nature

This is a **data archive**, not a software project. It has no build, no tests, no runtime, no package manifest. The entire repo is a flat root containing:

- CSV mirrors of BitMEX `/api/v1/...` endpoints (file names match the endpoint path: `api-v1-<segment>-<segment>.csv`)
- One derived series (`derived-equity-curve.csv`) and one derived figure (`cumulative-performance.png`)
- `manifest.json` — the build's checksums, row counts, time ranges, columns per file, and methodology string
- Bilingual READMEs (`README.md`, `README.zh-CN.md`)

There is no code to run. Tasks here are almost always: update the data, update the README/manifest, or answer questions about the dataset.

## Update workflow (the canonical task)

Per `README.md` § "Update policy", each refresh must:

1. Pull the full raw dataset from BitMEX.
2. Rebuild the public root with the **same filenames** (filenames are stable; versioning is via Git, not via the filename).
3. Apply the privacy rules below before committing.
4. Commit, then tag the commit `data-YYYY-MM-DD` (matches `manifest.json` → `version_strategy.tag_pattern`).
5. Regenerate `manifest.json` so `sha256`, `size_bytes`, `rows`, and `first_time`/`last_time` match the new files; bump `generated_at` and the `dataset_window`.
6. If `derived-equity-curve.csv` is rebuilt, refresh the matching numbers in `manifest.json` → `equity_curve_summary` and the "High-level facts" block in both READMEs.
7. If `cumulative-performance.png` is regenerated, the README references it as `cumulative-performance.png?v=<short-hash>` for cache-busting — update the query string when the image changes (see commit `76010fb`).

The actual build pipeline lives outside this repo (see `manifest.json` → `root: /Users/jarvis/.hermes/analysis_outputs/...`); this repo is the published output, not the generator.

## Privacy rules (must be preserved on every update)

These come from `README.md` § "Privacy policy" and `manifest.json` → `privacy_policy_version`. Do not weaken them when editing or regenerating files:

- `account` column is removed from every published file where it existed.
- `api-v1-user-walletHistory.csv`: `tx` and `text` removed; `address` redacted **only** when `transactType` is `Withdrawal` or `Transfer` (deposits keep their address).
- `api-v1-order.csv`: `text` removed.
- `api-v1-execution-tradeHistory.csv`: `text` is **intentionally kept** — it explains fills, funding, and settlements. Do not strip it.
- `/api/v1/user` profile payloads and `/api/v1/execution` lifecycle-noise rows beyond `tradeHistory` are not published.
- No login/IP/device data, no chain tx hashes from wallet history.

If you regenerate any CSV, verify the redactions still hold before committing.

## Derived equity curve methodology

`derived-equity-curve.csv` is the only file in this repo that is *computed*, not mirrored. The methodology is locked and auditable — match it exactly when rebuilding:

- **Baseline**: first fully-funded XBT wallet balance after the **last** completed deposit on the first trading day (`2020-05-01T14:39:40.387Z`, `1.83953943` XBT).
- **Wealth scope**: XBT wallet balance + USDt wallet balance, with USDt converted to XBT using the **latest observed internal XBT/USDT conversion or spot rate** in the published wallet ledger. It is *not* a full mark-to-market NAV across all assets BitMEX ever credited.
- **Cash-flow normalization** after baseline: completed withdrawals are added back, completed deposits are subtracted, internal `Transfer` rows are neutralized.
- **Internal swaps**: `Conversion` events and `SpotTrade` XBT↔USDt pairs are treated as internal wallet swaps, not gains/losses.
- **Event ordering**: use `timestamp` first when BitMEX provides both `timestamp` and `transactTime`. `transactTime` is preserved as the original exchange field but is **not** the accounting-effective order (see commit `e154663` for why — fixes a delayed-withdrawal ordering bug).
- The current methodology version string is `xbt-usdt-wallet-equivalent-v2` (column `methodologyVersion` in the CSV). Bump it if the methodology actually changes.

When you change the methodology, update: the column, `manifest.json` → `equity_curve_summary.methodology`, and the README "Derived performance methodology" section in **both** language files.

## Bilingual README invariant

`README.md` (English) and `README.zh-CN.md` (Chinese) are siblings and must stay in sync. Any factual edit (numbers, dataset window, methodology, privacy rules, file table) must land in both. They cross-link at the top via `>[English](README.md) | [中文](README.zh-CN.md)`.

## File-size note

Several CSVs are large (`api-v1-execution-tradeHistory.csv` ≈ 65 MB, `derived-equity-curve.csv` ≈ 4 MB). Prefer `Read` with explicit `offset`/`limit`, or `head`/`tail` via Bash, over reading whole files into context. Column lists are already enumerated in `manifest.json` → `files[*].columns` — consult that first instead of re-discovering schema.

## Common conventions

- Commit message style (see `git log`): `<type>: <imperative summary>`, where type is `data:` for dataset/derived-output changes, `docs:` for README/manifest-text changes. Keep the line short.
- Symbol `XBT` = Bitcoin (BitMEX-native ticker for BTC); `XBt` (lowercase t) is the satoshi-scale variant in BitMEX wallet ledgers. Both appear in the data — they are not typos.
- Live current-state companion site: `https://wsnb.online`. This repo is the historical layer only; do not try to fetch live state from here.
