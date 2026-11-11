# Tasks

## 1. SETUP.md registration matrix

- [ ] 1.1 In §Register an organization, add the matrix table: org_genesis (required — token verification resolves the org key), edge_authorization (optional for pinning — verification never checks the edge registry; recommended on replicated DAGs for replica consistency), org_key_revoke + new genesis (required on key rotation; link existing rotation docs).
- [ ] 1.2 Add the "what /auth/verify actually checks" note (org key → signature → expiry/skew → scope → optional audience/action/payload_hash), stating explicitly that the edge registry is not consulted.

## 2. Cross-links

- [ ] 2.1 Link the matrix from the rotation/revoke section added in PR #19 (one line, no duplication).

## 3. Quality gate

- [ ] 3.1 `openspec validate clarify-ipfs-org-registration` passes.
- [ ] 3.2 Commit on `feat/clarify-ipfs-org-registration`, push, open PR.
