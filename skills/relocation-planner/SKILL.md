---
name: relocation-planner
version: 1.2.0
description: Compare retirement relocation / state-tax arbitrage by orchestrating the public planfi MCP — how much a move (e.g. CA→TX/FL) saves in state income tax, property tax, estate tax, and cost of living — plus model the messy transition year you actually move in (part-year-resident apportionment, source-based wage/equity sourcing, the resident-state credit for taxes paid to the other state, reciprocity, and the 183-day residency test).
---

# relocation-planner

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All math + financial logic live server-side. This skill only gathers inputs and calls the tools —
it does **not** compute anything locally, carries no business logic, math, or defaults, and is
read-only (it never changes the user's data). The server is the source of truth.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__generate_financial_plan`):
`analyze_relocation`, `analyze_multi_state_part_year_tax` (the transition-year / part-year split), plus optional `generate_financial_plan` (to mint a `plan_id` for chaining + a `share_url`).
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
  - State income tax is computed by the shared engine (`state-tax.ts`): progressive bracket tables for all 50 states + DC, with first-class single and married-filing-jointly (MFJ) brackets — so `filing_status` branches every state (no-income-tax states report $0). Brackets are server-side; there is no user-supplied flat-rate field.

> **Shared bracket engine — `n/a (shared engine: state-tax.ts)`.** The full 50-state + DC
> single/MFJ progressive bracket tables (with per-state surtaxes) live in one shared engine module
> (`state-tax.ts`), the same one the broader **`tax-optimizer`** skill uses — no CA/NY/MA-only
> special-casing, no per-tool flat-rate fallback. For Roth-conversion rooms, gain-harvesting
> headroom, or multi-year tax timing alongside the move, use the **`tax-optimizer`** skill (it wraps
> the same `analyze_relocation` tool plus the full tax toolset).

> **Feed it into the forecast (not just a comparison):** `generate_financial_plan` now accepts a `relocation` object directly as a plan input — modeled as retire-and-relocate, the annual state-tax + cost-of-living savings reduce the portfolio-funded spend from the move age, so the move shows up in net worth, FIRE %, and Monte-Carlo backtesting. Use `analyze_relocation` for the headline from→to delta; pass `relocation` into the plan to see its effect on the whole household forecast.

### intent → analyze_multi_state_part_year_tax

**"I moved from CA to TX mid-year, how do I file two state returns?" / "part-year resident tax" / "how do I split my income between two states the year I moved?" / "my RSUs vested after I moved states — which state taxes them?" / "do I get credit for taxes paid to my old state?" / "183-day residency test" / "am I a statutory resident?" / "reciprocity between PA and NJ" / "I'm a remote worker taxed in two states this year"**

> **Always CALL `analyze_multi_state_part_year_tax` for these — do not answer from general knowledge / quote rules of thumb from memory. When the user gives the move date and income, run it and lead with its real output.** `analyze_relocation` deliberately models a *full* year in each state and excludes the messy *transition* year; this tool IS that transition year — day-count apportionment, source-based sourcing of wages/equity/gains, the resident-state credit for taxes paid to the other state, reciprocity, and the 183-day/domicile statutory-residency test. Whenever the user is describing the *year of the move itself* (two part-year returns, income straddling the line, equity vesting across the move, claiming the other-state credit), this is the tool — never hand-compute the apportionment or the credit.

- **REQUIRED:** `old_state`, `new_state` (2-letter codes), `days_in_old_state`, `days_in_new_state` (residency days — these split the year and have no server default; the tool needs the move date).
- **Server-defaulted and reported in `assumed_defaults[]` if omitted:** `filing_status` (`single` | `married_joint`, default `single`), `wages` (default 0), `total_workdays` (default 260), `equity_vest_value` / `capital_gains` / `deferred_comp` / `other_ordinary_income` (default 0), `capital_gains_realized_while_old_resident` (default false → new-resident sourcing), `deferred_comp_source_state` (`old` | `new`, default `old`), `reciprocity` (default false), `domicile_in_old_state` (default false), `tax_year` (default 2026).
- **Source-based sourcing inputs (optional; fall back to a residency-day split when omitted):** `wages_workdays_in_old_state` (workdays in the old state — sources wages to where work was performed), `equity_workdays_old_state_grant_to_vest` + `equity_total_workdays_grant_to_vest` (allocate an RSU grant→vest window straddling the move), `old_state_flat_rate` / `new_state_flat_rate` (flat-rate fallback for a tableless state).
- **Returns:** per-state `residencyStatus` (part-year-resident | nonresident | statutory-resident) with `residencyDeterminations[]` (basis: `183-day` | `domicile` | `reciprocity` | `part-year`), income apportioned to each state, `oldStateTax` / `newStateTax`, `doublyTaxedIncome`, `residentStateCredit` (the credit for taxes paid to the other state, limited to the resident-state tax on the doubly-taxed income), `combinedStateTax`, `naiveFullYearSingleStateTax`, and `netVsNaive` (positive ⇒ the part-year split beats a naive single-state assumption).
- **Shared engine — `n/a (shared engine: state-tax.ts)`.** Every per-state full-year bracket computation reuses the same 50-state + DC single/MFJ progressive `state-tax.ts` engine that `analyze_relocation` and the **`tax-optimizer`** skill use — no per-tool flat-rate special-casing, no re-declared brackets. For the underlying equity-vest sourcing mechanics (grant→vest workday allocation, ISO/RSU treatment) cross-reference the **`equity-comp-planner`** skill; for Roth-conversion rooms or multi-year tax timing around the move, the **`tax-optimizer`** skill.

Example (FICTIONAL — illustrate the call shape):

```
analyze_multi_state_part_year_tax({
  old_state: "NY", new_state: "CA",
  days_in_old_state: 120, days_in_new_state: 245,
  wages: 160000, wages_workdays_in_old_state: 120, total_workdays: 240,
  equity_vest_value: 100000, equity_workdays_old_state_grant_to_vest: 300,
  equity_total_workdays_grant_to_vest: 600, filing_status: "single"
})
// → NY part-year-resident + CA year-end resident; ~80k wages NY-sourced;
//   residentStateCredit = min(NY tax on the doubly-taxed slice, CA marginal on it);
//   combinedStateTax + netVsNaive vs assuming a full year all in CA
```

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
