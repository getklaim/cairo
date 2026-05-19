---
name: pricing-diagnose
description: A skill for self-diagnosing the pricing system of an AI product. Detects the pricing model (tiered, usage-based, or hybrid) and analyzes the project codebase, git history, and configuration files. Measures pricing maturity across 6 signals (page-code consistency, experimentation velocity, hidden SKUs, billing stack maturity, unit clarity, lifecycle coverage), with model-specific deep checks — usage-based projects additionally evaluate credit carryover, decoy pricing, one-time credit purchase, cancellation flow (bonus credits vs discount), credit meter, and credit usage email. Findings include exact file paths and line numbers. Use this when a user wants to diagnose pricing defects in their own project.
---

# Pricing Diagnose Skill

## Invocation

From the project root:
- `/pricing-diagnose` — full automatic exploration (auto-detects pricing model)
- `/pricing-diagnose --quick` — compact first-pass evaluation for users who want fast feedback before a full report
- `/pricing-diagnose --signal N` — analyze only a specific signal (1-6)
- `/pricing-diagnose --pricing-model <tiered|usage-based|hybrid>` — force a specific pricing model when auto-detection is wrong
- `/pricing-diagnose --usage-deep` — run all 6 usage-based deep checks (credit carryover, decoy, one-time purchase, cancellation flow, meter, email) regardless of model
- `/pricing-diagnose --persona "<description>"` — persona scenario
- `/pricing-diagnose --root <path>` — partial monorepo analysis
- `/pricing-diagnose --related-repos <path1>,<path2>` — combined analysis across separated repos like a marketing site and billing app
- `/pricing-diagnose --external` — supplementary external review/feedback search

No URL input. The current project is the analysis target.

## Prompt-Only Quick Evaluation

Some users will not install the skill first. They may paste the following prompt directly into Claude Code from their project root to get a lightweight evaluation:

```text
Act as Cairo, a pricing-system diagnostic reviewer for AI products.

Quickly evaluate this codebase's pricing system. Do not make code changes.

1. Detect whether the pricing model is tiered, usage-based, hybrid, or unknown.
2. Inspect only the highest-signal pricing files first: pricing page/components, plan config, billing/checkout code, Stripe/Paddle integration, credit or usage ledger, lifecycle/cancel flow, and relevant git history.
3. Score these 6 signals briefly:
   - Page-code consistency
   - Experimentation velocity
   - Hidden SKUs + experimentation infrastructure
   - Billing stack maturity
   - Unit clarity
   - Lifecycle coverage
4. If the model is usage-based or hybrid, also check:
   - Credit carryover
   - Decoy pricing
   - One-time credit purchase
   - Cancellation save offer
   - Credit meter
   - Credit usage email
5. Output a compact Reality Check:
   - Pricing model + confidence
   - Diagnostic map
   - 6-signal score table
   - Top 3 findings with exact file:line, impact, and estimated effort
   - Honest limits

Keep it concise. Prefer concrete file evidence over broad advice.
```

When this prompt or `--quick` is used:
- Run a bounded first pass instead of the full 60-180 second workflow.
- Prioritize files that directly define pricing, billing, checkout, credits/usage, lifecycle, and pricing UI.
- Inspect git history only enough to estimate Signal 2; do not exhaustively enumerate all pricing-related commits.
- Output at most 3 priority findings and omit full appendices.
- If key files are missing because pricing lives in another repo, state that once in Honest Limits and suggest the full command with `--related-repos`.
- Do not use external search unless the user explicitly asks for it.

## Step 1: Automatic Project Structure Discovery + Type Classification

Discover pricing-related code and output a diagnostic map first (analysis-scope transparency).

### 1-A. Automatic Project Type Classification

Discovery results are categorized into one of four types:

| Type | Criteria | Applicable Signals |
|---|---|---|
| **billing-system** | Payment SDK + webhook handler + plan definitions all present | All 1-6 |
| **marketing-only** | Pricing page components exist but no payment SDK | 1, 2, 3 + partial Signal 5. 4, 6 → N/A |
| **hybrid** | Both marketing + billing in one repo (monorepo or merged Next.js app) | All 1-6 |
| **none** | No pricing-related code at all | Terminate immediately |

If classified as **marketing-only**:
- Mark Signals 4 and 6 as **N/A** (honesty)
- Inform the user about the `--related-repos <billing-app-path>` option
- Signals 1, 2, 3, and 5 still operate strongly (page consistency, UI change velocity, hidden SKUs, unit expression)

