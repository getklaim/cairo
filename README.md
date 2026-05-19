# Cairo — Pricing Diagnose

> A Claude skill that self-diagnoses the pricing system of your AI product.
>
> Run `/pricing-diagnose` inside your project. Get a Reality Check Report with file paths, line numbers, estimated impact, and fix effort.

---

## Quick prompt-only check

Want to try Cairo before installing the skill? Open Claude Code at your project root and paste this:

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

Keep it concise. Prefer concrete file evidence over broad advice.
```

For the full repeatable workflow, install the plugin or skill below and run `/pricing-diagnose`.

---

## What it does

Cairo reads your codebase, git history, and config files — then scores your pricing maturity across **6 signals** and (for usage-based products) **6 deep checks**.

It detects whether your pricing model is:
- **tiered** — fixed plans (Free / Pro / Enterprise)
- **usage-based** — credits, tokens, metered billing
- **hybrid** — both (most AI products)

…and weights the signals + runs the right deep checks accordingly.

### The 6 signals

| # | Signal | What it measures |
|---|---|---|
| 1 | Page-Code Consistency | What your pricing page advertises vs. what your code actually charges |
| 2 | Experimentation Velocity | How often pricing changes ship (git history) |
| 3 | Hidden SKUs + Experimentation Infrastructure | Plans that exist in code but aren't shown; A/B SDK presence |
| 4 | Billing Stack Maturity | Stripe Meters? Custom ledger? Idempotent webhooks? |
| 5 | Unit Clarity | Does the user understand what 1 credit / 1 unit costs them? |
| 6 | Lifecycle Code Coverage | Exposure → Signup → Usage → Expansion → Churn → Recovery |

### The 6 usage-based deep checks (run for usage-based / hybrid)

| # | Check | Why it matters |
|---|---|---|
| U1 | Credit Carryover | Hard month-end reset is the #1 user complaint |
| U2 | Decoy Pricing | No anchor pack → conversion left on the table |
| U3 | One-Time Credit Purchase | Missing top-up forces unwanted subscription churn |
| U4 | Cancellation Flow (bonus credits vs % discount) | No save offer = 100% churn |
| U5 | Credit Meter | No live balance = "credits disappeared" support tickets |
| U6 | Credit Usage Email | No threshold alerts = surprise depletion churn |

---

## Install

### Option A — Plugin (recommended)

This repo doubles as its own Claude Code plugin marketplace. Add it once, then install:

```
/plugin marketplace add getklaim/cairo
/plugin install cairo@klaim
```

To run the plugin from a local checkout (e.g. for development):

```bash
git clone https://github.com/getklaim/cairo.git
claude --plugin-dir ./cairo
```

**Invocation when installed as a plugin:**

```
/cairo:pricing-diagnose
```

### Option B — Manual single-file install

Cairo's skill works standalone — just drop the `SKILL.md` into a Claude skills directory.

```bash
# Project-scoped
mkdir -p .claude/skills/pricing-diagnose
curl -L https://raw.githubusercontent.com/getklaim/cairo/main/skills/pricing-diagnose/SKILL.md \
  -o .claude/skills/pricing-diagnose/SKILL.md

# Or user-scoped (all projects)
mkdir -p ~/.claude/skills/pricing-diagnose
curl -L https://raw.githubusercontent.com/getklaim/cairo/main/skills/pricing-diagnose/SKILL.md \
  -o ~/.claude/skills/pricing-diagnose/SKILL.md
```

**Invocation when installed manually:**

```
/pricing-diagnose
```

Reload Claude Code after installing either way.

---

## Use

From the root of any project. Replace `/pricing-diagnose` with `/cairo:pricing-diagnose` if you installed via the plugin path.

```bash
# Full diagnosis (auto-detects pricing model)
/pricing-diagnose

# Fast first-pass diagnosis
/pricing-diagnose --quick

# Focus a single signal
/pricing-diagnose --signal 5

# Force pricing model when auto-detection is wrong
/pricing-diagnose --pricing-model usage-based

# Run only the 6 usage-based deep checks
/pricing-diagnose --usage-deep

# Combined analysis across split repos (marketing site + billing app)
/pricing-diagnose --related-repos ../billing-app
```

You'll get a Reality Check Report with:
- Pricing model + diagnostic map
- Score per signal (with the file path that drove the score)
- Top 3 priority findings with exact file locations and impact estimates
- Estimated effort per finding

See [skills/pricing-diagnose/SKILL.md](./skills/pricing-diagnose/SKILL.md) for the full spec.

---

## Privacy & security

- **Runs entirely on your machine.** Cairo analyzes files locally — no code, no findings, no scores leave your environment.
- **No telemetry.** Cairo contains no analytics SDKs and makes no outbound network calls.
- **Read-only.** Cairo never modifies your code. It produces a report.

Full details: [PRIVACY.md](./PRIVACY.md)

---

## What Cairo will *not* tell you

Honest limitations (also in `SKILL.md`):

- Short git history → Signal 2 (experimentation velocity) becomes unreliable
- Custom in-house billing systems weaken Signal 4 inference
- Decoy pricing detection (U2) without telemetry can only flag *possible* decoys
- Impact estimates ("~30% of support tickets") are heuristics from code signals — not statistically calibrated
- Cannot infer intent (deliberate ambiguity vs. inexperience look the same in code)

The report explicitly marks sections where these limitations apply.

---

## How does this compare to…

| Tool | Approach | Cairo difference |
|---|---|---|
| Tierly | External page scoring (11 axes) | Cairo reads your *code*, not just your page |
| LLM Price Check | Token unit-price comparison | Cairo evaluates how *your product* exposes cost |
| Helicone / Langfuse | LLM observability | Cairo diagnoses the pricing structure itself |
| ProfitWell / Baremetrics | Subscription metrics | Cairo doesn't need historical revenue data — it works on day 1 |

External tools observe from outside. Cairo runs *inside* your project.

---

## License

MIT. See [LICENSE](./LICENSE).

---

## Built by Klaim

Cairo is open-sourced by [Klaim](https://getklaim.com), a pricing experimentation platform for AI products.

If your Cairo report flags issues across Signals 1, 2, 3, and 5 and you'd rather buy than build — Klaim covers those signals as a product. Cairo will keep working without it; the 90% of value is the self-diagnosis itself.

Issues, ideas, PRs welcome.
