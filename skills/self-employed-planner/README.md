# self-employed-planner (Claude Agent Skill)

Plan self-employed / business-owner retirement (Solo 401k vs SEP vs SIMPLE) and the §199A QBI deduction via the planfi MCP

It's a **thin orchestration layer** over the public **planfi MCP** (`https://ai.planfi.app/mcp`,
public, no auth) — all the math and financial logic live server-side. The skill itself bundles no
engine; it just gathers inputs and calls the tools.

## What it answers

For sole proprietors, single-member LLCs, S-corps, and partnerships (founders / freelancers / 1099
contractors):

- **How much can I shelter** in tax-advantaged retirement accounts as a self-employed person?
- **Which account wins** — Solo 401(k) vs SEP-IRA vs SIMPLE IRA?
- **What's my §199A QBI deduction** (taxable-income thresholds, W-2-wage / 2.5%-UBIA cap, SSTB phaseout)?
- **(S-corp)** What **reasonable W-2 salary** optimizes payroll tax ↔ QBI ↔ retirement contribution room?
- **Quarterly estimated taxes & safe-harbor** — what to pay each quarter and the 90%/110% safe-harbor thresholds to avoid an underpayment penalty (`analyze_estimated_taxes`)?
- **Lumpy / back-loaded income** (a Q4 RSU vest, a year-end S-corp distribution) — can I **annualize** my estimated payments (Form 2210 Schedule AI) to defer them legally without a §6654 penalty, and how much penalty does that avoid vs paying even quarters? (`analyze_estimated_tax_annualized`)
- **Cash-balance / defined-benefit plan** — for a profitable 45–60yo owner, how much can a DB/cash-balance plan stacked on the Solo 401(k) let me deduct (actuarial annual contribution, §415(b) cap, combined deductible + tax saved), and does a SEP-IRA block my backdoor Roth? (`analyze_owner_cash_balance_db`)

It wraps `analyze_self_employed_retirement` (the headline tool) and chains `optimize_multi_year_tax`
for cross-year coordination; optionally mint a `plan_id` via `generate_financial_plan` for a
sharable plan.

### Fictional examples

```
# Sole-prop freelancer, "how much / which account?"
analyze_self_employed_retirement({ entity_type: "sole_prop", net_business_income: 200000,
  age: 45, filing_status: "single", is_sstb: false })

# S-corp consulting owner, "what salary should I pay myself?"
analyze_self_employed_retirement({ entity_type: "s_corp", net_business_income: 150000,
  s_corp_w2_wages: 90000, s_corp_distributions: 60000, is_sstb: true, filing_status: "single" })

# Profitable 50yo owner, "how much can a cash-balance / DB plan let me deduct?"
analyze_owner_cash_balance_db({ entity: "sole_prop", net_business_income: 400000,
  owner_age: 50, normal_retirement_age: 62, filing_status: "single" })
```

*(Fictional figures only — never reuse a real user's numbers.)*

### MCP bootstrap (one time)

If the planfi tools aren't connected yet, run:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

On **claude.ai**: Settings → Connectors → add a custom connector pointing at
`https://ai.planfi.app/mcp` (no auth). The skill also reminds you to do this if the tools are
missing when you invoke it.

## Install

### Quickest — skills.sh CLI (recommended)

```
npx skills add holdequity/planfi-self-employed-planner
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/self-employed-planner ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/self-employed-planner .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-self-employed-planner
/plugin install self-employed-planner@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r self-employed-planner.zip self-employed-planner`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.

## Notes & honest caveats

- All decimals are fractions (24% → `0.24`); all figures are today's (real, inflation-adjusted)
  dollars; tax brackets/limits are approximate ~2026 values.
- The skill surfaces every assumption the server reports — a structured `assumed_defaults[]`
  (`{ field, assumed_value, note }`) plus `disclosures.key_assumptions` — so you can correct any
  default and re-call with overrides.
- The S-corp salary recommendation is a labeled **estimate**: "reasonable compensation" is a
  facts-and-circumstances IRS rule.
- Not financial or tax advice. Planning estimates only.

See `SKILL.md` for the full instructions, exact tool params, and output format.
Source + issues: <https://github.com/holdequity/planfi-self-employed-planner>.

## License

MIT — see [LICENSE](./LICENSE).