### 1-B. Discovery Patterns

Discovery patterns:
- Folder/file names: `billing`, `subscription`, `plan`, `pricing`, `checkout`, `credit`
- Dependencies (`package.json` / `requirements.txt` / `Cargo.toml`):
  - Payments: `stripe`, `paddle`, `chargebee`, `lemonsqueezy`
  - Experimentation: `statsig`, `growthbook`, `optimizely`, `launchdarkly`, `posthog`
- Stripe price ID patterns: `price_*`, `prc_*`
- Webhook handlers: `stripe.webhook`, `paddle.webhook` routes
- Lifecycle function names: `cancel`, `retention`, `winback`, `pause`
- Pricing page components: `PricingPage`, `TierCard`, `CheckoutForm`
- Environment variables: `STRIPE_*`, `BILLING_*`, `PADDLE_*`
- DB migrations: `plans`, `tiers`, `subscriptions` tables

Example diagnostic map output:
```
📍 Diagnostic Map
- Payment SDK: stripe@14.2.0
- Pricing definition location: src/config/plans.ts
- Billing handlers: src/api/billing/* (12 files)
- Page components: src/components/Pricing/*
- Webhook: src/api/webhooks/stripe.ts
- Feature flag system: none
- Lifecycle: cancel.ts found, retention/winback missing
```

If analysis scope is insufficient, recommend supplementing with `--root`. If there is *no* pricing-related code at all, terminate immediately and report "this project has no pricing system or it is not yet implemented."

### 1-C. Separate Handling of Dynamic/Interactive Components

The following *interactive* pricing components are *not* treated like static plan definitions:

- `PricingSimulator`, `PricingCalculator`, `FeeCalculator` types (typically 300+ lines)
- Components that *compute prices at runtime* based on user input
- Interactions that directly expose A/B variants

These components qualify for **Signal 5 (Unit Clarity)** bonus points:
- Can users *immediately verify costs against their own scenarios*? +2
- Is the calculation formula *transparently exposed* in code/UI? +1
- Is the input → output mapping *accurate* (matches backend fees)? +1

Add an *interactive component report section* to the report as a separate analysis.

### 1-D. Combined Analysis of Related Repos

When using the `--related-repos` option:
- Main repo = primary analysis target (current directory)
- Related repos = secondary analysis (e.g., billing app next to the marketing site)
- Specify *which repo each signal was measured from*
- Additionally verify *plan definition consistency* between the two repos (what marketing advertises ↔ what billing actually charges)

### 1-E. Pricing Model Detection (separate axis from project type)

After project type, classify the **pricing model** — this changes which signals are weighted highest and which deep checks run.

| Model | Detection Signals | Weight Profile |
|---|---|---|
| **tiered** | Fixed plan enums (Free/Pro/Enterprise), monthly flat charge, no per-call deduction, plan tier guards in feature checks | Signals 1, 3, 6 weighted 1.5× |
| **usage-based** | Credit/token deduction functions, balance state in DB (`credits_remaining`, `tokens_balance`), metered Stripe products, top-up endpoints | Signals 4, 5, 6 weighted 1.5× + 6 usage-based deep checks |
| **hybrid** | Both tier enums AND credit deduction (most common for AI products like Cursor/Lovable/Replicate) | All signals at 1× + all 6 usage-based deep checks |
| **unknown** | Conflicting/insufficient signals | Ask user for `--pricing-model` override |

**Detection patterns for usage-based:**
- Functions: `deductCredits`, `consumeUsage`, `meterUsage`, `chargeTokens`, `recordUsage`
- DB columns: `credits_balance`, `usage_remaining`, `tokens_used`, `quota_remaining`
- Stripe Meters API (`stripe.billing.meterEvents.create`) or `usage_records`
- Top-up routes: `/api/credits/purchase`, `/billing/topup`, `buyCredits`
- UI components: `CreditBalance`, `UsageMeter`, `TokenCounter`, `CreditMeter`
- Cron jobs: monthly credit reset, carryover calculations

**Detection patterns for tiered:**
- Enums: `PlanType.FREE/PRO/ENTERPRISE`, `Tier.STARTER/GROWTH/SCALE`
- Guards: `if (user.plan === 'pro')`, `requireTier('enterprise')`
- Stripe Subscriptions (not Meters), single recurring price per plan
- No usage tracking; features gated by plan tier only

The diagnostic map output **must** state the detected pricing model:
```
📍 Diagnostic Map
- Project type: hybrid (marketing + billing)
- Pricing model: usage-based  ← drives signal weighting
- Payment SDK: stripe@14.2.0
  ...
```

