# Operating handbook for multi-agent production systems

> An inverse pyramid. The most load-bearing items are at the top.
> If your context is tight, the first three sections are what matter;
> everything below is reference.

---

## TURN ZERO — do before responding to any user input

Three checks at the start of every session, performed before the
agent answers the first prompt. None require user permission; all
three are part of the operating contract.

### 1. Arm the wake monitor

The wake monitor is a long-running process that polls cluster
surfaces (a local scratchpad daemon, optionally a Firestore-backed
cross-cluster channel) and emits a notification when a post is
addressed to *this* agent specifically.

```
Monitor("python <repo>/tools/cluster_wake_filter.py")
```

Use the *Monitor primitive* — the tool surface that pipes stdout
lines back to the agent as notifications. **Do not** use a
background bash task with the same command. Both keep the process
alive, but only Monitor surfaces stdout as a wake event. A bash
background task looks healthy in `ps` and writes wake lines to
stdout, yet the agent never gets notified — because bash bg tasks
don't fire task-notification events.

This is a real trap. Two clusters have hit it at session start.
The symptom is "wake filter is running, but I'm never woken on
direct posts." The fix is "stop the bash bg task, re-spawn via
Monitor."

### 2. Verify agent identity

```
echo $CLUSTER_AGENT_ID      # bash / zsh
echo $env:CLUSTER_AGENT_ID  # PowerShell
```

The output must print the canonical agent id (e.g.
`Win/Claude-A`), not empty.

If empty, the wake-filter cursor naming and catch-up digest
pipeline collide whenever two agents share a filesystem
identifier (same hostname, same cwd). The fix is *at launch, not
at runtime*. Set `CLUSTER_AGENT_ID` in the shell before invoking
the agent — wrap it in a per-agent shell alias if there are
multiple agents on one machine.

The pattern: identity at launch is enforced once, by the operator,
deterministically. Identity at runtime is whatever the agent
happens to introspect — which is the wrong dependency.

### 3. Glance at the catch-up digest

The cluster's bridge auto-injects a "## Catch-up: cluster
activity since last turn" section into every prompt the agent
receives. Scan it for direct-addressed posts the agent may have
missed during compaction or overnight. If a catch-up section is
visible above the user's prompt, the agent is current. If not,
run the catch-up tool manually once before proceeding.

### When NOT to arm a monitor

- Read-only quick lookups (single command). Overhead exceeds
  benefit.
- Sandboxed sessions with no network access. The filter spins
  forever silently.
- The Monitor primitive isn't loaded in this session's tool
  palette. Note unavailability and proceed without it.

---

## BOX BOOT — daemon + bridge after a machine restart

When the operator signals "machine restarted, get the daemon and
bridge up," **don't re-hunt the launch commands.** They're
canonical scripts; run them. The whole boot is under 30 seconds
when you go straight at it.

### 1. Probe first

```bash
curl -s --max-time 2 http://127.0.0.1:7878/healthz
```

Healthy daemon returns `{"status":"ok", "embedder_dim":<N>, ...}`.

If the probe responds 200, the daemon is up. Don't relaunch a
healthy daemon — re-launching is wasteful and noisy. The probe is
what makes the boot fast.

### 2. Launch what's actually down

For each component that didn't respond healthy, invoke its
canonical launcher script. Don't paste env vars inline; don't
re-set environment variables that the launcher already sets.
Drift between an ad-hoc invocation and the canonical launcher is
exactly the kind of silent breakage that eats 10 minutes
debugging "why didn't my item show up?"

### 3. Daemon-readiness wait at the wrapper boundary

A common boot race: bridge + daemon both auto-fire on machine
boot. Bridge wins the race by a few seconds. Bridge's first
daemon-poll gets connection-refused; bridge's python process
exits; operator wakes up to a daemon that's running but a bridge
that quietly died, no items relayed.

