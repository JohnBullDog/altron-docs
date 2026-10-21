# Explainability Architecture

How ALTRON makes an autonomous agent's behaviour reconstructible: five independent
logging layers over a graph of small typed nodes, and a hard bound on model authority
at every point where a model touches control flow.

---

## 1. Why monolithic agents are not auditable

The dominant agent pattern is a single model in a loop: give it a goal, give it tools,
let it reason until done. Its "explanation" is the reasoning trace it emitted along the
way.

That trace is not an audit record, for three reasons.

**It is generated, not recorded.** The model produces prose describing its intent in
the same forward pass that produces the tool call. The prose is correlated with the
action but is not its cause, and nothing checks that they agree. A model can narrate
one plan and emit a different call.

**It has no denominator.** "I chose to scan the subnet" tells you what happened. It
does not tell you what else was available. Without the option set, "why this one" has
no answer — there is nothing to compare against, because the system never enumerated
alternatives in the first place.

**It is unstructured and lossy.** Reasoning traces are prose in a context window. They
get summarised, truncated, and dropped as the window fills. Six hours into a run, the
early reasoning is gone. Queries like "every command run against host X" or "every
time a permission was denied" require parsing natural language that was never meant to
be parsed.

ALTRON's answer is to make the decomposition structural and the logging a property of
the executor rather than of the model.

---

## 2. Layer one: the typed node graph

The unit of work is a node, not a conversational turn. There are 36 node types,
grouped by concern:

**Execution** — `core.shell` (exact command), `core.simple_task` (natural-language
step the model turns into a command), `core.local_compute` (compute a value locally and
bind it to a parameter), `core.compute`, `core.write_file`, `core.return`.

**Structure** — `org.root`, `org.sequence`, `org.parallel`, `org.conditional`,
`org.for_each`, `org.checkpoint`, `org.gen_plan`.

**State capture** — `core.finding`, `core.findings_query`, `core.session`,
`core.credential`, `core.persistence`, `core.verify_persistence`.

**Operational** — `core.open_terminal`, `core.send_to_process`, `core.pivot`,
`core.tunnel_open` / `_exec` / `_close`, `core.spray`, `core.mission_init`,
`core.schedule_template`, `core.bastion_message`, `core.briefing`.

**Knowledge** — `core.mitre_lookup`, `core.research`, `core.browse`,
`memory.set_param`, `memory.extract`.

**Composition** — `custom.<id>` recurses into another template, so graphs nest.

Three properties follow from this, and each one buys something specific for
explainability.

**Nodes are small enough to describe in one line.** A node is "run this command",
"record this finding", "iterate this list" — not "accomplish this objective". The
granularity of the log is therefore the granularity of the action.

**Nodes declare their inputs and outputs.** Parameters flow through explicit
`{{placeholder}}` binding and `param_out` declarations rather than through a shared
context window. When a node's command contains a value, you can trace which upstream
node produced it.

**Most nodes do not involve a model at all.** `core.shell` runs an exact string.
`org.parallel` is a scheduling primitive. `core.finding` is a typed write. The model
appears only where genuine ambiguity exists, which means most of the trace is
deterministic and the model-touched portion is identifiable as such.

### Determinism is the default, not the fallback

This is the design inversion that matters most. In a conventional agent the model
drives and deterministic code is a tool it calls. In ALTRON deterministic nodes are the
substrate and the model is consulted at specific, declared points.

The same inversion shows up in the semantic-event classifier, which labels a captured
command line with a reusable event name. A rule library runs first:

```python
DEFAULT_LIBRARY: list[dict] = [
    {"event": "checksum_run",  "regex": r"\b(md5sum|sha1sum|sha256sum|shasum|cksum)\b",
     "description": "Trainee computed a file checksum/hash"},
    {"event": "port_scan",     "regex": r"\b(nmap|masscan|zmap)\b",
     "description": "Trainee ran a network port scanner"},
    {"event": "privilege_escalation_attempt",
     "regex": r"\b(sudo\s+-l|linpeas|winpeas|getcap\b.*=ep)\b",
     "description": "Trainee attempted privilege escalation enumeration"},
    {"event": "log_clearing",  "regex": r"\b(history\s+-c|rm\s+.*bash_history|wevtutil\s+cl)\b",
     "description": "Trainee attempted to clear logs/history"},
    # ... extend via manifest/semantic_events.json, no code change
]
```

Rule-based matching is deterministic, fast, needs no model, and is inspectable by the
operator. The model is a fallback for lines the library does not recognise, and it is
scoped to returning one snake_case word or the literal `unknown`. Unrecognised lines
are logged as suggestions so the pattern library can grow from real usage — the system
gets more deterministic over time rather than less.

