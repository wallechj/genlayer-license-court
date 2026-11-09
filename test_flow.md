# LicenseCourt — Studio verification flow

Run this flow top to bottom in GenLayer Studio. Two test accounts: A (audit owner / deployer) and B (claimant). Check every write with a view read-back; a transaction that is FINALIZED and Consensus Accepted is not enough — the GenVM Result column must say SUCCESS for the intended calls, and the intended failures must show GenVM ERROR.

Fixtures (public, on `main`, fetchable as raw text before starting):
- `examples/license_manifest.json` — acme-web 1.4.2 manifest: runtime deps react, lodash, moment, sharp-lib, data-tool; devDep build-tool.
- `examples/license_index.md` — license report: react MIT, lodash MIT, moment MIT, sharp-lib GPL-3.0, data-tool Apache-2.0, build-tool GPL-3.0 (build-time only, not shipped).
- `examples/license_exceptions.md` — exceptions page: build-tool (GPL-3.0) approved for build-time use only, not shipped (2026-02-15).

## 0. Deploy

Deploy `contracts/license_court.py` from account A. The deploy transaction must show GenVM Result SUCCESS. Note the contract address.

## 1. Create audits (account A)

Create audit 0 from `examples/sample_audit.json`:
- project_name "acme-web", manifest_url `examples/license_manifest.json`, license_index_url `examples/license_index.md`, exceptions_url `examples/license_exceptions.md`, forbidden_licenses "GPL-3.0, AGPL-3.0".
- Read back `get_audit(0)`: owner is A, `violation_count` 0, `status` "open", `forbidden_licenses` "GPL-3.0, AGPL-3.0".

Negative tests (each must fail with the exact message):
- Empty `manifest_url` → `manifest_url is required`.
- `manifest_url` "not-a-url" → `manifest_url must be an http(s) URL`.
- Empty `forbidden_licenses` → `forbidden_licenses is required`.

Create audit 1 from `examples/sample_audit_dead.json`: same project, `manifest_url` `https://example.invalid/acme-web/package.json`, `exceptions_url` empty. `get_audit_count()` must be 2.

## 2. Submit claims (account B)

Submit, in order, from B:
- Claim 0 from `examples/sample_claim_gpl.json` (audit 0): "Forbidden GPL-3.0 dependency shipped" — sharp-lib is licensed GPL-3.0, which the project policy forbids, and it is a runtime dependency shipped in the distributed bundle.
- Claim 1 from `examples/sample_claim_react.json` (audit 0): "react is GPL-3.0".
- Claim 2 from `examples/sample_claim_buildtool.json` (audit 0): "build-tool is GPL-3.0".
- Claim 3 from `examples/sample_claim_gpl_audit1.json` (audit 1): same GPL claim as claim 0, against the dead-manifest audit.
- Claim 4 from `examples/sample_claim_repeat_rejected.json` (audit 0): title and statement identical to claim 1.
- Claim 5 from `examples/sample_claim_repeat_accepted.json` (audit 0): title and statement identical to claim 0.

Negative tests (each must fail with the exact message):
- Account A (the audit owner) submits a claim → `audit owner cannot submit claims`.
- `claimed_severity` "urgent" → `claimed_severity must be one of critical/high/medium/low`.
- Submit to audit 999 → error (audit does not exist).

`get_claim_count()` must be 6.

## 3. Assess claims 0–3 (any account)

Call `assess_claim` on claims 0–3, then read each back with `get_claim`, `get_claim_severity`, `get_claim_evidence`. Expected:

- Claim 0 → **valid / high**. Evidence must cite the fetched license report (sharp-lib GPL-3.0) and the manifest (sharp-lib is a runtime dependency).
- Claim 1 → **invalid / none**. Evidence must cite the license report: react is MIT.
- Claim 2 → **invalid / none**. Evidence must cite the exceptions page: build-tool (GPL-3.0) is approved for build-time use only and not shipped.
- Claim 3 → **invalid / none**. Evidence must state the manifest could not be retrieved, so there is no corroborating material.

After claim 0, `get_violation_count(0)` must be 1.

Negative test: assess claim 0 a second time → error `claim has already been assessed`.

## 4. Duplicate authority (any account)

- Assess claim 4 (repeat of rejected claim 1) → **invalid / none**. Its evidence must judge the claim on its own grounding — the license report lists react as MIT — and must **not** call it a duplicate of claim 1. Claim 1 was rejected, so it is not duplicate authority.
- Assess claim 5 (repeat of accepted claim 0) → **invalid / none**. Its evidence must call it a duplicate of the prior accepted finding (claim 0, valid/high).

## 5. Severity invariant

Read back all six claims: the only valid claim (0) carries a real bucket (high); every invalid claim (1–5) carries "none". `get_violation_count(0)` is still 1.

## 6. Lifecycle

- Account B closes audit 0 → error `only the audit owner can do this`.
- Account A closes audit 0 → SUCCESS; `get_audit(0)` shows `status` "closed".
- Account B submits a claim to closed audit 0 → error `audit is not open for claims`.

Leave audit 0 closed and audit 1 open.

## 7. Final read-back

- `get_audit_count()` = 2
- `get_claim_count()` = 6
- `get_violation_count(0)` = 1
- Audit 0 status "closed", audit 1 status "open"