## Step 2: Collect 6 Signals

> Each signal has a **tiered focus** and a **usage-based focus**. Run the focus matching the detected pricing model. For **hybrid**, run both and report separately.

### Signal 1 — Page-Code Consistency

Alignment between code truth and page exposure.

**Tiered focus** (collect):
- Tier data from pricing page components
- Backend plan definition objects (`PLANS`, `plan-config.ts`, etc.)
- Number of plans in code vs. number exposed on the page → *hidden SKU gap*
- Backend limits vs. limits displayed on the page

**Usage-based focus** (collect):
- Credit pack SKUs in code vs. credit packs displayed on the page
- Per-unit price displayed (e.g., "$0.01 per credit") vs. actual deduction rate in code
- Stripe Meters product config vs. UI labels
- "Free credits per month" advertised vs. actual quota reset cron
- → *Decoy pricing check*: is there a plan/pack deliberately priced to make another look better? Look for a plan with worse $/credit ratio that no telemetry/A/B suggests anyone picks. Flag if found AND if it's clearly priced to anchor (Decoy Pricing — see Step 2.5)

### Signal 2 — Experimentation Velocity

```bash
# Change frequency of pricing-related files
git log --since="1 year ago" --oneline -- "**/pricing*" "**/billing*" "**/plan*" "**/checkout*" "**/credit*"

# Commits with pricing keywords in messages
git log --since="1 year ago" --grep="price\|pricing\|tier\|plan\|credit" --oneline

# Last change timestamp
git log -1 --format="%ai %s" -- "**/pricing*"
```

Outputs:
- 12-month count of pricing-related commits
- Last change date
- Traces of A/B setup/rollback commits

### Signal 3 — Hidden SKUs + Experimentation Infrastructure

Collect:
- Number of plans in code vs. number exposed (gap = hidden SKUs)
- A/B tool SDK import inspection
- Feature flag config files (`featureFlags.json`, `.flags.yaml`, `flags.config.ts`)
- Pricing variants tied to environment variables (`PRICE_VARIANT_A`, etc.)

Scoring ladder:
- Specialized A/B tool (Statsig/Growthbook/Optimizely): 9/10
- PostHog feature flags: 7/10
- Custom feature flag system: 5/10
- Code if/else branching: 3/10
- No experimentation infrastructure: 1/10

### Signal 4 — Billing Stack Maturity

**Tiered focus** (collect):
- Payment SDK version (`package.json` / `requirements.txt`)
- Stripe Subscriptions usage (`stripe.subscriptions.create`)
- Add-on libraries (`stripe-billing-portal`, `@stripe/connect`)
- Lines of code in custom ledger (`src/billing/ledger/*`)

**Tiered scoring:**
- Stripe Subscriptions only: 6/10 (correct for tiered)
- Stripe Subscriptions + Billing Portal: 8/10 (appropriate)
- Custom subscription tracking: 3/10 (reinventing the wheel)

**Usage-based focus** (collect):
- Stripe Meters API usage (`stripe.billing.meterEvents.create`) or legacy `usage_records`
- Custom credit ledger code (look for double-entry patterns, idempotency keys)
- One-time credit purchase routes (`/api/credits/purchase`, `createCheckoutSession` for one-shot price)
- Webhook handlers for `invoice.paid`, `customer.subscription.deleted`, `checkout.session.completed`
- Idempotency on credit grants (prevent double-credit on webhook retry)

**Usage-based scoring:**
- Stripe Meters + idempotent ledger + one-time purchase route: 9/10 (mature)
- Stripe Meters only, no top-up: 6/10 (missing One-Time Credit Purchase — see Step 2.5)
- Custom metering, no Stripe Meters: 4/10 (in-house build risk)
- Manual credit deduction without idempotency: 2/10 (data integrity risk)
- Mixed payment systems: 3/10 (incomplete migration)

### Signal 5 — Unit Clarity

The most important signal (the structural defect of AI pricing). **For usage-based, this is the make-or-break signal.**

**Tiered focus** (collect):
- Feature limit definitions per tier (`PRO.maxSeats`, `ENTERPRISE.apiCallsPerMonth`)
- Limit enforcement in code vs. limit displayed on the pricing page
- Overage behavior (hard cap? soft warning? auto-upgrade?)
- Are limits expressed in user-meaningful units (e.g., "10,000 emails") or technical ones ("10k API tokens")?