The source note on that module is explicit about the boundary:

> `classify_llm()` only ever labels data. It is not a decision-maker and does not
> compose or influence any reaction content — it is a narrowly scoped classification
> call (return one snake_case word, or "unknown").

---

## 3. Layer two: per-node event emission

The node dispatcher emits a start and a done event for every node, before and after the
body runs, regardless of node type:

```python
_emit_nd = lambda status: self._ws_emit({
    'type': 'task_node_start', 'data': {
        'id': nid, 'title': title, 'node_type': ntype, 'depth': depth,
        'parent_id': node.get('parent_id'),
        'action': node.get('action', ''),
        'flexible': _flex,
        'context_from': node.get('context_from') or [],
    }})
```

`depth` and `parent_id` mean the event stream reconstructs the full execution tree,
including nesting through `custom.<id>` recursion. `action` is the resolved command
after parameter substitution — what actually ran, not the template. `context_from`
records which upstream nodes fed this one.

### Skips are logged as skips

A node that does not run still emits both events, with a status explaining why:

```python
# A - skip nodes already completed in a prior run
if self._checkpoint and self._checkpoint.is_done(nid):
    cached = self._results.get(nid, "[resumed from checkpoint]")
    print(f"  [checkpoint] skip '{title}' (already done)")
    _emit_nd(nid)
    self._ws_emit({'type': 'task_node_done', 'data': {
        'id': nid, 'title': title, 'status': 'skipped'}})
    return cached
```

The planner can also override a node mid-run — skip it, or rewrite its action. Both
cases are logged, and a rewritten action writes a `replan` thought into the chain
recording the new value.

This sounds pedantic and is not. A node absent from a log is indistinguishable from a
node that never existed. If an operator is reconstructing a mission and node 14 is
missing, they cannot tell whether it was skipped by checkpoint resume, rewritten by the
planner, or silently failed. Emitting skips closes that gap, and every status value is
one of a known set.

---

## 4. Layer three: chain of thought as typed data

`TaskChain` is the per-task reasoning record. Critically, it is **passive** — a log,
not a decision-maker — and its entries are typed rather than prose:

```python
THOUGHT_KINDS = {
    "plan":      "PLAN",    # intent before execution
    "execute":   "EXEC",    # step is running
    "observe":   "OBS",     # result after success
    "stuck":     "STUCK",   # step failed, chain is blocked
    "heal":      "HEAL",    # self-heal attempt in progress
    "resume":    "RESUME",  # user chose to continue past a failure
    "note":      "NOTE",    # free-form finding to carry forward
    "monologue": "MONO",    # inner-voice reasoning decision
    "replan":    "REPLAN",  # plan was updated mid-execution
}
```

Each `Thought` carries a step index, a kind, content, an optional subtask id, and a
timestamp. Because the kind is a typed enum, the chain is queryable: every `STUCK`, in
order; every `HEAL` and whether a `RESUME` followed; every `REPLAN` and what changed.
That is not something you can do with a prose reasoning trace.

The chain drives two behaviours beyond logging. It supplies each subsequent subtask a
compact summary of what has already been found, so later steps have context without
replaying the whole history. And when a subtask fails after healing, the chain marks
itself blocked — the execution loop checks that flag and halts rather than continuing
silently past a failure.

---

## 5. Layer four: the structured audit log

Separate from reasoning, an append-only JSON-lines audit log records security-relevant
events with a thread lock around every write:

| Event | Fields |
|---|---|
| `tool_executed` | tool, command, task_id, approved_by |
| `tool_denied` | tool, command, task_id, reason |
| `permission_granted` | permission, task_id, tool |
| `permission_denied` | permission, task_id, reason |
| `api_call` | model, trigger, task_id, tokens_used |
| `self_heal` | tool, failure_hash, outcome, task_id, step_reached |
| `task` | task_id, sub_event, detail |
| `scheduler` | task_name, sub_event, detail |

```python
def _write(self, entry: dict) -> None:
    entry["ts"] = datetime.now(timezone.utc).isoformat(timespec="seconds")
    with self._lock:
        with open(self.log_path, "a") as f:
            f.write(json.dumps(entry) + "\n")
```

The design choices here are deliberate. **Append-only** — no entry is ever rewritten,
so the log cannot be retroactively tidied. **JSON lines** — greppable by a human,
parseable by a tool, and a partial write damages one line rather than the file.
**UTC ISO-8601** — correlates with external evidence without timezone guesswork.
**Denials recorded alongside grants** — "the agent tried to do X and was stopped" is
as important to an audit as what it did. **Token counts on API calls** — cost and
escalation frequency are measurable, not estimated.

