# LicenseCourt

An open-source license compliance court, as a GenLayer Intelligent Contract.

## The problem

A project can declare "we ship no GPL code" and nobody checks. The dependency manifest is public, the license report is public, but nobody reads them side by side — and a single reviewer's word is not proof. When a forbidden license does slip into a shipped dependency, the project finds out from a lawyer, not from its own tooling.

## How it works

An audit owner (a maintainer, a compliance team, or a downstream user) opens an audit for one project and declares three evidence sources up front:

- `manifest_url` — the dependency manifest (the material under test)
- `license_index_url` — the published per-dependency license report
- `exceptions_url` — the project's approved license exceptions (optional)

plus the policy itself: `forbidden_licenses`, a comma-separated list such as `"GPL-3.0, AGPL-3.0"`.

Anyone except the owner can submit a claim against the audit: a statement that a named dependency ships under a forbidden license, with a claimed severity (critical / high / medium / low).

When a claim is assessed, every validator fetches the manifest, the license report, and the exceptions page itself and judges the claim against that fetched material only:

1. **Grounded** — a claim is never valid on the claimant's word alone. If the manifest or the license report cannot be retrieved, or neither shows the named dependency under a forbidden license, the claim is invalid.
2. **Duplicate** — a claim matching the published exceptions page, or a prior *accepted* finding on the same audit, is a duplicate and invalid. A match against a *rejected* claim does not count: the same allegation was already rejected on its own merits, and a repeat must be judged again on its own merits.
3. **Strict verdict** — `valid` must be an actual JSON boolean (a string `"true"` or a `1` is a malformed verdict and the claim is rejected). A valid claim always carries a real severity bucket; an invalid claim always carries `"none"`. A valid verdict with no usable severity is rejected, not rewarded.

## Deterministic outcome: the violation count

Every audit starts at a violation count of 0. Each upheld claim adds exactly one. The project is in violation while the count is above zero; the count never comes from free-form text and is readable at any time via `get_violation_count`. Each verdict stores a one-sentence evidence note on-chain naming the fetched material that supports or rejects the claim, so any reviewer can audit the audit.

## Demo (see `test_flow.md`)

- Audit 0: acme-web 1.4.2, policy forbids GPL-3.0 and AGPL-3.0.
  - Claim 0 (sharp-lib is GPL-3.0, shipped) → **valid / high**, violation count 0 → 1.
  - Claim 1 (react is GPL-3.0) → **invalid**: the license report says MIT.
  - Claim 2 (build-tool is GPL-3.0) → **invalid**: already approved on the exceptions page (build-time only, not shipped).
  - Claim 4 (repeat of rejected claim 1) → **invalid** on its own grounding, not called a duplicate.
  - Claim 5 (repeat of accepted claim 0) → **invalid** as a duplicate of the accepted finding.
- Audit 1: same project, unreachable manifest URL → claim 3 **invalid** (no corroborating material).

Final state: 2 audits, 6 claims, violation count 1. 9 negative tests all rejected as expected.

## Verification

Deployed and verified in GenLayer Studio on 2026-11-09: contract [0x06a852688E326D0596E786519E15168AE16C98B4](https://explorer-studio.genlayer.com/address/0x06a852688E326D0596E786519E15168AE16C98B4), 25 transactions (16 SUCCESS, 9 intentional ERRORs from the negative tests). Claim 0 valid/high (violation count 0 → 1); claims 1–5 invalid/none — including a repeat of the rejected claim ruled invalid on its own grounding (not called a duplicate) and a repeat of the accepted claim ruled invalid as a duplicate. The severity invariant holds on all six claims. 9 negative tests rejected as expected. See `test_flow.md` for the full run.

Known boundary: submitting a claim to a nonexistent audit fails with a raw `KeyError` instead of a friendly message (the TreeMap lookup raises before any check). The submission is still rejected, so the behavior is correct; the message is just unfriendly.

## Layout

- `contracts/license_court.py` — the contract
- `test_flow.md` — the full Studio verification flow, step by step
- `examples/` — the demo fixtures (manifest, license report, exceptions page) and the sample inputs used in the flow