**Tiered scoring:**
- Limits match between code and page, expressed in user units: 9/10
- Limits match but expressed in technical units only: 6/10
- Limits in code don't match page: 2/10

**Usage-based focus** (collect):
- Credit/usage deduction functions (`deductCredits`, `consumeUsage`, `meterUsage`)
- Inside the function:
  - Number of conditional branches
  - Parameters used (model, prompt length, tool usage, etc.)
- User-facing UI representation, especially the **Credit Meter** component (real-time balance display)
- Unit definition documentation (README, docs/, code comments)
- "What costs 1 credit?" — is there a public table mapping action → credit cost?
- Credit meter update latency: real-time vs. polling vs. cached daily

**Usage-based scoring:**
- Algorithm exposed transparently + meter live + cost table public: 9/10
- Algorithm complex but meter accurate: 7/10
- Meter present but algorithm hidden / complex branching: 4/10
- No meter OR meter shows wrong value: 1/10 (most common failure mode)

Report format:
```
🔎 Signal 5 — Unit Clarity (N/10) [pricing model: <model>]

Algorithm location: <file>:<line> <function name>
Complexity: N conditional branches
User-facing exposure: <component> "<description>"
Credit Meter: <component path> — update strategy: <realtime|poll|cached>
Cost table: <docs path or "MISSING">
Gap: N variables in algorithm vs. one-line exposure
```

### Signal 6 — Lifecycle Code Coverage

Checklist (1 point each, 6 total). Each stage has different markers depending on pricing model.

| Stage | Tiered marker | Usage-based marker |
|---|---|---|
| Exposure | PricingPage with tier comparison | PricingPage with credit packs + per-unit price |
| Signup | onboarding/signup flow → default tier | signup grants free credits + welcome usage flow |
| Usage | per-tier feature usage tracking | **Credit Meter** in UI + **Credit Usage Email** at thresholds (50%/80%/100%) |
| Expansion | upgrade/downgrade nudge to next tier | low-balance nudge → top-up CTA + auto-recharge option |
| Churn | cancel handler + save offer (discount, pause) | cancel handler + **bonus credits offer** OR **% discount offer** branching |
| Recovery | win-back trigger / re-engagement email | dormant credit reactivation + carryover honor on return |

Specify the key file path for each stage. If there is no handler at all, report "0/6 stages covered."

**Usage-based extra checks within Lifecycle:**
- Churn stage must have **at least one** of: bonus-credit retention offer, % discount, or pause subscription. Score 0 on Churn if all three are missing
- Usage stage requires both meter AND email — partial credit if only one exists

## Step 2.5: Usage-Based Pricing Deep Checks

Run when pricing model is `usage-based` or `hybrid`, or when `--usage-deep` is set. Each check produces a Pass/Fail/Partial with a file:line citation. Aggregate result feeds into the relevant signal score.

### Check U1 — Credit Carryover

**What to find:**
- Cron/scheduled job that resets credits monthly: `resetMonthlyCredits`, `creditRefillJob`
- Inspect logic: does unused balance carry over (rollover)? Roll up to a cap? Or zero out?
- Public docs/UI explaining the rollover policy

**Scoring:**
- Carryover with cap + clearly documented in UI: Pass
- Carryover undocumented (silent feature): Partial — risk of support tickets when users don't notice
- Hard reset to zero with no warning: Fail — top user complaint pattern

**Feeds:** Signal 5 (Unit Clarity) and Signal 6 (Recovery)

### Check U2 — Decoy Pricing

**What to find:**
- 3+ credit packs/plans where one has clearly worse $/credit ratio
- Telemetry/A/B history showing nobody picks the decoy
- Or absence of any anchor pack (only one pack exists → no anchoring)

**Scoring:**
- Intentional decoy with conversion uplift evidence: Pass (sophisticated pricing)
- Single pack only, no anchoring strategy: Partial
- Multiple packs with random/unintentional pricing: Fail (leaving conversion on the table)

**Feeds:** Signal 1 (Page-Code Consistency) and Signal 3 (Experimentation)

### Check U3 — One-Time Credit Purchase

**What to find:**
- Route/handler for one-time credit top-up without subscription: `/api/credits/purchase`, `createOneTimeCheckout`
- Stripe Checkout `mode: 'payment'` (not `subscription`) for credit packs
- UI button: "Buy more credits" / "Top up"

**Scoring:**
- One-time purchase available + auto-recharge option: Pass
- One-time purchase only via support contact: Partial
- No one-time option (must upgrade subscription): Fail — common conversion blocker

