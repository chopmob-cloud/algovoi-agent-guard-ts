# AlgoVoi Agent Guard (TypeScript)

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](./LICENSE) [![algovoi-agent-guard](https://img.shields.io/badge/algovoi--agent--guard-8A2BE2)](https://github.com/chopmob-cloud/algovoi-agent-guard) [![algovoi-fabric](https://img.shields.io/badge/algovoi--fabric-8A2BE2)](https://github.com/chopmob-cloud/algovoi-fabric) [![algovoi-secure-hub](https://img.shields.io/badge/algovoi--secure--hub-8A2BE2)](https://github.com/chopmob-cloud/algovoi-secure-hub)

**The budget your agent can't raise.**

A human signs a spend mandate. The agent holds its own key but never the human's.
Every charge is checked fail-closed *before* execution, and every decision produces
a receipt anyone can recompute offline from public material - no mediator signature
required.

⭐ **If this is useful, a star helps others find it.**

## Documentation

Full documentation: **[docs.algovoi.co.uk](https://docs.algovoi.co.uk)** — the
AlgoVoi documentation site.

| | |
|---|---|
| This product | [docs.algovoi.co.uk/agent-guard](https://docs.algovoi.co.uk/agent-guard) |
| How the layers fit | [docs.algovoi.co.uk/application-matrix](https://docs.algovoi.co.uk/application-matrix) |
| Free licence & self-host | [docs.algovoi.co.uk/free-licence](https://docs.algovoi.co.uk/free-licence) |

This is the TypeScript port of [`algovoi-agent-guard`](https://github.com/chopmob-cloud/algovoi-agent-guard).
It is **byte-conformant** with the Python reference and the estate's frozen public
L1 spend-authority conformance vectors: the same canonical bytes produce the same
refs and the same Ed25519 signatures in both languages.

```bash
npm install algovoi-agent-guard      # deps: @noble/curves, @noble/hashes (pure JS, no native build)
```

## Quickstart

```js
import {
  issueMandate, makeIntent, approve, Guard, policyBoundRef,
  guardrailRef, ref, Ledger, verifyDecision,
} from "algovoi-agent-guard";
import { mkdtempSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";

const humanPriv = /* base64url raw 32-byte Ed25519 private key */ "";
const now = new Date("2026-11-24T03:00:00Z");

// 1. A human signs a mandate: 1000 minor units / month, 600 per charge.
const env = issueMandate({
  payer: "0x00000000000000000000000000000000DEADBEEF",
  cap: "1000", humanPriv, period: "monthly", payee: "*", maxPerCharge: "600", now,
});

// 2. The agent proposes a charge; the human countersigns the prospective ALLOW.
const agentRef = ref({ type: "agent", id: "demo-agent" });
const allowRef = guardrailRef(agentRef, env.mandate_ref, policyBoundRef(env), "ALLOW");
const intent = makeIntent({
  chargeId: "charge-001", mandateRef: env.mandate_ref,
  payee: "merchant-a", amount: 400, asset: "USDC", at: now,
});
const countersign = approve(allowRef, intent, humanPriv, "2026-11-24T04:00:00Z");

// 3. The guard decides fail-closed and commits only an ALLOW.
const dir = mkdtempSync(join(tmpdir(), "guard-"));
const guard = new Guard(env, new Ledger(join(dir, "ledger.jsonl")), agentRef, () => now);
const receipt = guard.decide(intent, countersign);   // verdict: "ALLOW"

// 4. Any third party recomputes the decision offline from public material.
const [ok, reasons] = verifyDecision(env, receipt, guard.ledger.snapshot());
console.log(receipt.verdict, ok, reasons);           // ALLOW true []
```

## The one claim

Pre-execution **monetary** spend authority under a **human key the agent never
holds**, with receipts **recomputable offline** from public material.

| Neighbour | What it does | What it does not do |
|-----------|--------------|---------------------|
| Resource firewalls (e.g. pipelock) | Count requests/bytes/domains; mediator-attested identity | No monetary fields; identity is attested, not human-keyed |
| Approval binding (e.g. Countersign) | Bind a human "yes" to an action | Its own wire-transfer examples move no money |
| Adjudication (e.g. internet-court) | Settle agent-to-agent disputes after the fact | Does not stop the spend before it happens |

This stops an agent raising its own budget *before* the money moves, and lets a
third party recompute the monetary receipt without trusting a mediator.

## How it works

- **Mandate** - human-signed envelope; `mandate_ref = sha256(JCS({cap, payer,
  period, revocation_state}))` over a frozen, float-free RFC 8785 JCS form.
- **Guard** - pure `evaluate()` shared by enforcement and verification; checks
  authority binding → validity → payee → scope → per-charge cap → period cap →
  replay → human countersign bound to the exact intent. `setEnvelope` refuses any
  envelope with a different human key or payer.
- **Receipts** - `guardrail_ref = sha256(JCS({agent_ref, mandate_ref,
  policy_bound_ref, verdict}))`; recomputable, not mediator-signed. DENY receipts
  embed the presented countersign.
- **Ledger** - single-writer hash chain, full-history rewrite + atomic rename.
- **Keys** - Ed25519 via `@noble/curves` (raw 32-byte keys, 64-byte signatures),
  matching the Python `cryptography` model byte-for-byte.

## Cross-language conformance (the proof)

The frozen public vector files are the contract; both implementations are gated on
the same expected hashes.

- `test/conformance.test.mjs` - every frozen `payment_mandate_lite` and
  `spend_guardrail_lite` positive ref, negative recomputation and rejection
  reproduced byte-for-byte (11 tests).
- `test/interop.test.mjs` - TypeScript verifies a bundle the **Python** reference
  signed (mandate, countersign, ALLOW/DENY receipts, settlement, ledger), then
  rebuilds the **identical** bundle from the same key/instant and asserts
  byte-for-byte equality, including deterministic Ed25519 signatures (8 tests).
- `tools/verify_ts_fixtures.py` - the reverse direction: **Python** verifies the
  bundle **TypeScript** produced (8 checks).

## Tests

```bash
npm test                 # build + node --test (39 tests)
node tools/bench.mjs     # measured benchmarks -> bench.json
```

39 tests: 11 frozen-vector conformance, 8 cross-language interop, 20 guard
enforcement / tamper / JCS-edge / BigInt matrix. The Python↔TS interop is
additionally verified in reverse by the Python reference (8 checks).

## Fault tolerance & error correction

- **Fail-closed hostile input** - missing/malformed `at`, non-integer amounts,
  non-object intents and lone-surrogate strings DENY with a reason; they never
  raise into a fail-open path and never reach a canonicalisable ALLOW.
- **Recomputable refusals** - a policy/hostile-input DENY receipt embeds a
  canonical-safe form of the intent and re-runs `evaluate()` over it, so the
  refusal itself verifies offline just like an ALLOW.
- **Fail-closed ledger fault** - if the atomic append fails, the decision is a
  final DENY (`ledger: …`); the pure policy is *not* re-run, so a failed commit can
  never surface as an uncommitted ALLOW.
- **Crash-safe commit** - the ledger writes a temp file, `fsync`s, then renames;
  the prior chain stays intact and the lock is released on failure.
- **Tamper detection** - rewriting any entry breaks `verifyChain` and the dependent
  ALLOW receipt no longer verifies.

## Benchmarks

Real `performance.now()` runs, regenerated from each run's `bench.json`
([`BENCHMARKS.md`](BENCHMARKS.md), `tools/bench.mjs` + `tools/gen_benchmarks.mjs`).
Pure-JS Ed25519 is slower than a native backend (each decision verifies two
signatures); auditability and offline recomputability are favoured over throughput.
Hash-chain verify and tamper detection scale linearly and every injected mid-chain
mutation is detected. The scaling is reported, not hidden.

A captured 2026-11-24 VM2 run (Node 22, `node:22-slim`) is kept with its raw `bench.json`
and a comparison chart in [`docs/evidence/EVIDENCE.md`](docs/evidence/EVIDENCE.md).

## Honest limits

- **Ed25519 only** in this public lite format. The commercial post-quantum
  (Falcon-1024) engine is a separate product and is not reimplemented here.
- The guard **moves no funds**; it is an enforcement + verification layer emitting
  canonical refs for a settlement rail.
- The ledger is a single-writer chain, **not** distributed consensus.
- This port covers the **core library**. The autonomous red-team engine and demo
  CLI are part of the Python reference and are not reimplemented here.
- **No production adoption is claimed.** This is a runnable reference, not a
  deployed service.
- Replay is exact-`charge_id`: an agent can mint id variants to dodge replay, but
  every charge still counts against the period cap, so total spend stays bounded.
- **Amounts** are integers in minor units. `BigInt` is accepted up to
  `Number.MAX_SAFE_INTEGER` (2^53-1); beyond it the ledger refuses fail-closed,
  because JavaScript JSON cannot round-trip a larger integer exactly. The Python
  reference has arbitrary-precision integers; this is a language limitation of the
  TypeScript lite port, not a policy difference.

## AI disclaimer

This implementation was built and cross-validated with AI assistance. AI output is
fallible; the conformance and interop results are measured against the frozen
vectors and the Python reference, **not** certified by a model's verdict. The guard
moves no funds and signs with no real key. No warranty; no certified security audit
is claimed. Full terms: [`DISCLAIMER.md`](DISCLAIMER.md).

## Standalone, and wired into the AlgoVoi estate

Zero AlgoVoi runtime dependencies (`@noble/curves` + `@noble/hashes` only, pure JS,
isomorphic Node/browser). Emits refs byte-identical to the published conformance
vectors and to the Python reference, so it composes with
[`algovoi-fabric`](https://github.com/chopmob-cloud/algovoi-fabric) by construction.

The spend-authority guard is wired into the wider estate: the Python reference
bolts into [`algovoi-secure-hub`](https://github.com/chopmob-cloud/algovoi-secure-hub)
as a registered service (exercised live on VM2, 2026-11-24, posture `up:true`), and
this port is byte-conformant with that reference.

## License

Apache-2.0.