Note what is *not* in this layer: no reasoning, no model output, no narrative. This log
answers "what did the system do and what was it allowed to do", and keeping it free of
generated text keeps it trustworthy as evidence.

---

## 6. Layer five: the durable mission event log

Mission execution emits its own event vocabulary — `branch_armed`, `gate_fired`,
`trigger_fired`, `decision_made`, `reaction_static`, and so on. Originally these were
broadcast once over a websocket for the live GUI and then discarded.

That was a real gap, and the fix is the clearest example of why this architecture
earns its complexity. From the module's own note:

> Nothing durably captured any of this before; `tick()`'s events were only ever
> broadcast once over the websocket for the live GUI and then gone. This is the raw
> material `mission_report.py`'s technical appendix and narrative report are built
> from.

The log is one JSON file per mission — following the per-entity-file convention already
used by the findings and credential stores rather than sharing a single file, because
event logs grow large over a long mission and have no reason to contend with every
other mission's writes.

### The race condition, and how the log's own absence revealed it

The first implementation of `append()` was an unprotected read-modify-write on a shared
JSON file. That is safe only if calls never overlap, and in practice they routinely do:
the mission executor's `tick()` calls it in a tight loop for every event in one tick,
the SSH bastion relays each trainee session on its own OS thread, the RPC server's
asyncio loop runs on the main thread, and scheduled tasks fire concurrently with all of
it.

Two overlapping appends silently lose whichever wrote first. The symptom, found during
a live mission:

> `reaction_static` events (which fire in bursts alongside several other events in the
> same tick, e.g. a Gate firing straight into its confirm Reaction) were completely
> absent from a real mission's persisted log despite clearly having fired — their
> downstream Triggers had armed and fired, which only happens after the Gate they
> depend on reached "done" — silently breaking the trainee page's Narrative Feed for
> every character-attributed `static_inject` reaction.

The diagnosis is worth reading twice, because it is inference from structure rather
than from a stack trace. The events were missing from the log, but their *consequences*
were present: downstream Triggers had armed, and a Trigger can only arm after its
upstream Gate reaches `done`. The graph's own causality proved the missing events had
occurred. A flat reasoning trace gives you nothing to make that argument with.

The fix is a `threading.Lock` rather than an `asyncio.Lock`, specifically because the
racing callers span both real OS threads and asyncio tasks on the main thread — an
asyncio lock would not have excluded the bastion's threads at all.

---

## 7. Bounded model authority

Logging establishes what happened. Bounding authority is what makes "why" answerable,
because it gives every decision a finite, enumerable option set.

### Intent routing: validated against a closed set

The classifier assigns one of six lanes. The prompt is built around a single
disambiguating question rather than a list of examples:

> **"If this request were completed successfully, what would exist that didn't
> before?"**
>
> - A) Something new exists only inside this chat (an answer, explanation, plan)
> - B) Something new exists outside this chat (a file, process, scan result, exploit run)
> - C) Live data that only exists in external sources (CVE details, live URLs, current intel)

Five numbered principles resolve the hard cases — notably **final output wins**, so
"research X then write code" classifies on the deliverable rather than the
intermediate step, and **depth distinguishes simple from complex**, so step count and
unknown inputs decide between the two task lanes.

Whatever the model returns is checked against an explicit set:

```python
_VALID_LANES = {"INFORMATIONAL", "SIMPLE_TASK", "COMPLEX_TASK",
                "DESIGN_MEMORY", "RESEARCH", "MISSION"}
```

Anything outside it falls back deterministically with a recorded confidence, rather
than propagating an invalid lane into the pipeline.

### Post-failure reasoning: four outcomes and a stagnation guard

`InnerMonologue` runs between every action. Unlike the chain, it is active — it decides
what happens next — so its output grammar is fixed:

```
THOUGHT: <reasoning in 1-2 sentences>
DECISION: RETRY | REPLAN | ABANDON
FIX: <corrected shell command>            (required if DECISION is RETRY)
NEW_STEP_TITLE: <short title>             (required if DECISION is REPLAN)
```

Four outcomes, one of which must be chosen; the `THOUGHT` line is the recorded
rationale, written into the chain as a `MONO` entry. Two guards prevent the classic
agent failure mode of looping forever on the same bad idea:

> If the same shell command is proposed more than once for the same step the monologue
> automatically returns `ABANDON` rather than looping forever. A hard `SAFETY_CAP` of
> 12 iterations is also enforced as an absolute ceiling.

The stagnation guard is deterministic and sits outside the model. The model cannot talk
its way past it.

### Scenario branching: the closed set is derived from the graph

The strongest constraint in the system. A `Decision` node asks a model to choose what
happens next in a mission — the one place a model genuinely selects a branch rather
than narrating around operator content. The safety property:

> The closed set of choices is **not** stored here — it's derived at evaluation time
> from which branches are actually wired to depend on this Decision node (same
> connections an operator drags in the editor), so the model's valid-answer set can
> never drift out of sync with the graph. `descriptions` only supplies the
> operator-written text for each of those branch_ids — the model sees this description,
> never the option's own `reaction_spec`/payload content.

Three distinct guarantees, each doing separate work:

1. **The option set is the graph.** Not a copy of it, not a list maintained alongside
   it. An operator rewiring the editor changes the model's option set in the same
   action, so the two cannot diverge.
2. **The model must return a `branch_id` verbatim, or the literal `none`.** Anything
   else is rejected. It cannot invent a branch, and it cannot act on an option the
   operator did not wire.
3. **The model sees descriptions, not payloads.** It reads the operator's framing of
   each option, never the option's actual reaction content. It cannot be steered by
   text inside a payload it was never meant to evaluate.

Plus a deterministic escape hatch: `timeout_seconds`, `retry_cooldown_seconds`,
`retry_enabled`, and `default_branch_id`. If the model times out or retries are
exhausted, the mission proceeds down an operator-chosen default rather than stalling —
and the fallback is recorded as a fallback.

That escape hatch earned its keep. In live testing across two scenarios the local model
**never once returned a valid choice** — the closed-set validation worked exactly as
designed and rejected every malformed answer, but a mechanism that always falls back to
its default is a default with a latency penalty. The safety property held; the
reliability property did not.

