# AI-Assisted Lease Audit & IFRS 16 Analyser
> Follows on from my earlier [AI-in-Accounting-Portfolio](https://github.com/Haagen-Lim/AI-in-Accounting-Portfolio) - this project applies a more advanced, agentic approach to a single audit-specific problem.

## Objective
A working demonstration of AI-assisted audit evidence extraction: instead of manually reading every lease contract to pull out IFRS 16-relevant terms, this pipeline uses Claude (Anthropic) with **forced structured tool-use output** to extract lease data, then runs it through a deterministic Python engine for measurement and audit risk flagging.

The brief was deliberately not "summarise a contract" — it was "build something that surfaces the judgement calls an auditor would actually need to make," including the case a keyword search would miss entirely: a contract not labelled as a lease that meets IFRS 16's lease definition anyway.

## Methodology
1. **Sample contracts** — four synthetic lease/services agreements, each built to test a specific judgement area: a clean baseline, a renewal-option judgement call, a related-party lease with a bargain purchase option, and a "services agreement" that is actually an embedded lease.
2. **AI extraction** — Claude reads each contract and returns structured JSON via forced tool-use against a fixed schema, rather than free text that needs parsing. The schema includes a `payment_schedule` field — a list of rent tiers read directly off the contract's own rent table, rather than a category label the calculator has to guess a formula from.
3. **IFRS 16 measurement** — a Python engine computes the initial lease liability, right-of-use asset, and full amortisation schedule directly from the extracted payment tiers, correctly holding index-linked payments (e.g. CPI) flat at the commencement-date rate rather than forecasting future movements, per IFRS 16.27(b).
4. **Audit flags** — a separate, deterministic rules layer (not another AI call) raises related-party, judgement-required, multi-tier-schedule, and possible-embedded-lease flags with a stated rationale for human review.
5. **Consolidated summary** — a cross-contract view of liabilities, ROU assets, and audit risk profile.

## Headline finding
Contract D — framed as a "Managed Print & Document Services Agreement" — meets IFRS 16's control-based lease definition despite never using the word "lease." A review that only opened documents tagged as leases would miss it; reading every contract the same way catches it. That's the core argument for this kind of tool sitting in an audit workflow.

## A real bug, found and fixed
The first version of the calculation engine inferred the payment stream from a category label (`flat` / `+3%-per-year` / `CPI-held-flat`) — three buckets that covered the four synthetic contracts but nothing else. Testing it against a real lease pulled from a public SEC filing (Applied Optoelectronics, Inc., Exhibit 10.1, filed April 2025) surfaced the gap immediately: that lease has a 7-tier rent schedule with a partial floor-area rent abatement in the first two years, a shape that fit none of the three categories, so the engine silently defaulted to a flat rate for the entire term. The extraction step had actually read the contract correctly — the bug was entirely in the deterministic code downstream of it.

The fix: the extraction schema now asks the model to read the contract's actual rent tiers directly into a `payment_schedule` field, and the calculation engine consumes that schedule as-is instead of reconstructing one from a label. Verified with a full regression test (all four original contracts produce byte-identical payment streams and liability figures before and after the change) plus a synthetic 7-tier test case matching the real lease's structure (confirms the new engine correctly handles a schedule shape the old one couldn't).

## Running it
The notebook defaults to `DEMO_MODE = True`, using pre-computed reference extractions so it's fully readable on GitHub without an API key. Set `DEMO_MODE = False` and supply your own Anthropic API key to run live extraction against the sample contracts, or add your own PDF via the `PDF_CONTRACTS` cell further down.

## Limitations
- The discount rate is an illustrative incremental borrowing rate assumption where not stated in a contract, not a verified market rate.
- The calculation engine doesn't yet handle multi-component contracts (e.g. a single fee bundling a lease and non-lease services) or formal lease/non-lease component allocation.
- This is a portfolio demonstration of the workflow, not an audit opinion or substitute for professional judgement.

## Stack
Python · Claude API (Anthropic, structured tool-use) · pandas · matplotlib
