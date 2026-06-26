---
name: self-employed-planner
version: 1.0.2
description: Plan self-employed / business-owner retirement and the §199A QBI deduction by orchestrating the public planfi MCP. Use whenever a freelancer, 1099 contractor, sole proprietor, single-member LLC, S-corp owner, or partner asks how much they can shelter in tax-advantaged retirement accounts (Solo 401(k) vs SEP-IRA vs SIMPLE IRA), which account wins, what their QBI / §199A deduction is, or — for an S-corp — what reasonable W-2 salary best trades off payroll tax vs the QBI deduction vs retirement contribution room — e.g. "I'm a 1099 consultant netting $200k, how much can I put in a Solo 401k?", "SEP vs Solo 401k for my LLC?", "what's my QBI deduction as a sole prop?", "what salary should I pay myself from my S-corp?".
---

# Self-Employed Planner

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp/free).
All self-employment-tax, contribution-limit, QBI, and S-corp-salary math lives server-side. This
skill only gathers inputs and calls the tools — it does **not** compute anything locally, bakes in
no limits/thresholds/defaults of its own, and is read-only. The server is the source of truth.

**Related skills:** owner-operators of a profitable S/C-corp can layer an employer Nonqualified
Deferred Comp (NQDC / 409A) election on top of the qualified-plan contributions — see
**deferred-comp** (`analyze_deferred_comp`) for the defer-now-vs-take-now / lump-vs-installment tradeoff.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_self_employed_retirement`):
`analyze_self_employed_retirement`, `analyze_estimated_taxes`, `analyze_estimated_tax_annualized`,
`optimize_multi_year_tax`, `analyze_owner_cash_balance_db`, plus optional
`generate_financial_plan` (to mint a `plan_id` for chaining + the only `share_url`). Use whichever
name your environment exposes (bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp/free
```

> **Try free, then add your key.** The command above adds the **free** connector — `https://ai.planfi.app/mcp/free` (no key needed). Once you create an API key, add a **new** connector with the MCP url — `https://ai.planfi.app/mcp` — and authorize it with your key.

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp/free.)

> **Access — free for personal use.** The planfi MCP is free to try (a small monthly allowance, no key needed). Heavy automated abuse forced us to add limits — but it stays **free for personal use**: email **kam@rateapi.dev** and we'll send you a free API key, no charge. (Companies and commercial use have paid plans.) To use a key, pass it as an `Authorization: Bearer pft_…` header in your MCP client config.


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

