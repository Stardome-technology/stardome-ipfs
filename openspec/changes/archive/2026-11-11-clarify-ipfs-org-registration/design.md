# Design

## Context

See proposal.md - Why. Ground facts (verified in `stardome-sead-private`):
`lib/auth/verifier.hpp::verify_auth_token_full` = resolve org pk (KeyResolver,
60 s cache) → XMSS verify → expiry/skew → scope → optional audience/action →
optional payload_hash binding. No edge-registry lookup anywhere in the path.
The IPFS compose stack runs its own sead-core replica, so DAG consistency is
the only reason to POST edge-authorization there.

## Goals / Non-Goals

- Goals: one unambiguous table (event × required/optional/reason); one
  paragraph explaining the verification path; keep the existing security notes
  intact.
- Non-Goals: no changes to the compose stack, nginx config, or verification
  behavior; no new tooling.

## Decisions

- **Matrix as a Markdown table in §Register an organization** rather than a new
  top-level section: the confusion happens exactly at that step; the reader
  must not have to navigate away.
- **Wording for edge-authorization**: "optional for pinning, recommended for
  DAG consistency on replicated deployments" — matches what we actually do on
  ipfs/ipfs2 (both events POSTed).
- **Link, don't duplicate** the rotation procedure already documented by PR #19.

## Risks / Trade-offs

- Doc drift if verification ever starts checking edge authorization — the
  matrix states the current check list explicitly so a change is visible.
