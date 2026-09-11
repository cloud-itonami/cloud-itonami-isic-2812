# cloud-itonami-isic-2812: Manufacture of fluid power equipment

Open Business Blueprint for **ISIC 2812**: manufacture of fluid power equipment — an autonomous "actor" (LLM advisor behind an independent Governor, langgraph-clj StateGraph, append-only audit ledger) that coordinates back-office **fluid-power-equipment plant operations**: production-batch data logging (product-type/pressure-test/quantity/defect-rate), machining/assembly/pressure-test-bench-equipment maintenance scheduling, safety-concern flagging, and outbound product shipment coordination.

This repository designs a forkable OSS business for fluid-power-equipment-plant
operations: run by a qualified operator so a plant keeps its own
operating records instead of renting a closed SaaS.

## Scope: plant operations coordination, not machining/assembly-line control

ISIC 2812 covers the **manufacturing plant** that machines, assembles and pressure-tests finished hydraulic/pneumatic fluid power equipment (hydraulic pumps, hydraulic cylinders, hydraulic valves, hydraulic motors, pneumatic cylinders, pneumatic valves) — including pressure testing — before shipment. This actor coordinates the back-office record keeping around that plant — it never touches the machining/assembly-line equipment directly, and it is never a pressure-safety certification authority (e.g. ASME BPVC / PED 2014/68/EU pressure-equipment conformity mark or ISO 4413/ISO 4414 fluid-power-system safety compliance mark).

## What this actor does

Proposes **plant operations coordination**, not equipment operation:
- `:log-production-batch` — machining/assembly/pressure-test batch, output-quality data logging (administrative, not an operational decision)
- `:schedule-maintenance` — machining/assembly/test-bench-equipment maintenance scheduling proposal
- `:flag-safety-concern` — surface a pressure-rating/hydraulic-fluid-safety concern (always escalates)
- `:coordinate-shipment` — outbound product shipment coordination proposal

## What this actor does NOT do

**CRITICAL SCOPE BOUNDARY — this is a safety-critical domain**
(machining/assembly/pressure-test-bench line equipment, hydraulic-fluid
and stored-pressure-energy hazards, pressure-equipment certification,
downstream mechanical/consumer-safety consequence via the systems the
batch's fluid power equipment ends up installed in):

- Does NOT control machining or assembly-line equipment directly
- Does NOT make plant-safety or certification decisions (that's the plant supervisor's / certification body's exclusive human/institutional authority)
- Does NOT actuate machining/assembly-line equipment (human plant supervisor decides)
- Does NOT self-issue a pressure-equipment conformity or fluid-power-system safety compliance mark (e.g. ASME BPVC / PED 2014/68/EU / ISO 4413 / ISO 4414 — the accredited certification body's exclusive authority — a PERMANENT, unconditional block)
- ONLY proposes/coordinates operations back-office; all actuation and certification requires explicit human/institutional authority
- Safety-concern flagging ALWAYS escalates — never auto-decided, no confidence threshold or phase below escalation

## Architecture

Classic governed-actor pattern (`fluidpowermfg.operation/build`, a langgraph-clj StateGraph):
1. **`fluidpowermfg.advisor`** (sealed intelligence node, `FluidPowerAdvisor`): proposes decisions only, never commits
2. **`fluidpowermfg.governor`** (independent, `Fluid Power Equipment Plant Operations Governor`): validates against domain rules, re-derived from `fluidpowermfg.registry`'s pure functions and `fluidpowermfg.store`'s SSoT -- never trusts the advisor's own self-report
   - HARD invariants (always `:hold`, no override):
     - Plant/batch record must be independently verified/registered (`:verified?` AND `:registered?`) before any action is taken against it (equipment before maintenance scheduling, batch before shipment coordination)
     - The request's own `:effect` must be `:propose` (never a direct-write bypass)
     - `:op` must be in the closed four-op allowlist
     - The proposal's own `:effect` must be one of the four propose-shaped effects (no direct machining/assembly-line-equipment control)
     - Directly actuating machining/assembly-line equipment (`:actuate-equipment? true`) is a PERMANENT, unconditional block
     - Self-issuing a pressure-equipment conformity or fluid-power-system safety compliance mark (`:issue-certification? true`, any op) is a PERMANENT, unconditional block
     - A shipment may not push a batch's own recorded shipped quantity past its own logged production quantity (independently recomputed)
     - No double-scheduling the same maintenance record
     - No fabricated `:product-type` value on a production-batch patch
     - No physically implausible `:pressure-test-bar` value on a production-batch patch
     - No physically implausible `:defect-rate-percent` value on a production-batch patch
   - ESCALATE (always human sign-off, overridable by a human):
     - `:flag-safety-concern` always escalates, regardless of confidence
     - Low-confidence proposals
3. **`fluidpowermfg.phase`** (Phase 0->3 rollout): `:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` are NEVER in any phase's `:auto` set (permanent, matching the governor's own posture); only `:log-production-batch` may auto-commit at phase 3 when clean
4. **`fluidpowermfg.store`** (append-only audit ledger + SSoT): a single `MemStore` backend behind a `Store` protocol (see ns docstring for why a second Datomic-backed backend is out of scope for this build)

## Development

```bash
# Run tests (top-level deps.edn already pins langgraph+langchain local/root)
kbb -M:test

# Run tests via the workspace :dev override alias (equivalent, kept for sibling-repo parity)
kbb -M:dev:test

# Run the demo
kbb -M:dev:run

# Lint
kbb -M:lint
```

## Status

`:implemented` — `governor.cljc`/`store.cljc`/`advisor.cljc`/`registry.cljc` + `deps.edn` complete the module set; tests green, demo runnable, langgraph-clj integration verified.

## License

AGPL-3.0-or-later