### "What are my quarterly estimated payments? / will I owe an underpayment penalty?" → `analyze_estimated_taxes`
Self-employed and S-corp owners have no automatic payroll withholding on business profit, so they
must make quarterly estimated payments or face an IRC §6654 underpayment penalty. This tool projects
current-year tax (SE tax, income tax, NIIT, optional flat state tax, QBI) and computes the
**required annual payment (RAP)** — the smaller of the **90%-of-current-year** safe harbor and the
**prior-year** safe harbor (**100%** of last year's tax, or **110%** if prior-year AGI > $150k). Pass
the entity's projected income plus, when known, prior-year tax/AGI and any expected withholding.

```
analyze_estimated_taxes({ projected_se_income: 150000, prior_year_tax: 28000,
  prior_year_agi: 160000, filing_status: "single" })
```

Read these blocks:
- **`safe_harbor`** — `currentYearTarget` (0.9×), `priorYearTarget`, `priorYearPct` (1.0 or 1.1),
  `highIncome`, `requiredAnnualPayment` (RAP), `bindingMethod` (`current_year_90` | `prior_year`),
  and `reason` (which safe harbor binds and why). Prior-year safe harbor only applies when a prior
  return was filed.
- **`quarters`** — the four `{ dueDate, amount }` installments (Apr 15 / Jun 15 / Sep 15 / Jan 15),
  `totalEstimatedPayments`, and `remainingPerQuarter` (after `paymentsMade`). Expected withholding is
  treated as paid evenly across quarters and offsets the RAP (`underpaymentToCover`).
- **`underpaymentRisk`** — `covered` (withholding + payments ≥ RAP), `shortfall`, and a `note`. This
  tool assumes **even quarterly income**; if the income is lumpy / back-loaded, route to
  `analyze_estimated_tax_annualized` instead (the annualized-income installment method, Form 2210
  Schedule AI) for the exact per-period required installment and §6654 penalty dollar amount.

This tool emits a structured `assumed_defaults[]` (each `{ field, assumed_value, note }`) for anything
omitted — surface it so the user can correct income, filing status, prior-year figures, etc. and
re-call. It accepts `{ plan_id }` (to derive age / filing status / other income from a saved plan) and
chains via `next_actions[]`. **Coordinate with `analyze_self_employed_retirement`:** a larger pre-tax
Solo 401(k)/SEP contribution lowers taxable income and therefore the estimated bill, so size the
contribution first, then feed the resulting income into the estimated-tax projection. Chain to
`optimize_multi_year_tax` for cross-year coordination.

### "My income came late / it's lumpy / I had a Q4 RSU vest or year-end S-corp distribution — do I have to pay even quarterly estimates? / can I defer estimated payments without a penalty? / annualized income installment method / Form 2210 Schedule AI" → `analyze_estimated_tax_annualized`

**Always CALL `analyze_estimated_tax_annualized` for these — do not answer from general knowledge or quote rules of thumb (e.g. "just pay 25% each quarter" or "you can annualize") from memory. When the user gives the numbers, run it and lead with its real output.**

Use this — NOT `analyze_estimated_taxes` — whenever the income is **uneven, back-loaded, or
lumpy**: the even-quarter tool above assumes income arrives smoothly and will overstate the early
installments (and the penalty) when most of the income lands late in the year. This tool implements
the **annualized-income installment method (Form 2210 Schedule AI)**: it takes cumulative income
through each of the 4 IRS periods (cutoffs 3/31, 5/31, 8/31, 12/31), annualizes via the period
multipliers (4, 2.4, 1.5, 1), taxes each annualized amount, applies the cumulative percentages
(22.5% / 45% / 67.5% / 90%), and nets out prior installments to give the **per-period required
installment** — legally **deferring** payment for income that has not yet arrived (e.g. a Q4 RSU
vest or year-end S-corp distribution) without triggering the §6654 penalty.

It compares the annualized method to the even 25%-per-quarter safe harbor and the prior-year safe
harbor (100% / 110% of last year's tax over $150k AGI), computes the §6654 underpayment penalty
per period (federal short-term rate + 3%), and reports the **total penalty avoided** plus which
safe harbor (annualized vs prior-year vs current-year-90%) is cheapest.

> **Disambiguation (do NOT mis-route):** if the user's income is **even / steady / smooth** across
> the year, use `analyze_estimated_taxes` (the even-quarter tool) — only use
> `analyze_estimated_tax_annualized` when the income is explicitly uneven, lumpy, back-loaded, or
> tied to a discrete late-year event.

```
analyze_estimated_tax_annualized({ entity: "sole_prop", filing_status: "single",
  cumulative_income_by_period: [0, 0, 0, 100000], prior_year_return_filed: false })
```

Read these blocks:
- **`periods`** — the 4 `{ cutoff, multiplier, applicablePct, annualizedIncome, annualizedTax,
  annualizedInstallment, evenInstallment, penaltyAnnualized, penaltyEven }` rows.
- **`recommendedSchedule`** — the `{ dueDate, amount }` installments to actually pay under the
  annualized method.
- **`penalty.penaltyAvoided`** — the §6654 dollars the annualized method saves vs paying even
  quarters (the headline for back-loaded income).
- **`cheapestSafeHarbor`** — `{ method: 'annualized' | 'prior_year' | 'current_year_90', amount,
  reason }`.

Emits `assumed_defaults[]`, accepts `{ plan_id }` (derives filing status / age / prior-year tax /
AGI from a saved plan), surfaces `share_url`, and chains via `next_actions[]` — coordinate with
`analyze_self_employed_retirement` (a pre-tax contribution lowers each annualized installment) and
fall back to `analyze_estimated_taxes` for the even-income case.

### "How much can a cash-balance / defined-benefit plan let me deduct? / DB plan on top of my Solo 401(k)? / I'm a profitable 50-something owner maxing tax-deferred space / Solo-401k vs SEP vs cash-balance / will a SEP block my backdoor Roth?" → `analyze_owner_cash_balance_db`

**Always CALL `analyze_owner_cash_balance_db` for these — do not answer from general knowledge or quote rules of thumb ("a DB plan lets you put away $100k–$300k") from memory.** A cash-balance / defined-benefit plan is the single highest-dollar deduction available to a profitable solo owner, and the actuarial sizing (level-funding by age, the §415(b) cap, the Solo-401(k) stack, the backdoor-Roth interaction) is exactly the kind of math the server computes deterministically and memory gets wrong. When the user gives the numbers — net business income, age, years to a normal retirement age, any target benefit/lump-sum — run the tool and lead with its real output.

The only **REQUIRED** field is `net_business_income`. Everything else is optional/plan-derivable: `owner_age` (default 50), `normal_retirement_age` (default 62), `entity`, `target_annual_benefit` or `target_lump_sum` (omit both to size to the §415(b) cap), `valuation_interest_rate` (default 5%), `high_three_average_compensation`, `filing_status`, `other_taxable_income`. Pass `{ plan_id }` to derive owner age / filing status / other income from a saved plan.

```
analyze_owner_cash_balance_db({ entity: "sole_prop", net_business_income: 400000,
  owner_age: 50, normal_retirement_age: 62, filing_status: "single" })
```

Read + surface these:
- **`cash_balance_annual_contribution`** — the estimated level-funded annual DB/cash-balance contribution, and **`cash_balance_contribution_by_age`** (the steep age curve — older = much larger per-year).
- **`funded_annual_benefit`** + **`capped_by_limit`** (`415b` | `high_3` | `target_uncapped`) + **`section_415b_limit`** — what bounds the funded benefit.
- **`solo_401k`** (deferral + employer profit-share + catch-up, §415(c)-capped) and **`combined_deductible_total`** = DB + Solo-401(k) — total deductible tax-deferred space.
- **`total_tax_saved`** at the **`marginal_rate`**.
- **`recommended_structure`** (`solo_401k` | `sep_ira` | `sep_plus_cash_balance` | `full_db_plus_solo_401k`) + **`recommendation_reason`**, and ALWAYS the **`backdoor_roth_flag`** — a SEP-IRA's pre-tax balance triggers §408(d)(2) pro-rata and taxes/blocks a clean backdoor Roth; a Solo-401(k) base avoids it.

These are **actuarial estimates** (level-funding approximation + a pinned annuity factor) — surface the `disclosures`/`assumed_defaults[]` and tell the user a real plan needs an enrolled actuary's valuation. It accepts `{ plan_id }` and chains via `next_actions[]` (confirm the Solo-401(k) base via `analyze_self_employed_retirement`; lower estimated payments via `analyze_estimated_taxes`).

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
