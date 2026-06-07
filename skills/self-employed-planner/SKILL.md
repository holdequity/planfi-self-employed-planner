---
name: self-employed-planner
version: 1.0.0
description: Plan self-employed / business-owner retirement and the §199A QBI deduction by orchestrating the public planfi MCP. Use whenever a freelancer, 1099 contractor, sole proprietor, single-member LLC, S-corp owner, or partner asks how much they can shelter in tax-advantaged retirement accounts (Solo 401(k) vs SEP-IRA vs SIMPLE IRA), which account wins, what their QBI / §199A deduction is, or — for an S-corp — what reasonable W-2 salary best trades off payroll tax vs the QBI deduction vs retirement contribution room — e.g. "I'm a 1099 consultant netting $200k, how much can I put in a Solo 401k?", "SEP vs Solo 401k for my LLC?", "what's my QBI deduction as a sole prop?", "what salary should I pay myself from my S-corp?".
---

# Self-Employed Planner

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All self-employment-tax, contribution-limit, QBI, and S-corp-salary math lives server-side. This
skill only gathers inputs and calls the tools — it does **not** compute anything locally, bakes in
no limits/thresholds/defaults of its own, and is read-only. The server is the source of truth.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_self_employed_retirement`):
`analyze_self_employed_retirement`, `optimize_multi_year_tax`, plus optional
`generate_financial_plan` (to mint a `plan_id` for chaining + the only `share_url`). Use whichever
name your environment exposes (bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp — no auth.)

## Step 1 — (Optional) build a plan first to chain context + get a share link

> **Feed it into the forecast (not just plan_id chaining):** `generate_financial_plan` now accepts `self_employed` directly as a plan input, so it flows into net worth, FIRE %, and Monte-Carlo backtesting — the recommended tax-advantaged contribution flows in as pre-tax savings. Use the standalone analyze tool below for a focused what-if; pass `self_employed` into the plan to see its effect on the whole household forecast.

If the user has (or wants) a full household model, call **`generate_financial_plan`** once and
**capture the returned `plan_id`**. `analyze_self_employed_retirement` accepts `{ plan_id }`, which
lets the server derive the owner's **age**, **filing status**, and **other taxable income** (e.g. a
spouse's W-2) from the saved plan's earners instead of you re-sending them. `generate_financial_plan`
also returns a **`share_url`** (planfi.app). `analyze_self_employed_retirement` never emits a
`share_url`; `optimize_multi_year_tax` emits one **only** when you pass a `{ plan_id }` that resolves
to a saved plan. Minting a `plan_id` here is the reliable way to give the user a sharable plan.

This step is optional: the tool runs cold from raw inputs too. Prefer the plan path when the session
already has a model or the user wants a sharable artifact.

> **Engine facts to bake in:** all dollars are **today's (real) dollars**; all decimals are
> **fractions** (24% → `0.24`); contribution limits, the SS wage base, and the §199A QBI thresholds
> are approximate **~2026** values (reported back in each call's `disclosures`). Override `tax_year`
> if the user needs a different year.

## Step 2 — Route by intent

The single specialist tool here is `analyze_self_employed_retirement`. The only **REQUIRED** field is
`entity_type` (`sole_prop` | `single_member_llc` | `s_corp` | `partnership`). Every other field —
including `net_business_income` (defaults to 200000) — is optional with a server default, but you
should always pass the entity's income figure so the result reflects the user's actual numbers. Pass
`{ plan_id }` to resolve age / filing status / other income; otherwise pass them inline. Every
omitted default is reported back (see Step 3).

### "How much can I shelter? / which account — Solo 401(k), SEP, or SIMPLE?" → `analyze_self_employed_retirement`
For `sole_prop` / `single_member_llc` / `partnership`, pass `net_business_income` (the owner's net
SE earnings). Headline fields: `contributions_by_account` (per-account max) and `recommended_account`
+ `recommended_contribution` + `recommendation_reason`. Optional context: `age` (age-50 catch-up),
`employee_count`, `spouse_on_payroll`, `existing_elective_deferrals` (402(g) coordination across jobs).

```
analyze_self_employed_retirement({ entity_type: "sole_prop", net_business_income: 200000,
  age: 45, filing_status: "single", is_sstb: false })