**Feeds:** Signal 4 (Billing Stack) and Signal 6 (Expansion)

### Check U4 — Cancellation Flow: Bonus Credits vs Discount

**What to find:**
- Cancel handler (`cancelSubscription`, `/api/billing/cancel`)
- Branching logic for save offers: bonus credits path AND/OR percentage discount path
- A/B test on which offer converts better (look for `cancelOffer` variant in feature flags)

**Scoring:**
- Both options present with A/B routing: Pass (mature retention)
- Only one option (either bonus credits OR discount, e.g., 50% off): Partial — common starting point
- No save offer at all (cancel button → goodbye): Fail

**Feeds:** Signal 6 (Churn)

### Check U5 — Credit Meter

**What to find:**
- UI component displaying current balance: `CreditMeter`, `UsageBar`, `BalanceWidget`
- Update mechanism: WebSocket/SSE (real-time), polling (every N seconds), or cached (page reload)
- Visibility: persistent in nav vs. hidden in settings

**Scoring:**
- Persistent in main nav + real-time updates + low-balance color states: Pass
- Present but hidden behind a click OR cached/stale: Partial
- No meter at all (users don't know their balance until depleted): Fail — root cause of "credits disappeared" complaints

**Feeds:** Signal 5 (Unit Clarity) and Signal 6 (Usage)

### Check U6 — Credit Usage Email

**What to find:**
- Email handler with usage thresholds: `sendUsageAlert`, `creditThresholdEmail`
- Trigger thresholds defined (e.g., 50%, 80%, 100% consumed)
- Email template content: includes top-up CTA? Forecasted depletion date?

**Scoring:**
- Multiple thresholds + actionable CTA + depletion forecast: Pass
- Single threshold only (e.g., only at 0%): Partial
- No usage email at all: Fail — leads to surprise depletion churn

**Feeds:** Signal 6 (Usage)

### Aggregate Usage Deep Check Score

```
🧪 Usage-Based Deep Checks (X/6 Pass, Y Partial, Z Fail)

U1 Credit Carryover         : <Pass|Partial|Fail> — <file:line>
U2 Decoy Pricing            : <Pass|Partial|Fail> — <file:line>
U3 One-Time Credit Purchase : <Pass|Partial|Fail> — <file:line>
U4 Cancellation Flow        : <Pass|Partial|Fail> — <file:line>
U5 Credit Meter             : <Pass|Partial|Fail> — <file:line>
U6 Credit Usage Email       : <Pass|Partial|Fail> — <file:line>
```

This block appears in the report **only** when pricing model is usage-based or hybrid.

## Step 3: Supplementary External Verification (Optional)

When the `--external` flag is set:
- Search user reviews (Reddit, X, ProductHunt)
- Competitor comparison (`--compare <competitor-url>`)

OFF by default. Only when explicitly opted in.

## Step 4: Reality Check Report

### Tone Principles

90% self-diagnosis value + 10% mention of integrated solution options. Provide *exact code location + estimated impact + estimated effort* so users can fix things themselves (or feed the finding to an AI assistant). **Do not write fix code or pseudo-code** — a precise `file:line` plus a one-line description of the defect is enough; the user will apply the fix with their own tooling.

### Anti-redundancy Rules

- **Omit, don't placeholder.** When a section does not apply (e.g., usage-based deep checks for a tiered project, Signal 4/6 for marketing-only), **remove the section entirely**. Do not render a "SKIPPED" or "N/A — this doesn't apply because…" block. A single N/A cell in the 6-signal table is enough; one footnote in Honest Limits is enough.
- **State scope limits once.** Scope/repo-boundary caveats (e.g., "billing lives in a separate repo") belong in the closing Honest Limits list. Do not repeat them in the Diagnostic Map, in per-signal rows, and in the limits list — pick the limits list.
- **References block is minimal.** Do not list skill version or re-list the analysis commit (it's already in the header). Only include items the reader cannot derive from the rest of the report.

### Output Template

```markdown
# 📊 Pricing Reality Check — <project-name>

Overall maturity: <score>/10 (<label>)
Pricing model: <tiered|usage-based|hybrid>     ← drives weighting
Analysis time: <ISO datetime>
Analysis commit: <git short-hash>

## 🗺️ Diagnostic Map
- Project type: <billing-system|marketing-only|hybrid>
- Pricing model: <tiered|usage-based|hybrid>
- Payment SDK: <list with versions>
- Pricing definition location: <files>
- Analyzed git history: <range>

## 📊 6-Signal Diagnosis

| Signal | Score | One-line diagnosis | Key location |
|---|---|---|---|
| 1. Page-Code Consistency | x/10 | ... | <file> |
| 2. Experimentation Velocity | x/10 | ... | git log |
| 3. Hidden SKU/Experimentation | x/10 | ... | <file> |
| 4. Billing Stack | x/10 | ... | package.json |
| 5. Unit Clarity | x/10 | ... | <file:line> |
| 6. Lifecycle | x/6 | ... | <files> |

## 🧪 Usage-Based Deep Checks

**Render this whole section only when pricing model is usage-based or hybrid. Otherwise OMIT — do not leave a "skipped / N/A" placeholder.**

| Check | Result | Location |
|---|---|---|
| U1. Credit Carryover | Pass/Partial/Fail | <file:line> |
| U2. Decoy Pricing | Pass/Partial/Fail | <file> |
| U3. One-Time Credit Purchase | Pass/Partial/Fail | <file:line> |
| U4. Cancellation Flow (bonus credits vs discount) | Pass/Partial/Fail | <file:line> |
| U5. Credit Meter | Pass/Partial/Fail | <component> |
| U6. Credit Usage Email | Pass/Partial/Fail | <file:line> |

## 🚨 Priority Findings (Top 3)

Top 3 sorted by impact × cost-to-fix.

### 1. <Finding title>
- Location: `<file>:<line>` (multiple locations allowed if the defect spans files)
- Current state: <one or two sentences of fact — what the code/page actually does today>
- Impact: <support ticket/trust/revenue estimate, explicitly marked as inferred when it is>
- Estimated effort: <hours or "trivial" / "half-day" / "1 day">

### 2. <same format>
### 3. <same format>

## 🔧 Items You Can Fix Directly (All)

[All findings — in priority order]

## 🛠️ Integrated Solution Option

Implementing the above findings directly takes about <X-Y weeks>.

Fast path:
- Klaim — covers [1, 2, 3, 5] of the 6 signals: klaim.com/integrate
  - *Parts of Signal 6 (lifecycle) are upcoming features*
  - *Signal 4 sits on top of Stripe*

Partial solutions:
- Statsig / GrowthBook — A/B testing (Signal 3)
- Stripe Billing portal — partial lifecycle (Signal 6)

## 🔗 References (optional)
- Analyzed files (full): <appendix — only if the reader will plausibly want the full list; otherwise omit this section entirely>

## ⚠️ Honest Limits
- <one bullet per real limitation: stale telemetry, separate-repo scope, inferred-impact estimates, etc.>
- <do not repeat limits already implied by N/A cells in the table — only include ones the reader cannot derive on their own>
```

## Limitations (Honestly Stated)

- If git history is very short, Signal 2 is inaccurate → mark as "insufficient analysis history"
- Automatic monorepo discovery may miss some folders → recommend `--root`
- Custom billing systems weaken Signal 4 inference
- Pricing model auto-detection can be wrong for *transitional* products (mid-migration from tiered → usage-based). When confidence is low, prompt for `--pricing-model` override
- Decoy pricing detection (U2) requires telemetry signals or explicit anchor patterns — without them, only flagged as "possible decoy"
- Impacts like "30% of support tickets" are *estimates* based on code signals (not statistically calibrated)
- Cannot infer intent outside the system (e.g., cannot distinguish deliberate ambiguity from inexperience)

Within the report, sections where limitations apply are *explicitly* marked.

## Usage Examples

```
$ cd ~/projects/our-ai-product
$ /pricing-diagnose
→ Diagnostic map → pricing model detection → 6-signal analysis → Reality Check Report
  (+ Usage-Based Deep Checks if model is usage-based or hybrid)

$ /pricing-diagnose --quick
→ Bounded first pass → compact Reality Check → top 3 findings only

$ /pricing-diagnose --signal 5
→ Deep dive into Signal 5 only

$ /pricing-diagnose --pricing-model usage-based
→ Force usage-based weighting + run all 6 deep checks

$ /pricing-diagnose --usage-deep
→ Run only the 6 usage-based deep checks (U1-U6)

$ /pricing-diagnose --persona "5-person design agency"
→ Persona cost simulation
```

## Operational Notes

- Execution time: 60-180 seconds (depends on project size)
- External transmission: OFF by default. Scores transmitted only with anonymous aggregation opt-in (code contents are *never* transmitted)
- Security: All analysis runs locally on the user's machine
