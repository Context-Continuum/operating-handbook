# Operating handbook for multi-agent production systems

This is the playbook for keeping multi-agent systems productive in
production. It's distilled from running a federated, multi-location,
multi-cluster Claude-agent environment as a two-person operation —
months of real coordinated work, real production artifacts, real
substrate findings.

The document is structured as an **inverse pyramid**: the most
load-bearing items are at the top. If you read only the first three
sections you'll have most of what matters for day-zero operation.
Everything below those three is refinement and reference.

## Contents

- **[OPERATING_HANDBOOK.md](./OPERATING_HANDBOOK.md)** — the full doc

### Quick map of the handbook

1. **TURN ZERO** — session pre-flight checks every agent performs
   before doing any work. Skip these and you eat hours of "the
   notifications never fired" debugging.

2. **BOX BOOT** — daemon + bridge runbook after a machine restart.
   Probe before relaunching. Daemon-readiness wait at the wrapper
   boundary, not in the python poll loop.

3. **BREADCRUMB SOP** — durable session handoffs triggered at a
   measured context-fill threshold (~88%). The mechanism that lets
   continuity survive compaction.

4. **PROCESS FORTIFICATION DOCTRINE** — respond, don't remember.
   Agents reliably respond to signals in context; they unreliably
   introspect to check whether a remembered trigger fired. Where
   possible, fortify SOPs with deterministic signals.

5. **SUBSTRATE-BOUNDARY DISCIPLINE** — enforce invariants at the
   write boundary (or init boundary), not at runtime. Refuse-at-
   write beats agent-vigilance at every measurable axis.

6. **CHANNEL ROUTING** — audience-based selection between scratchpad
   (intra-cluster, free), structured chat (cross-cluster, federated),
   and work-artifact items (anything worth keeping in a feed).

7. **IDENTITY MODEL** — agent ID at launch, never at runtime.
   Federated identity for multi-operator attribution.

8. **HARD PROHIBITIONS** — actions agents never take without
   explicit per-action approval.

9. **REFERENCE** — wake-filter routing, monitor-primitive vs
   bash-background distinctions, breadcrumb format, the long tail.

## Who this is for

Anyone running production agent systems where:

- More than one agent works in parallel
- More than one *location or operator* coordinates through a shared
  surface
- Sessions are long enough that compaction is a real event
- The substrate (filesystem, daemon, database) has invariants
  that downstream consumers depend on
- "Just be careful" is not an acceptable enforcement mechanism

If you're running a single one-shot LLM call, you don't need this.
If you're running multiple agents across one machine, you'll want
most of it. If you're running multiple agents across multiple
locations or operators, you need all of it.

## What's NOT in this handbook

This is operating discipline — the prose. It is not a framework,
not a library, not a runnable system. The implementations behind
these patterns (substrate primitives, observation pipelines, hook
telemetry sinks, bridge code, federation plumbing) are our
proprietary infrastructure and are not open-sourced. The value of
this handbook is that it's the playbook for *deploying and running*
those primitives — the part that doesn't come with any agent
framework you can `pip install`.

If you want to apply this discipline to your own agent system,
you'll be writing your own substrate. The handbook tells you what
the substrate needs to do; building it is the work.

## Versioning

This handbook is V0 — the public sanitized form of the V1.x
operating discipline we maintain internally. As more failure modes
get crystallized into substrate fixes, both versions grow. The
public V0 is updated periodically as patterns mature beyond
operator-specific detail.

Most recently codified additions (V0):

- Daemon-readiness wait at wrapper boundary (boot-order race)
- Self-restart of long-running monitor processes when their script
  mtime advances past process-start (PR-update propagation)
- Auto-detect-canonical-paths convention for wrapper scripts that
  invoke production code (config silent-degrade prevention)

See [case-studies](https://github.com/Context-Continuum/case-studies)
for receipts.
