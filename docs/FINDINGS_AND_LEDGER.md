# Findings and review ledger: a reusable template

Evidence Checker writes **findings** (machine output) and reads a **review ledger**
(human decisions). The two are kept separate: the checker never writes to the
ledger. Teams that run fairness audits can reuse both formats with any tool.

## Check result (`findings.json`)

The command line writes one JSON object per bundle:

| Field | Meaning |
|---|---|
| `snapshot_id` | Hash of the bundle's file manifest, model, cohort and policy |
| `policy_version` | Version of the policy that produced the findings (e.g. `policy-v1`) |
| `model`, `cohort` | What was checked |
| `counts` | Findings per rule and outcome |
| `gate` | `FAIL` if any finding has an outcome listed in the policy's `fail_on` (default: `INCOMPLETE`), else `PASS` |
| `gate_note` | Always states that `PASS` is not a fairness certification or deployment approval |
| `findings` | List of findings, below |

## Finding

| Field | Meaning |
|---|---|
| `rule` | `R1` to `R6` |
| `outcome` | `INCOMPLETE` (evidence defect; fails the gate), `REVIEW` (needs a human decision; does not block) or `OBSERVED` (recorded; no decision needed) |
| `kind` | The specific condition (see the table below) |
| `message` | One-sentence description |
| `model`, `attribute`, `subgroup`, `key` | What the finding is about |
| `evidence` | The values the finding rests on |
| `refs` | Source of each value: `{"artifact": <file>, "row": {<column>: <value>}}` |
| `role` | Owner role who should act |
| `action` | Suggested action |
| `finding_id` | Stable identifier: hash of rule, model, attribute, subgroup and key. The same condition keeps its identifier across runs |
| `evidence_fp` | Evidence fingerprint: hash of the cited values and row references (value scope) or, with artifact scope, also of the whole cited files |

### Kinds per rule

| Rule | INCOMPLETE | REVIEW | OBSERVED |
|---|---|---|---|
| R1 Schema | `missing_field`, `invalid_field`, `invalid_component`, `undocumented_gap`, `dangling_ref` | | |
| R2 Support & uncertainty | `eligibility_mismatch`, `uncertainty_missing`, `interval_gate_violation` | `low_support`, `parity_not_computable` | `interval_not_approved` |
| R3 Metric coverage | `missing_metric`, `invalid_metric`, `unjustified_na`, `scope_mismatch`, `unknown_status` | | `declared_limitation` |
| R4 Cross-layer | | `divergence` (only with a pre-registered threshold) | `gap_recorded`, `not_comparable` |
| R5 Provenance | `missing_provenance`, `artifact_unavailable`, `hash_mismatch` | `open_gate` | `declared_unverified` |
| R6 Continuity | | `unresolved`, `stale` | `disposed`, `orphaned` |

## Review ledger (`review_ledger.json`)

A JSON object with a list of entries. Entries are only appended; when several
entries refer to the same finding, the last one counts.

| Field | Meaning |
|---|---|
| `finding_id` | The finding the decision is about |
| `evidence_fp` | The finding's evidence fingerprint **at the time of the decision** |
| `model`, `rule` | Copied from the finding (optional; used to match orphaned entries) |
| `assignee` | Person or role responsible |
| `decision` | One of `accepted_limitation`, `action_required`, `fixed`, `not_a_problem`, `deferred` |
| `rationale` | Why the decision was taken |
| `timestamp` | When it was taken |

`docs/review_ledger.example.json` is a complete example for one real finding of
the evaluated audit. Pass it with `--ledger`.

## How R6 reads the ledger

On each run, R6 compares every finding that needs a decision (REVIEW or
INCOMPLETE) with the ledger:

| Situation | R6 finding |
|---|---|
| No valid entry (missing, unknown decision or no assignee) | `unresolved` (REVIEW) |
| Entry exists, but `evidence_fp` differs from the current fingerprint | `stale` (REVIEW): the decision was made on different evidence |
| Entry exists and the fingerprint is unchanged | `disposed` (OBSERVED): the decision still holds |
| Entry refers to a finding that no longer occurs | `orphaned` (OBSERVED): confirm the problem is really gone |

`evidence_checker.ledger.ledger_entry(finding, assignee, decision, rationale, timestamp)`
creates a correctly filled entry from a finding.
