# Privacy Policy — Cairo

_Last updated: 2026-05-07_

Cairo is a Claude Code plugin that diagnoses your AI product's pricing system. This document describes exactly what data Cairo handles and what it does **not** do.

## Summary

**Cairo collects no data.** Everything runs on your local machine. No code, findings, scores, file paths, or any telemetry are transmitted to Klaim, Anthropic, or any third party.

## What Cairo accesses on your machine

When you invoke `/cairo:pricing-diagnose` (or the manual-install equivalent `/pricing-diagnose`), Cairo asks Claude Code to:

1. Read files in the current project (source code, `package.json`, config files, pricing-related components)
2. Read git history (`git log` for pricing-related commits)
3. Optionally read related repositories you point at via `--related-repos`

This data is read by your local Claude Code instance. It is processed in the conversation between you and Claude. It is not sent to Klaim or any service operated by Klaim.

## What Cairo does NOT do

- ❌ No analytics — Cairo contains no analytics SDKs, no tracking pixels, no telemetry endpoints
- ❌ No "phone home" — Cairo never makes outbound HTTP requests to Klaim or any third party
- ❌ No usage data — Klaim does not know whether or when you ran Cairo
- ❌ No score reporting — Your diagnosis scores are never collected or aggregated by Klaim
- ❌ No code transmission — Your source code and findings stay on your machine

## Optional `--external` flag

If you explicitly run `/cairo:pricing-diagnose --external`, the skill instructs Claude Code to perform external web searches (e.g., on Reddit, ProductHunt, X) for user feedback about competitor products you specify. These searches are routed through Claude's standard web search tooling and follow Anthropic's privacy practices, not Klaim's. **Cairo itself initiates no network calls.**

## Anonymous aggregation (not implemented)

The `SKILL.md` mentions a future opt-in feature for anonymous score aggregation. **This feature is not implemented in the current release.** If it is ever added, it will:

- Be off by default
- Require explicit consent on first run
- Transmit only numeric scores (e.g., "Signal 5: 6/10"), never code, file paths, project names, or identifying information
- Be documented in this Privacy Policy with a changelog entry

Until that day, this section is a placeholder for transparency about future intent. As of v1.0.0, Cairo is fully offline.

## Data Klaim has about you

If you discovered Cairo through `getklaim.com`, Klaim's website may have collected standard analytics (separate privacy policy applies to that website). The Cairo plugin itself contributes nothing to that.

If you have **never visited** `getklaim.com` and **only installed Cairo** via the Anthropic plugin marketplace or by cloning the repo, Klaim has no record that you exist.

## Reporting a privacy concern

If you believe Cairo is doing something this policy says it does not, please open an issue:

→ https://github.com/getklaim/cairo/issues

Or email: hello@getklaim.com

## Changes to this policy

If Cairo's data handling ever changes, this file will be updated and the change will appear in the repository's git history. Material changes will also be noted in the GitHub release notes for the version that introduces them.

---

**TL;DR:** Cairo runs locally, sends nothing anywhere, and Klaim has zero visibility into your usage of it. This is intentional and structural — there is no opt-out needed because there is no opt-in to anything.