The response was to move the same closed-set mechanism somewhere a failure is cheap. A
reaction's `side_effect_mode: "approved_list"` has the model pick one pre-approved
template by ID using the identical grammar-constrained validation, but the reaction
fires either way — a poor pick still produces a reasonable action, where a failed
Decision stalls the mission until timeout. That became the default shape. See
[ENGINEERING_NOTES.md](ENGINEERING_NOTES.md#the-decision-node-worked-on-paper-before-it-worked-in-practice).

The general lesson is worth stating because it is easy to miss: a bound on model
authority is a correctness guarantee, not a reliability one. Validating that the model
returned a legal answer does nothing to make it return a *useful* one. Both properties
need designing for, and they are independent.

### Narrative generation: narrating, not authoring

A `Reaction` node's `llm_narrative` mode writes prose for a trainee. The boundary is
that the model never authors the content it narrates:

> `payload_context` is passed as immutable, opaque context the model never authors or
> judges — it only writes narrative text per `narrative_prompt` around content it
> doesn't control.

So a reaction can say "the adversary has re-established its foothold" in varied,
responsive prose, while *what* happened is operator-authored and, where it changes real
state, executed by a deterministic plan template via `side_effect_template_id`. The
model's output is flavour text over a fixed event.

Two further controls: `narrative_mode` can be `structured` rather than `freeform`,
assembling the prompt from `narrative_tone` / `speaker_role` / `what_happened` /
`what_next` fields and trading authoring flexibility for a narrower output shape. And
`narrative_fallback` holds pre-approved text — typically captured from a preview call —
used if live generation times out, so a slow local model cannot stall a mission.

### Failure healing: a deterministic gate before any model

The self-healer's first act is to decide whether a failure is a failure at all, with a
pattern table and no model involvement:

```python
# Tuples: (command_regex, exit_codes_that_are_benign, human_reason)
# Empty exit_codes set -> all non-zero codes are benign for this command.
_BENIGN = [
    (re.compile(r"^(e?grep|egrep|fgrep|rg|ag|ack)\b"),    {1},          "no matches (expected)"),
    (re.compile(r"^diff\b"),                               {1},          "files differ (expected)"),
    (re.compile(r"^(test\b|\[)"),                          {0, 1},       "test condition (expected)"),
    (re.compile(r"^(curl|wget)\b"),                        {22,6,7,28},  "http status code (expected)"),
    (re.compile(r"^(nmap|masscan)\b"),                     {1, 2, 3},    "scanner exit (expected)"),
    (re.compile(r"^(hashcat|john)\b"),                     {1},          "cracker status - exhausted (expected)"),
    (re.compile(r"^(wc|sort|uniq|cut|tr|head|tail|tee)\b"), set(),       "text-processing utility (expected)"),
]
```

`grep` exiting 1 means "no matches", which is frequently the correct and expected
result. `nmap` exits non-zero routinely. A healer that treats every non-zero exit as a
fault burns API calls on nothing and fills the log with phantom failures — which
corrupts exactly the record an operator needs.

A second heuristic catches the general case: a non-zero exit with empty stderr and
informational stdout is almost always a result code rather than a fault. A third gate
requires at least a 10-character error signal before diagnosis begins, and
classification reads stderr first, falling back to stdout only when stderr is empty,
with a 15-character minimum on the matched signal to avoid single-keyword false
positives in normal tool output.

Every skip is logged with its reason, so "the healer did not run" is itself an
explained event.

### Authoring-time assistance is not runtime authority

A fair question at this point is who writes the graph in the first place, and whether a
model doing that authoring quietly reintroduces what this architecture exists to avoid.

Using an AI coding agent to draft Plan Templates and mission graphs is **recommended,
and it is not required.** The node vocabulary is documented as a reference written for
that purpose — node types, required fields per type, worked end-to-end examples, and the
structural rules a valid graph has to satisfy — so describing a scenario in prose and
getting back a candidate JSON graph is a practical way to work. Hand-authoring the same
graph in the visual editor is fully supported and produces an identical artifact.

The distinction that matters is *when* the model acts:

| | Authoring time | Run time |
|---|---|---|
| What the model produces | A JSON graph — a static, inspectable artifact | A choice from a closed, enumerated set |
| What reviews it | A human, in the editor, before anything runs | A validator, against the wired option set |
| If it is wrong | The graph is edited or discarded; nothing has executed | Rejected, logged, deterministic fallback |
| Cost of a bad output | A draft that does not get used | None that is not recorded |

A model that drafts a graph holds no authority, because its output is not an action — it
is a proposal in a format a human reads, diffs, and edits before execution. The graph is
then fixed for the run: the authoring model is gone by the time anything executes, and
the executing system's own model authority is still bounded at each point above.

This is the inverse of the monolithic case in section 1, where the plan and the actions
are the same stream of tokens and there is no artifact in between to review. Here there
is always an artifact, and it is reviewable precisely because the graph is small typed
nodes rather than prose.

---

## 8. What this buys

With these layers in place, the three questions from the README have concrete answers.

**What did it run, in what order, against what?** The node event stream, with
`parent_id` and `depth` reconstructing the tree and `action` holding the
post-substitution command. Cross-referenced against the audit log's `tool_executed`
entries, which record the approver.

**What alternatives did it have?** For a Decision node, the wired branch set — which is
the graph, recoverable from the mission definition. For the monologue, the four fixed
outcomes. For routing, the six lanes. Every option set is enumerable because every
option set was enumerated at decision time.

**Why that one?** The `MONO` thought for execution decisions. The `decision_made` event
for scenario branching. The classifier's lane and confidence for routing. The
healer's recorded root-cause classification and the step the chain reached.

And a fourth, which only falls out of durable structured logs: **what did it get
stopped from doing?** Every `tool_denied` and `permission_denied`, with reasons.

### The costs, honestly

This is more machinery than a plan-and-execute loop, and the tradeoffs are real.

**Authoring overhead.** A 36-type node vocabulary is more to learn than "write a
prompt", and expressing a task as a graph takes longer than describing it in a
sentence. Plan templates and the mission editor exist to amortise that, but the first
graph an operator writes is slower than the first prompt they would have written.

**Expressiveness ceiling.** A model free to improvise can handle situations no node
type anticipates. ALTRON trades some of that for auditability, and `core.simple_task`
plus the flexible-planner overrides are the deliberate pressure valve — a node that
*is* model-driven, inside a graph that is not.

**Logging is not free.** Five layers of write means I/O on every node, a lock on every
audit and mission-event write, and real disk growth over a long mission. The
per-mission file split and the lock scope are both responses to that, and the race
condition above is what carelessness there actually costs.

The bet is that for an agent executing privileged actions against real systems under
written authorization, auditability is not a feature competing with capability — it is
the precondition for being allowed to run at all.

---

## See also

- **[ARCHITECTURE.md](ARCHITECTURE.md)** — subsystem map and execution lanes
- **[MISSION_TREE.md](MISSION_TREE.md)** — the node kinds and condition types referenced above
- **[ENGINEERING_NOTES.md](ENGINEERING_NOTES.md)** — further design decisions and bugs found live
