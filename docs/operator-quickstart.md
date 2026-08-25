# Operator quickstart

Every step below was actually executed on 2026-08-25 (Node v26.7.0 /
npm 11.19.0, Python 3, macOS) against commit `b10198e`. Observed outputs are
quoted as measured — if a step stops matching, the repo has drifted, not this
page.

## What you are operating

Two things are runnable from this repo today:

1. **`kotoba/`** — the TypeScript library (listings / offers / on-chain
   earnest-deposit settlement). Its test suite runs the full
   listing → offer → accept → settle round trip against
   `@etzhayyim/sdk-mock`; no network, no chain, no credentials.
2. **`worker/python/real_estate_worker.py`** — the JSONL normalizer
   (stdlib only). `--dry-run` shows exactly what would be submitted to the
   BPMN dispatcher without submitting anything.

What is **not** operable from here: the edge facade
(`appview/real-estate-r3alestate/`) needs a deployed dispatcher
(`DISPATCHER_URL` + internal secret) and is proxied infrastructure, not a
local target; and real on-chain settlement requires a configured
`@etzhayyim/sdk` `DonateConfig` — the tests deliberately inject a fake
executor instead. A quickstart that claimed you could settle USDC from this
page would be lying.

## 1. Install the library deps

```bash
cd kotoba
npm install
```

Both `@etzhayyim/sdk` deps are git-pinned GitHub URLs; npm clones and builds
them, so on a loaded machine this step takes several minutes.

⚠ **On this workspace's machines** the user-level `~/.npmrc` carries an
`allow-scripts[]=…` entry, and npm ≥11 refuses to forward it into the nested
project-scoped install that prepares the git deps — `npm install` dies with
`EALLOWSCRIPTS`. Every dependency of this package is public (GitHub +
registry), so a clean user config is a safe workaround:

```bash
NPM_CONFIG_USERCONFIG=/dev/null npm install --no-audit --no-fund
```

Measured 2026-08-25: exit 0 with the workaround; `EALLOWSCRIPTS` without it.

## 2. Run the tests

```bash
npm test
```

Measured output:

```
 Test Files  1 passed (1)
      Tests  7 passed (7)
```

The 7 tests cover: tithe 10% split with no rounding leak; listing
create/get/list; rejection of invalid property type and non-positive price;
offer creation and rejection when the listing is missing; deposit refused
before acceptance; accept → on-chain deposit (tithe split + offer
`deposited` + listing `under_offer`); and double-deposit refusal.

## 3. Typecheck

```bash
npm run typecheck
```

Measured: `tsc --noEmit` exits 0 with no output.

## 4. Dry-run the ingest worker

The worker is stdlib-only Python — no install step. Feed it JSONL rows with a
`kind` of `source` / `property` / `listing` / `transaction`:

```bash
printf '{"kind":"property","propertyId":"prop-001","location":"Tokyo","sizeM2":120}\n' \
  | python3 worker/python/real_estate_worker.py - --dry-run
```

Measured output (one line per accepted row, `dryRun: true`, nothing
submitted):

```
{"line": 1, "nsid": "com.etzhayyim.apps.realEstate.registerProperty", "result": {"dryRun": true, "nsid": "com.etzhayyim.apps.realEstate.registerProperty", "payload": {"propertyId": "prop-001", "location": "Tokyo", "sizeM2": 120}}}
```

And the refusal direction — an unsupported `kind` goes to stderr and flips
the exit code, so a bad batch cannot look like a clean one:

```bash
printf '{"kind":"nonsense","x":1}\n' \
  | python3 worker/python/real_estate_worker.py - --dry-run
# stderr: {"line": 1, "error": "unsupported kind: 'nonsense'"}
# exit code: 1
```

Dropping `--dry-run` POSTs each row to
`$DISPATCHER_URL/xrpc/<nsid>` (default `https://dispatcher.etzhayyim.com`) —
only do that against a dispatcher you own.
