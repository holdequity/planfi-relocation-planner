---
name: relocation-planner
version: 1.0.0
description: Compare retirement relocation / state-tax arbitrage by orchestrating the public planfi MCP — how much a move (e.g. CA→TX/FL) saves in state income tax, property tax, estate tax, and cost of living, and whether it's worth it.
---

# relocation-planner

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All math + financial logic live server-side. This skill only gathers inputs and calls the tools —
it does **not** compute anything locally, carries no business logic, math, or defaults, and is
read-only (it never changes the user's data). The server is the source of truth.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__generate_financial_plan`):
`analyze_relocation`, plus optional `generate_financial_plan` (to mint a `plan_id` for chaining + a `share_url`).
Use whichever name your environment exposes (bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp — no auth.)

## Step 1 — Gather inputs (prefer a `plan_id`)

Ask only for what the user's question needs. Every input has a sensible server default, so the
tools run cold — but the cleanest path is to mint a **`plan_id`** first and chain off it:

- **Optional but recommended:** call **`generate_financial_plan`** with whatever household model you
  have (earners' age + salary, stocks `current_value` + `monthly_contribution`, plus any account
  balances / spend the user volunteered). **CAPTURE the returned `plan_id`** and the `share_url`.
  Pass `{ plan_id }` to `analyze_relocation` (which takes a `plan_id` plus inline `overrides`) to
  derive figures from the saved plan.
- **Or run cold:** pass `analyze_relocation` its raw inputs directly. Only `from_state` and
  `to_state` are required; everything else is defaulted server-side.

> **Engine facts to bake in:** all dollars are **today's (real) dollars**; all decimals are
> **fractions** (24% → `0.24`, 5% → `0.05`); tax brackets/limits are approximate **~2026** values
> (noted in each tool's `disclosures`).

## Step 2 — Route by intent

- **"Should I move from <state A> to <state B> in retirement?" / "How much would relocating to TX/FL/NV save me?" / "Is leaving California worth it?"** → **`analyze_relocation`**
  - REQUIRED: `from_state`, `to_state` (2-letter codes).
  - Recommended (each defaults server-side if omitted, reported in `assumed_defaults[]`): `annual_retirement_income` (pension / IRA / 401k withdrawals / RMD / interest — the state-taxable ordinary income; default 0), `social_security_income` (default 0), `annual_capital_gains` (annual realized long-term gains; default 0), `annual_spend` (living spend at cost-of-living index 100 — drives the COL delta; default 0), `real_estate_value` (home value for property-tax comparison; default 0), `current_age` (default 50), `life_expectancy` (default 90 — current_age→life_expectancy sets years in retirement), `filing_status` (`single` | `married_joint`, default `single`).
  - Other optional fields: `liquid_assets`, `mortgage_principal` (informational; default 0), `estimated_growth_rate` (real rate, default 0.05), `tax_year` (default 2026), `plan_id` (+ inline `overrides`) to derive figures from a saved plan.
  - Returns: `annual_after_tax_delta` and `lifetime_after_tax_delta` (**positive = the move saves**), `estate_tax_delta`, `total_lifetime_advantage`, a `headline_recommendation` + `recommendation_reason`, and `from_breakdown` / `to_breakdown` line items (state income tax, property tax, COL).

> **Feed it into the forecast (not just a comparison):** `generate_financial_plan` now accepts a `relocation` object directly as a plan input — modeled as retire-and-relocate, the annual state-tax + cost-of-living savings reduce the portfolio-funded spend from the move age, so the move shows up in net worth, FIRE %, and Monte-Carlo backtesting. Use `analyze_relocation` for the headline from→to delta; pass `relocation` into the plan to see its effect on the whole household forecast.

Example (FICTIONAL — illustrate the call shape):

```
analyze_relocation({
  from_state: "CA", to_state: "TX",
  annual_retirement_income: 90000, social_security_income: 40000,
  annual_capital_gains: 0, annual_spend: 70000, real_estate_value: 600000,
  current_age: 60, life_expectancy: 90, filing_status: "married_joint"
})
// → annual_after_tax_delta ≈ +$18,000/yr (move saves), lifetime ≈ +$540k, + estate-tax delta
```

## Step 3 — Surface the result

- **Lead with the headline** dollar figure / decision the tool returns.
- **Read back `assumed_defaults[]`** — `analyze_relocation` returns a structured
  `assumed_defaults[]` array of `{ field, assumed_value, note }` for every input you omitted (e.g.
  `annual_retirement_income`, `social_security_income`, `current_age`, `filing_status`). Surface
  these so the user can correct any silent default, then re-call with the corrected values.
- **`disclosures.key_assumptions`** — a SEPARATE array of static prose strings (engine conventions,
  e.g. which states use bracket tables). Optional context; the per-call assumptions live in
  `assumed_defaults[]` above.
- **`disclosures.not_advice`** — a boolean (always `true`); relay that this is a planning estimate,
  not financial advice.
- **`next_actions[]`** — each `{ tool, why, prefilled_args }`. For `analyze_relocation` these point to
  `analyze_advanced_taxes` and `analyze_estate_exposure` (and only when a `plan_id` was passed are the
  args prefilled with it). Follow these server-suggested chains rather than guessing the next call.
- **`share_url`** — `analyze_relocation` emits a `share_url` when you pass a `plan_id` (and the plan
  resolves with the household). `generate_financial_plan` always returns one. Offer it so the user
  can open the full interactive plan on planfi.app.

## Recommended call sequence (typical session)

1. (optional) `generate_financial_plan` → **capture `plan_id`** + `share_url`.
2. Route by intent to the relevant specialist tool(s) above, passing `{ plan_id }`.
3. Surface headline + `assumed_defaults[]` + `next_actions` + `share_url`.
4. Follow `next_actions[]` chains on demand.

## Notes

- All decimals are fractions; all dollars are today's (real) dollars.
- Reuse the `plan_id` across the session — don't re-send the full model each call.
- `analyze_relocation` emits a structured `assumed_defaults[]` (read these back to the user) and a
  `share_url` when called with a `plan_id`. Run it cold (just `from_state` + `to_state`) and it still
  returns `assumed_defaults[]` for everything it defaulted.
- Not financial advice. Planning estimates only (approximate ~2026 brackets/limits where tax applies).