The fix lives at the **wrapper boundary**, not in the python
poll loop. The bridge launcher script should poll
`<daemon>/healthz` in a wait loop *before* invoking the python
module. Two seconds between polls, 120-second total timeout,
proceed-with-warning on timeout. Boot-time race is now bounded
to whatever time the daemon needs to come up cleanly.

This is one instance of a generalizable pattern: wrapper-level
enforcement of an invariant the substrate downstream of it
depends on. The python code shouldn't carry boot-order retry
policy in its exception handling; the wrapper refuses to launch
python until the dependency is reachable.

### 4. Don't restart what's already running

Re-launching a healthy daemon is wasteful (the model cache
flushes, fresh log lines accumulate, the daemon momentarily
unresponds during restart). The probe in step 1 is what makes
this fast.

### 5. Arm the monitor

Machine restart = session restart. The prior wake monitor is
gone. Re-arm via the TURN ZERO recipe. Re-arming *last* (after
daemon + bridge) means the monitor is pointed at a healthy
daemon when it first polls.

---

## BREADCRUMB SOP — durable session handoffs at a measured threshold

The session-end counterpart to TURN ZERO. When the session's
context approaches **88% fill**, the in-flight work stops and the
agent writes a forward-looking *breadcrumb* to the cluster
scratchpad.

The trigger is a percentage of context fill, **not session-end**.
The breadcrumb is written *before* compaction kicks in, while the
agent is still fully-contexted, so it crystallizes from a high-
fidelity moment instead of a degraded post-compaction state.

### What the breadcrumb captures

A scratchpad post with four required fields, in order:

1. **What shipped** — commits, items, PRs, scratchpad/cluster-
   message IDs of substantive work this session. Cite by ID/hash,
   not just description.
2. **Deferred work** — what was started but not finished, what
   was discussed but not started, what blockers are pending.
3. **Cross-refs** — `decision_id`s touched, item URLs, PR URLs,
   follow-up IDs, related breadcrumbs being superseded or built on.
4. **State the next agent needs** — env vars at non-default
   values, in-flight messages awaiting reply, daemon/bridge
   restart gotchas discovered this session, anything an
   inheriting agent would have to re-derive otherwise.

### Why 88% specifically

- Above 88%, compaction is imminent. Writing the breadcrumb
  AFTER compaction starts means losing high-fidelity recall of
  what just happened.
- Below 88%, premature; the breadcrumb may be out-of-date by
  the time it's needed.

The 88% trigger crystallizes the session at peak fidelity. Treat
it as a hard interrupt — finish the current tool call, then
breadcrumb, then continue.

### Lifecycle

Breadcrumbs are append-only on the scratchpad. **No formal
rollover.** Three things substitute for explicit lifecycle:

- **Time-windowed catch-up surfacing.** The catch-up digest
  pipeline shows the last ~24h by default. Older breadcrumbs
  persist as durable reference but don't auto-surface.
- **Supersession by timestamp.** The freshest breadcrumb is
  canonical. Older ones become "how did we get here" lineage.
- **Cross-ref via `in_re_decision` / `breadcrumb_<id>`.** When
  a new lane revives an old breadcrumb's deferred work, the new
  lane's decision_id cites it explicitly. Keeps the graph
  navigable without forcing every lane to walk the whole chain.

### Edge cases

- **Operator-initiated session close before 88%** — write a
  breadcrumb anyway. Same shape, with a note that it's an EOD
  trigger not 88%.
- **Compaction kicked in before the agent noticed 88%** — write
  a post-compaction breadcrumb with an explicit "fidelity-
  warning: written post-compaction" tag so the next agent knows
  recall may be partial.
- **Mid-task at 88%** — pause the task, write the breadcrumb,
  resume. The breadcrumb captures the in-flight state so a
  hypothetical hand-off to another agent would land cleanly.

### Trigger fortification

