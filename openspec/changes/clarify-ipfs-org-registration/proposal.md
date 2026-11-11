# Proposal

## Why

`SETUP.md` §"Register an organization" documents only the org-genesis POST and
never states **which events an IPFS auth node actually needs**. This caused
real confusion in the field (integrator question, 2026-11-11): the pin-token
verification path (`verify_auth_token_full`) checks the **org key only** —
signature, expiry, scope, audience/action, payload_hash binding — and never
consults the edge registry, so operators cannot tell whether the
`edge_authorization` must also be POSTed, or what to do when an org key
rotates. The repo already documents revoke/rotation on the IPFS node (PR #19),
but the registration matrix itself is missing.

## What Changes

- `SETUP.md` §"Register an organization": add a **registration matrix** —
  - `org_genesis`: **required** (token verification resolves the org key from it);
  - `edge_authorization`: **not required for pinning** (verification never
    checks the edge registry) but **recommended** when the node participates in
    a replicated DAG, so all replicas hold the same events and later
    replication cannot diverge;
  - `org_key_revoke` + new genesis: required when an org rotates its key
    (link to the existing rotation section from PR #19).
- Add a short "what verification actually checks" note (org key → signature →
  expiry → scope → optional bindings) so operators understand why the edge
  event is optional.
- No behavior, compose, or image changes — documentation only.

## Capabilities

### New Capabilities

None (docs-only).

### Modified Capabilities

None.

## Impact

- `SETUP.md` only. Cross-reference from `README.md` if a registration section
  pointer exists there.
