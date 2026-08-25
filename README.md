# real-estate

`cloud-itonami/real-estate` is a global real-estate property and land-rights
data management system: property **listings** and purchase **offers** live as
AT PDS records, and an accepted offer's earnest deposit settles **on-chain**
(USDC on Base L2, ERC-4337, TitheRouter with the constitutional 10%
Public-Fund split). No Stripe, no RisingWave — the on-chain-only Tier-2
function split of ADR-2606011400, on the substrate posture of ADR-2605172000.

It was extracted verbatim from `etzhayyim/root`
(`60-apps/etzhayyim-project-real-estate`, seed copied 2026-05-21 — see
`migration.edn` for the exact source revision and tree) and is registered in
the west manifest as `orgs/cloud-itonami/real-estate`.

## Honest status

This is a **TRANSFORM-stage seed: the codemod is pending.**
`MIGRATION-TODO.md` is the authority for what remains — the TRANSFORM
classification came from the app's domain pattern, not from detected
violations (the automated scan found none), and manual Charter Rider
§2(a)–(h) review has not been done. `README.edn` still carries the original
migration identity (`com-etzhayyim-app-real-estate`); it is left untouched
because `migration.edn` pins it as one of the two allowed additions of the
extraction.

What *does* work today, verified by test: the `kotoba/` library's full
listing → offer → accept → settle round trip against the mock substrate.
See [docs/operator-quickstart.md](docs/operator-quickstart.md) for the
steps, actually executed.

## Layout

| path | what it is |
|---|---|
| `kotoba/` | TypeScript reference implementation on `@etzhayyim/sdk`: `listing` (create/get/list), `offer` (create/get/accept/settle), `tithe` (bigint 10% split, no rounding leak), `settlement` (the **only** value-transfer seam — wraps SDK `donate()`, per ADR-2605172100; tests inject a fake executor). Vitest tests in `kotoba/test/`. |
| `appview/real-estate-r3alestate/` | Thin edge facade Worker: `/health`, and `/xrpc/com.etzhayyim.apps.realEstate.*` proxied to the BPMN dispatcher. No business logic at the edge. |
| `worker/python/real_estate_worker.py` | Batch normalizer: JSONL rows (`source` / `property` / `listing` / `transaction`) submitted to the dispatcher for graph writes. |
| `data/` | `RealEstateProperty` JSON-LD schema + a sample property document. |

## Identity

- Controller: `did:web:real-estate.etzhayyim.com`; listings and offers get
  derived DIDs (`…:listing:{id}`, `…:offer:{id}`).
- Record collections: `com.etzhayyim.apps.realEstate.{listing,offer,payment}`.
- Amounts are USDC base units (micros) as decimal strings at the record
  boundary, `bigint` inside `tithe.ts`.

## Boundary with the nearest repos

The upstream seed lives on in `etzhayyim/root`; this repo is the standalone
extraction, and divergence from the seed is expected to happen *here*, not
there. Within `cloud-itonami`, the ISIC 68xx (real-estate industry) actors
are governed-actor repos with their own facts/render planes — this repo is
not one of them: it is an application seed (listings/offers/settlement
domain code), not an industry record actor.