The 88% threshold is one of the SOPs hardest to enforce purely
in prose — agents are bad at introspecting current context fill,
and self-reported estimates have been observed to be off by 3-4x.
The deterministic version reads the agent's session transcript
file directly and fires a notification when measured fill crosses
threshold. The agent's job becomes "respond to the notification"
(reliable), not "remember to check" (unreliable). See PROCESS
FORTIFICATION DOCTRINE below.

---

## PROCESS FORTIFICATION DOCTRINE

> Any SOP that depends on an agent **remembering** to do something
> is fragile. Agents reliably **respond** to signals in context;
> they unreliably **introspect** to check whether a remembered
> trigger has fired.

When a cluster norm exists, evaluate it against this three-step
audit and fortify if possible:

1. **Audit.** Identify the SOP — what is the agent supposed to do,
   and when?
2. **Measure.** Is there a file-system-observable proxy (file
   size, tool-call argument, event marker) that fires at the same
   moment the agent would have remembered? If yes → automate. If
   no → either redesign the SOP to *create* a measurable signal,
   or accept it stays prose-only with eyes open about its
   fragility.
3. **Wire.** Build the deterministic trigger (a session hook, a
   monitor process, a substrate validator) that injects a context-
   reminder when the signal crosses threshold. The agent's job
   becomes "respond to signal" (reliable), not "remember to check"
   (unreliable).

### Why this matters

Agent self-tracking failures observed in real operation:

- Eyeball estimate of own context fill: off by 3× (60-75% guess
  vs 20% real)
- Eyeball estimate after instrumentation: off by 4× (113% claim
  vs 20% real — wrong window-size assumption)
- Lane-stomp corrections from misreading "agent X isn't waking"
  as "stand down" vs "engage the LGTM ask"

In every case, the cost of getting it wrong was real. In every
case, a deterministic signal could have fired the right reminder
at the right moment.

### Cost discipline

Hooks run on every tool call or every prompt — they **must** be
cheap:

- Local file reads + regex only by default (microseconds)
- No API calls unless the signal genuinely requires them
- Use scoped matchers so a hook only fires when relevant
- Warn, don't block, unless the SOP violation is genuinely
  destructive

### Reference inventory

Maintain a living catalog of SOPs and their automation state.
Each new hook adds a row. Each prose-only SOP that gains a
measurable signal moves up the ladder (`prose-only` → `partial`
→ `automated`).

The inventory is itself a portfolio artifact. Reading it tells a
new agent which signals to expect, and tells the operator which
disciplines are still fragile.

---

## SUBSTRATE-BOUNDARY DISCIPLINE

The constructive corollary to "respond, not remember." Where
PROCESS FORTIFICATION asks *when* to add a signal, SUBSTRATE-
BOUNDARY asks *where* to enforce the rule once you have one.

Same skeleton — trust the substrate over runtime memory/judgment —
applied to two different axes:

| Axis        | Doctrine                       | Pattern                                                       |
|-------------|--------------------------------|---------------------------------------------------------------|
| Measurement | substrate-not-proxy            | Read authoritative data; don't estimate.                      |
| Enforcement | substrate-boundary-discipline  | Refuse at the schema/validator/sweeper boundary; don't ask agent vigilance. |

substrate-not-proxy and substrate-boundary-discipline are
complements, not duplicates. The first one says where to *read*
truth; the second says where to *enforce* truth.

### The test

Before relying on agent-judgment-at-runtime to enforce a rule,
ask:

> *Could this discipline be enforced by a refuses-at-write or
> refuses-at-state-transition check rather than agent vigilance?*

If yes → move it to the substrate. If no → flag it in the
inventory as known-fragile.

### Receipts

Six concrete instances where moving a rule to the substrate was
strictly cheaper than relying on agent vigilance:

1. **Urgency convention** — a pre-emit validator refuses
   wake-storm shapes at the scratchpad write boundary instead of
   asking the sender to remember which urgencies wake the cluster.
2. **Telemetry schema** — the validator rejects unknown fields at
   the sink boundary instead of asking writers to remember the
   schema.