```

### "What's my QBI / §199A deduction?" → `analyze_self_employed_retirement`
Same tool — read the `qbi_deduction` block: `amount`, `qbi_base`, `deduction_before_cap`,
`w2_wage_limit`, `threshold_status` (`below` | `phase_in` | `above`), `sstb_phaseout_applied`.
Pass `is_sstb: true` for a specified service trade or business (law, health, consulting, finance,
etc.), `other_taxable_income` for the taxable-income threshold, and `ubia` (unadjusted basis of
qualified property) for the 2.5%-UBIA alternative wage limit. `net_federal_tax_delta` shows how much
contributing + QBI lowers federal tax.

### "What salary should I pay myself from my S-corp?" → `analyze_self_employed_retirement`
Pass `entity_type: "s_corp"`, `net_business_income` (business PROFIT before the owner W-2), and
optionally `s_corp_w2_wages` (if the salary is fixed by the user) + `s_corp_distributions`
(informational). Omit `s_corp_w2_wages` to let the server solve a reasonable salary. Read the
`s_corp_salary_recommendation` block: `salary`, `distribution`, `payroll_tax`, `estimated_total_tax`,
and its `note` (reasonable compensation is a facts-and-circumstances IRS rule — relay it as an
estimate). The S-corp employer profit-share is 25% of W-2, so salary also drives retirement room.

```
analyze_self_employed_retirement({ entity_type: "s_corp", net_business_income: 150000,
  s_corp_w2_wages: 90000, s_corp_distributions: 60000, is_sstb: true, filing_status: "single" })
```

### "Coordinate this across years / with ISO exercises + Roth conversions" → `optimize_multi_year_tax`
After sizing the contribution, chain to `optimize_multi_year_tax` (AMT-crossover / NIIT-threshold /
IRMAA-tier aware year-by-year plan) — pass `{ plan_id }` when you have one. REQUIRED:
`baseline_ordinary_income`. This is the typical `next_actions[]` chain.

## Step 3 — Surface the result

- **Lead with the headline:** the **recommended account + annual contribution** it shelters (and how
  the alternatives compare), then the **QBI deduction** and the **net federal tax delta**. For an
  S-corp, lead with the recommended salary + distribution split. The `summary` block has ready-made
  one-liners.
- **Read back assumptions:** this tool **does** emit a structured `assumed_defaults[]` (each
  `{ field, assumed_value, note }`) for anything you omitted — surface those so the user can correct
  any (income, age, filing status, is_sstb, ubia, …) and re-call with overrides. Also read
  `disclosures.key_assumptions` (SE-tax method, ~2026 limits/thresholds, the S-corp
  facts-and-circumstances note).
- **`disclosures.not_advice`** — relay that this is a planning estimate, not tax advice; for S-corps
  emphasize that "reasonable compensation" is an IRS facts-and-circumstances rule.
- **`next_actions[]`** — each `{ tool, why, prefilled_args }` (carrying `{ plan_id }` when
  available). Follow these server-suggested chains (often `optimize_multi_year_tax` or
  `generate_financial_plan`) rather than guessing the next call.
- **`share_url`** — `generate_financial_plan` always returns one; `optimize_multi_year_tax` returns
  one only when called with a plan-resolving `{ plan_id }`. `analyze_self_employed_retirement` does
  not emit one. If you minted a `plan_id`, offer that plan's `share_url` so the user can open the full
  interactive plan on planfi.app.

## Recommended call sequence (typical session)

1. (optional) `generate_financial_plan` → **capture `plan_id`** + `share_url`.
2. `analyze_self_employed_retirement` with `entity_type` + income (+ `{ plan_id }`).
3. Surface recommended account/contribution + QBI + tax delta; read back `assumed_defaults[]` and
   `disclosures.key_assumptions`.
4. Follow `next_actions[]` (commonly `optimize_multi_year_tax`); offer the plan's `share_url`.

## Fictional examples

**1.** *"I'm a 45-year-old 1099 consultant, sole prop, netting $200k — how much can I shelter and in
what?"* → `analyze_self_employed_retirement({ entity_type: "sole_prop", net_business_income: 200000,
age: 45, filing_status: "single", is_sstb: false })`. Lead with `recommended_account` (likely
solo_401k) + `recommended_contribution`, compare SEP/SIMPLE, then the `qbi_deduction.amount` and
`net_federal_tax_delta`. Read back `assumed_defaults[]`.

**2.** *"My S-corp (consulting) profits $150k; I'm paying myself $90k W-2 with $60k distributions —
is that the right split?"* → `analyze_self_employed_retirement({ entity_type: "s_corp",
net_business_income: 150000, s_corp_w2_wages: 90000, s_corp_distributions: 60000, is_sstb: true,
filing_status: "single" })`. Lead with `s_corp_salary_recommendation` (salary/distribution/payroll
tax), flag that consulting is an SSTB (so `sstb_phaseout_applied` may zero the QBI above the
threshold), and relay the facts-and-circumstances note.

*(Both examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; contribution limits, the SS
  wage base, and §199A QBI thresholds are ~2026 (override `tax_year` as needed).
- Pass `{ plan_id }` to derive age / filing status / other taxable income from a saved model; any
  field you also pass is an override.
- This tool emits a structured `assumed_defaults[]` — surface it. The S-corp salary recommendation is
  a labeled **estimate** (reasonable compensation is a facts-and-circumstances IRS rule).
- Not financial or tax advice. Planning estimates only (approximate ~2026 brackets/limits).
