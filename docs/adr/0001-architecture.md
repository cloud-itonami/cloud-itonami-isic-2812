# ADR-0001: FluidPowerAdvisor ⊣ Fluid Power Equipment Plant Operations Governor architecture

## Status

Accepted. `cloud-itonami-isic-2812` promoted from `:spec` to
`:implemented` in the `kotoba-lang/industry` registry, following the
verified fresh-scaffold protocol established by prior actors in this
fleet.

## Context

`cloud-itonami-isic-2812` publishes an OSS blueprint for fluid power
equipment (hydraulic/pneumatic pumps, cylinders, valves, motors)
**plant operations coordination** (production-batch product-type/
pressure-test/quantity/defect-rate data logging, machining/assembly/
pressure-test-bench-equipment maintenance scheduling, safety-concern
flagging, and outbound product shipment coordination). Like every
actor in this fleet, the blueprint alone is not an implementation:
this ADR records the governed-actor architecture that promotes it to
real, tested code, following the same langgraph StateGraph +
independent Governor + Phase 0->3 rollout pattern established across
the cloud-itonami fleet.

The closest domain analog is `cloud-itonami-isic-2814` (Manufacture of
bearings, gears, gearing and driving elements): both are back-office
coordination actors for a fixed manufacturing plant with QC-tested,
discrete-unit finished-goods output and a real physical/consumer
safety dimension, and both share the same four-op shape
(`:log-production-batch`/`:schedule-maintenance`/`:flag-safety-
concern`/`:coordinate-shipment`), the same two-entity verified/
registered gate structure (equipment for maintenance scheduling, batch
for shipment coordination), and the same permanent equipment-actuation
and certification-authority blocks. This build mirrors
`cloud-itonami-isic-2814`'s architecture closely but adapts the hazard
profile, equipment vocabulary, and product taxonomy to the fluid-power
plant: its finished goods are machined, assembled and pressure-tested
hydraulic/pneumatic components (hydraulic pumps, hydraulic cylinders,
hydraulic valves, hydraulic motors, pneumatic cylinders, pneumatic
valves) rather than bearings/gears, so its equipment kinds are
`:machining-line` and `:pressure-test-bench` rather than 2814's
precision-machining line and grinding line, and its routine QC field
is `:pressure-test-bar` (hydraulic/pneumatic proof-pressure test,
plausibility-checked 0-2000 bar -- informed by real fluid-power test
practice: rated system pressures for high-pressure hydraulic equipment
commonly run up to roughly 700 bar, and proof/pressure tests are
typically performed at 1.5x-2x the rated working pressure per ISO
4413 hydraulic / ISO 4414 pneumatic fluid-power-system practice)
rather than 2814's `:tolerance-test-um` (dimensional-tolerance test).
Like 2814, shipment quantity is tracked in finished-unit UNITS
(`:units`/`:quantity-units`/`:shipped-units`), since fluid power
equipment is likewise discrete counted units rather than a bulk
weight.

This vertical shares 2814's structural DOMAIN-SPECIFIC permanent
block, adapted to the fluid-power pressure-safety certification
regime: hydraulic/pneumatic fluid power equipment is subject to
pressure-equipment safety regimes (e.g. ASME BPVC, EU Pressure
Equipment Directive 2014/68/EU, ISO 4413 hydraulic / ISO 4414
pneumatic fluid-power-system safety practice). This actor is never the
certification authority -- any proposal (regardless of op) that
declares `:issue-certification? true` is a HARD, PERMANENT,
unconditional block
(`fluidpowermfg.governor/certification-authority-blocked-violations`),
the same "no phase, no human override" posture as the equipment-
actuation block.

This vertical has NO pre-existing `kotoba-lang/fluidpowermfg`-style
capability library to wrap (verified: no such repo exists). This build
therefore uses self-contained domain logic -- pure functions in
`fluidpowermfg.registry` (equipment/batch verification, shipment-
quantity recompute, product-type validation, pressure-test
plausibility validation, defect-rate plausibility validation) are
re-verified independently by the governor, the same "ground truth, not
self-report" discipline established across prior actors (most
directly `cloud-itonami-isic-2814`'s `beargearmfg.registry`).

This blueprint's own `:itonami.blueprint/governor` keyword,
`:fluid-power-plant-operations-governor`, is grep-verified UNIQUE
fleet-wide (`gh search code "fluid-power-plant-operations-governor"
--owner cloud-itonami`, zero hits before this repo was created).

## Decision

### Decision 1: Self-contained domain logic (no external fluid-power-manufacturing capability library to wrap)