3. **State-machine lock gate** — the `executing` transition
   requires all `open_questions` answered; the schema refuses the
   transition instead of asking the agent to self-check.
4. **Observation pairing matrix** — the validator refuses
   non-listed (claim_kind, evidence_kind) combinations at write
   time instead of asking the caller to remember the matrix.
5. **Result-kind enum** — a 4-value enum refuses free-form
   completion shapes at write time instead of asking delegates
   to remember the canonical names.
6. **Circuit breaker** — `escalate_after_n_attempts` substrate
   field plus sweeper-on-read enforcement caps tier-bumps instead
   of asking the agent to remember to stop retrying.

### Bureaucracy-as-token-cost

Bureaucracy in multi-agent systems isn't just deadlock — it's
tokens spent on coordination instead of work. Every "agent X must
remember to check Y before doing Z" produces coordination
overhead that scales with cluster size.

Substrate-boundary enforcement collapses that overhead to a
single refuse-at-write check, paid once at the boundary
regardless of how many agents pass through.

Push routing scales O(N×M) coordination tokens. Pull routing with
a substrate-side capability filter scales O(M). SUBSTRATE-BOUNDARY
DISCIPLINE pays for itself in both reliability *and* token cost
— they're the same property looked at from different angles.

### When to invoke this doctrine

- **Designing a new typed alias or substrate primitive** — ask
  the test before deciding what to enforce in code vs. document
  in prose.
- **Reviewing a "convention" cluster norm** — if the convention
  can be moved to a refuses-at-write check, propose the migration.
- **Auditing a session's failure modes** — for each "I forgot to
  do X" or "I assumed Y," ask whether the substrate could have
  refused the failure-shape at the boundary instead.

---

## CHANNEL ROUTING — audience-based

In a multi-cluster, multi-agent setup, three channels typically
exist:

| Sender → Recipient                       | Channel                       | Why |
|------------------------------------------|-------------------------------|-----|
| Agent → agent **same cluster**           | Local scratchpad daemon        | Free, zero API cost, stays on local daemon, doesn't pollute cross-cluster UI |
| Agent → agent **across clusters**        | Structured chat collection    | Only durable channels reach foreign clusters |
| Agent → agent **work artifacts**         | Items collection with brief + extended-report mirror | Heavy schema, lives in feed indefinitely |
| Human → any agent                        | Same chat collection (UI-side) | Humans on browsers can't hit the daemon |

**Rule of thumb:** if the recipient is on YOUR cluster's daemon,
use the scratchpad. Otherwise use the structured channel.

### The anti-pattern

Agent posting to the cross-cluster chat collection addressed to a
same-cluster agent. That puts cluster-internal chatter into the
cross-cluster scrollback for no benefit. Use the scratchpad
instead — wake reaches the recipient just as fast, and humans
don't see the intra-cluster noise.

### Positive corollary

The rule isn't about *volume*. It's about *channel choice*. The
sharp form:

> Don't use the cross-cluster channel to talk to agents you can
> already reach via scratchpad.

There's no "save it for important things" budget on the
scratchpad — it's free, daemon-local, and exists for exactly
intra-cluster coordination. Chatter freely.

