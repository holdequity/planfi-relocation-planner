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
  Pass `{ plan_id }` to every specialist tool below — any field you also pass becomes a shallow
  override. This is also the ONLY way to get a `share_url` (the specialist tools don't emit one).
- **Or run cold:** pass the specialist tool its required raw inputs directly. Anything omitted is
  defaulted server-side.

> **Engine facts to bake in:** all dollars are **today's (real) dollars**; all decimals are
> **fractions** (24% → `0.24`, 5% → `0.05`); tax brackets/limits are approximate **~2026** values
> (noted in each tool's `disclosures`).

## Step 2 — Route by intent

- **"Should I move from <state A> to <state B> in retirement?" / "How much would relocating to TX/FL/NV save me?" / "Is leaving California worth it?"** → **`analyze_relocation`**
  - REQUIRED: `from_state`, `to_state` (2-letter codes).
  - Recommended (each defaults server-side if omitted, surfaced in `disclosures`): `annual_ordinary_income` (pension / IRA / 401k withdrawals / RMD / interest — the state-taxable ordinary income), `annual_social_security`, `annual_long_term_gains`, `annual_baseline_spend` (living spend at a national-average cost of living — drives the COL delta), `home_value` (property tax), `projected_estate` (state estate tax), `years_in_retirement`, `filing_status`.
  - Optional `plan_id` to derive figures from a saved plan.
  - Returns: `annual_after_tax_delta` and `lifetime_after_tax_delta` (**positive = the move saves**), `estate_tax_delta`, `total_lifetime_advantage`, a `headline_recommendation` + `recommendation_reason`, and `from_breakdown` / `to_breakdown` line items (state income tax, property tax, COL).

> **Feed it into the forecast (not just a comparison):** `generate_financial_plan` now accepts a `relocation` object directly as a plan input — modeled as retire-and-relocate, the annual state-tax + cost-of-living savings reduce the portfolio-funded spend from the move age, so the move shows up in net worth, FIRE %, and Monte-Carlo backtesting. Use `analyze_relocation` for the headline from→to delta; pass `relocation` into the plan to see its effect on the whole household forecast.

Example (FICTIONAL — illustrate the call shape):

```
analyze_relocation({
  from_state: "CA", to_state: "TX",
  annual_ordinary_income: 90000, annual_social_security: 40000,
  annual_baseline_spend: 70000, home_value: 600000,
  years_in_retirement: 30, filing_status: "married_joint"
})
// → annual_after_tax_delta ≈ +$18,000/yr (move saves), lifetime ≈ +$540k, + estate-tax delta
```

## Step 3 — Surface the result

- **Lead with the headline** dollar figure / decision the tool returns.
- **Read back `disclosures.key_assumptions`** — these specialist tools apply silent defaults and
  report what they assumed only as **prose in `disclosures.key_assumptions`** (they do NOT emit a
  structured `assumed_defaults[]`). Surface those assumptions so the user can correct any that are
  wrong, and re-call with overrides.
- **`disclosures.not_advice`** — relay that this is a planning estimate, not financial advice.
- **`next_actions[]`** — each `{ tool, why, prefilled_args:{ plan_id } }`. Follow these
  server-suggested chains rather than guessing the next call.
- **`share_url`** — only `generate_financial_plan` returns one. If you minted a `plan_id`, offer that
  plan's `share_url` so the user can open the full interactive plan on planfi.app.

## Recommended call sequence (typical session)

1. (optional) `generate_financial_plan` → **capture `plan_id`** + `share_url`.
2. Route by intent to the relevant specialist tool(s) above, passing `{ plan_id }`.
3. Surface headline + `disclosures.key_assumptions` + `next_actions` + `share_url`.
4. Follow `next_actions[]` chains on demand.

## Notes

- All decimals are fractions; all dollars are today's (real) dollars.
- Reuse the `plan_id` across the session — don't re-send the full model each call.
- These specialist tools do **not** emit `assumed_defaults[]` or a `share_url`; read assumptions from
  `disclosures.key_assumptions` and chain to `generate_financial_plan` for a `share_url`.
- Not financial advice. Planning estimates only (approximate ~2026 brackets/limits where tax applies).