Unlike actors that delegate to pre-existing domain libraries, this
fluid-power vertical has NO pre-existing capability library to
wrap. The equipment/batch-verification / shipment-quantity /
product-type / pressure-test / defect-rate validation
functions live as pure functions in `fluidpowermfg.registry` and are
re-verified independently by `fluidpowermfg.governor` -- the same
"ground truth, not self-report" discipline established across prior
actors (most directly `cloud-itonami-isic-2814`'s `beargearmfg.registry`).

### Decision 2: Coordination, not control — scope boundary at the back-office

This actor is **strictly back-office coordination** of fluid-power
plant operations. It does NOT:
- Control machining or assembly-line equipment directly
- Make plant-safety or certification decisions (exclusive to the human plant supervisor / accredited certification body)
- Actuate machining/assembly-line equipment
- Self-issue a pressure-equipment conformity or fluid-power-system safety compliance mark (e.g. ASME BPVC / PED 2014/68/EU / ISO 4413 / ISO 4414)

All proposals are `:effect :propose` only. The advisor proposes; the
governor validates; escalation paths funnel to human plant-supervisor
approval. This is not a replacement for the supervisor's authority or
the certification body's authority — it is a proposal-screening and
documentation layer.

**CRITICAL SAFETY BOUNDARY**: fluid-power-equipment manufacturing is a
safety-critical domain (machining/assembly/pressure-test-bench line
hazards, hydraulic-fluid and stored-pressure-energy hazards,
pressure-equipment certification, downstream mechanical/consumer-
safety consequence via the systems the batch's fluid power equipment
ends up installed in). Safety-concern flagging NEVER auto-commits. All
safety concerns escalate immediately to human review.

### Decision 3: Safety-concern escalation — always human sign-off

`:flag-safety-concern` (pressure-rating concern, hydraulic-fluid-
safety concern) ALWAYS escalates, never auto-commits. This is not a
"low-stakes proposal" -- it is a circuit-breaker that must reach human
authority.

### Decision 4: Two independent verified/registered gates (equipment AND batch), not one

Like `cloud-itonami-isic-2814`, this vertical has TWO entity kinds
each gating a different op: `:schedule-maintenance` independently
verifies the referenced **equipment** unit's own `:verified?`/
`:registered?` fields; `:coordinate-shipment` independently verifies
the referenced **batch**'s own `:verified?`/`:registered?` fields.
Both are the same "plant/batch record must be independently
verified/registered before any action" HARD invariant applied to the
two distinct record kinds this domain actually has.
`:coordinate-shipment` additionally independently recomputes whether a
batch's own recorded shipped-to-date unit quantity plus the
proposal's own claimed unit quantity would exceed the batch's own
recorded production quantity -- never taken on the advisor's
self-report.

### Decision 5: HARD invariants (no override)

Four HARD governor invariants (elaborated into twelve concrete checks
in `fluidpowermfg.governor`, mirroring `cloud-itonami-isic-2814`'s own
elaboration of its HARD invariants into concrete checks) block
proposals and cannot be overridden by human approval:
1. Plant/batch record (equipment for maintenance, batch for shipment) must be independently verified/registered before any action is taken against it, and a shipment's quantity must independently recompute within the batch's own logged production quantity
2. Proposals must be `:effect :propose` only (never direct equipment control)
3. Direct machining/assembly-line-equipment control, equipment actuation, or self-issued pressure-safety certification is permanently blocked
4. The op allowlist is closed — `:log-production-batch`/`:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` only

## Consequences

(+) Fluid-power-equipment plant operations back-office now has a
documented, governed, auditable coordination layer that funnels all
decisions through independent validation before human approval.

(+) The "coordination, not control" boundary is explicit in code: all
`:effect :propose`, all real-world actuation requires human plant-
supervisor sign-off, and no pressure-safety certification mark can
ever be self-issued.

(+) Scope is bounded and verifiable: four HARD invariants (elaborated
into twelve concrete governor checks) protect against scope creep into
unauthorized equipment operation, equipment actuation, or
certification self-issuance. Safety concerns are a circuit-breaker,
not a threshold.

(+) Safety-critical discipline is explicit: safety-concern flagging
cannot be rate-limited, suppressed, or auto-decided by phase gate.
Human review is mandatory.

(-) Still a simulation/proposal layer, not a real plant-operations
control system. Equipment actuation, line operation, and certification
issuance remain human-/institution-controlled via external channels.

(-) No integration with real plant-management databases (equipment
telemetry, batch tracking, freight dispatch, certification-body APIs)
— this is a standalone coordinator blueprint.

## Verification

- `cloud-itonami-isic-2812`: `clojure -M:test` green (all tests pass;
  see the superproject ADR and `kotoba-lang/industry` registry entry
  for the exact `Ran N tests containing M assertions, 0 failures, 0
  errors` output, verified from an independent fresh clone), `clojure
  -M:lint` clean, `clojure -M:dev:run` demo narrative exercises
  proposal submission, escalation, and every HARD-hold scenario
  directly (not-propose-effect, unknown-op, equipment-not-verified,
  batch-not-verified, shipment-quantity-exceeded, equipment-actuate-
  blocked, certification-authority-blocked, already-scheduled,
  invalid-product-type, invalid-pressure-test-bar, invalid-defect-rate).
- All source is `.cljc` (portable ClojureScript / JVM / nbb) — no
  JVM-only interop; the actor graph is invoked exclusively via
  `langgraph.graph/run*` (not `.invoke`, which is not cljs-portable).
- Audit ledger is append-only, all decisions are traced; every settled
  request (commit or hold) leaves exactly one ledger fact.
- `deps.edn` pins `io.github.kotoba-lang/langgraph` and
  `io.github.kotoba-lang/langchain` via `:local/root` directly in the
  top-level `:deps` (not only under a `:dev` alias), so a bare
  `clojure -M:test` resolves offline inside the monorepo checkout.