Silence after a coordination event ("ok done, restarted on
task X") is also the wrong default — explicit telemetry beats
inference-by-absence; coordination is the scratchpad's purpose.

---

## IDENTITY MODEL — at launch, not at runtime

Two distinct uses of "operator identity":

| Use                                        | Identity                  |
|--------------------------------------------|---------------------------|
| Human UI sign-in (browser)                 | `<operator>@<domain>`     |
| Cluster bridge writes (SA-impersonated)    | `<operator>cluster@<domain>` |

Per-operator sign-in keeps audit trails distinguishable. The
bridge SA is what writes durable artifacts (items, build-lore .md
files); the human is what signs in to the UI.

Internal agent attribution (which Claude in the cluster wrote
this?) lives in the *artifact metadata* (`authored_by` field),
not in Firebase Auth. The bridge worker authenticates **once** as
the cluster operator and writes artifacts tagged with the
originating agent.

This separation lets a leaked bridge SA impersonate the *cluster*
but not individual agents at the audit level — and the human
operator emails stay distinct from the cluster bot's writes.

### Agent ID at launch

```
CLUSTER_AGENT_ID=Mac/Claude-A claude   # zsh / bash
$env:CLUSTER_AGENT_ID = "Win/Claude"; claude   # PowerShell
```

The agent shouldn't try to derive its identity at runtime by
introspecting `hostname` or `whoami`. Two reasons:

1. Macs share a hostname when multiple are open at once. Whoami
   doesn't disambiguate two Claude sessions on the same Mac.
2. Hostname-derivation is the kind of code that works during
   development and silently fails the day a new machine joins
   the cluster with an unexpected hostname.

Identity at launch is enforced once, by the operator, at the
shell — and inherited deterministically by everything downstream.

---

## HARD PROHIBITIONS

Actions an agent never takes without explicit per-action user
approval:

- **Don't push** or open PRs.
- **Don't run destructive git ops**: `reset --hard`, `push
  --force`, `branch -D`, `clean -f`.
- **Don't modify global git config**, never modify `.git/config`.
- **Don't `git checkout`** to a different branch from inside a
  worktree.
- **Don't write outside** the project root the session is
  currently working in.
- **Don't create `.md` docs** unless asked.
- **No filler preambles** ("Great question!", "I'll do X", "Let
  me think"). Get to the point.

The list is operator-specific in practice; the principle is that
*destructive* and *write-amplifying* actions need explicit user
approval, every time, no inferred consent.

---

## REFERENCE — long-form

### What the wake filter wakes on, and what it suppresses

A typical wake filter wakes when:

- The in-text header `[X → <your agent id> | ...]` addresses the
  agent directly (or multi-recipient form including the agent's id)
- A cluster broadcast tagged with urgency in `{action, blocker,
  urgent}` arrives
- A cross-cluster item with the agent's id in `_xClusterTo`
  arrives

A typical wake filter suppresses:

- Self-pings (sender matches the agent)
- Mission-control digest writes (those are the bridge's job)
- Routine cluster broadcasts at info-urgency (no wake)
- Already-seen messages (cursor)

### When a wake event fires

When a notification arrives:

1. Read it carefully — sender, recipient, message text.
2. Check whether it's actionable *right now*. Don't drop the
   user's in-flight request to chase a side ping.
3. If actionable: reply via scratchpad.
4. If not actionable now: acknowledge briefly with "queued, will
   look after current task." Even a one-line ack lets the sender
   unblock.
5. Never silently ignore a direct-addressed message.

### Monitor primitive vs background task

The wake filter (and similar long-running scripts) MUST be
spawned via the Monitor primitive, not via a background bash
task. Both keep the python process alive, but ONLY the Monitor
primitive pipes stdout lines back to the agent as notifications.
A background bash task with the same command appears healthy
(`ps` shows it running, the script writes wake lines to its
stdout file), and yet the agent never gets woken — because bash
bg tasks don't surface stdout as task-notification events.

The trap is real. Pin it explicitly in agent instructions.

### Breadcrumb format

```
[<sender> -> <recipient> | breadcrumb @ ~88% context] <one-line frame>
  Shipped: ...
  Deferred: ...
  Cross-refs: ...
  State for next agent: ...
```

Downstream lanes can reference the post as
`breadcrumb_<scratchpad_id>` in their `decision_id` chain when
picking up deferred work.

### Quarantine table (project scope discipline)

For sessions that work across multiple projects, maintain a
table that lists each project and its scope (writable, read-only
reference, frozen quarantined). The HARD PROHIBITIONS rule above
("don't write outside project root") becomes operational by
referencing this table.

---

*This handbook is the deployable form of operating discipline
that survived running multi-agent production for months. It is
not a framework. The implementations behind these patterns are
proprietary; the discipline is shared.*
