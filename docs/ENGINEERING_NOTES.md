# Engineering Notes

Design decisions, tradeoffs, and bugs found the hard way. Most of what follows came out
of running real scenarios against live Docker labs rather than from code review or unit
tests, which is itself the main lesson.

---

## Testing methodology: live labs over unit tests

ALTRON has 7 test modules covering the pipeline, templates, LLM routing, scope,
research, and cron missions. They catch regressions. They caught almost none of the
bugs below.

The bugs that mattered needed a running system: concurrent bastion threads, an asyncio
RPC loop on the main thread, real Docker containers with real PID namespaces, a local
model under real load, and a human or agent actually playing the scenario. Four
scenarios were built and run end to end for this purpose — `hospital-target`,
`hospital-network`, `Contested Ward`, and `hospital-v2` — and each one surfaced
platform bugs that had been dormant, sometimes for weeks.

That is not an argument against unit tests. It is an argument that for a system whose
correctness is mostly about concurrency, timing, and integration with the host, the
test that finds things is the one that runs the whole thing for real.

A related discipline, learned by getting it wrong: **a branch showing `listening` only
proves a reaction dispatched, not that its side effect succeeded.** Verification has to
read the executor's own `task_node_done` status, not mission-tree branch status. Of 14
persistence items in Contested Ward, 11 had directly-confirmed-successful countermeasures
— confirmed from node-level logs, not branch state.

---

## Bugs worth reading

### The `artifact_state` trigger that could never fire

The best bug in the project, because it was invisible to every form of review.

An `artifact_state` Trigger watches the persistence registry for a state transition —
typically "this mechanism is now dead", meaning the trainee evicted it. Triggers are
edge-triggered: they fire on a *new* satisfying event since the last check, tracked with
a cursor.

The condition's `match_spec` includes the target status, because that is the whole point
— you are watching for a transition *to* `dead`. So the query was filtered by status
before the cursor logic ran.

Which made it impossible to fire. The first observation of a matching entry and the
transition being detected were the same event. There was no prior state to compare
against, because the pre-filtered query could never return the entry in its *previous*
status. The cursor had nothing to advance from.

The condition type had existed for days and had never been wired to anything real. It
looked correct in isolation; the design note explaining the cursor was coherent; a unit
test asserting "fires when an entry becomes dead" would plausibly have passed by
constructing exactly the state the real query could never produce. It took wiring it to
a live four-host scenario to discover that the mechanism was structurally incapable of
doing its job.

**Lesson:** a filter that encodes the thing you are detecting destroys your ability to
detect it. When watching for a transition, the query has to be able to see both sides.

### The scheduler that was never started

`core.schedule_template` registers a plan template to run later. It worked in the sense
that registration succeeded and returned cleanly.

`Scheduler.run_forever()` was never called anywhere in production code.

Every scheduled template, in every plan template — not just the lab being tested —
registered and then silently never fired. The node reported success, because
registration *was* successful. Nothing downstream ever checked that the scheduler which
would eventually run the task was itself running.

**Lesson:** "the call succeeded" and "the thing will happen" are different claims. A
registration primitive needs to verify its consumer exists, or the success it reports is
a lie by omission. This is the single most embarrassing bug in the project and the one I
would mention first in an interview, because it is a whole category: the component that
nobody starts.

### Zombies and `pgrep -f`

Persistence items are checked for liveness with a `check_command`. Several used
`pgrep -f <pattern>` or `ps | grep` to confirm a planted process was running.

These false-positived forever once the trainee killed the process. In these containers
PID 1 is `tail -f /dev/null`, which never reaps children. A killed process becomes a
zombie and stays one. And `pgrep -f` still matches a zombie, because the pattern is
matched against `/proc/<pid>/stat`'s comm field, which a zombie still has.

So the trainee successfully killed the implant, and the registry cheerfully reported it
alive, and the eviction gate never fired.

**Lesson:** container PID 1 is not `init` unless you made it `init`. Process-liveness
checks that are correct on a normal Linux host are wrong inside a container with a
non-reaping PID 1 — and this is a trap for anyone building Docker-based labs, not just
this project.

### The mission event log race

Covered in detail in
[EXPLAINABILITY.md §6](EXPLAINABILITY.md#the-race-condition-and-how-the-logs-own-absence-revealed-it).
Summary: `append()` was an unprotected read-modify-write on a shared JSON file, and the
callers span real OS threads (one per bastion session) *and* asyncio tasks on the main
thread. Overlapping appends silently lost whichever wrote first.

Two things make it worth revisiting. The diagnosis was inference from graph causality
rather than from a stack trace — the missing events' downstream Triggers had armed,
which only happens after the upstream Gate reaches `done`, so the graph proved events
had fired that the log did not contain. And the fix had to be a `threading.Lock` rather
than an `asyncio.Lock`, specifically because an asyncio lock would not have excluded the
bastion's OS threads at all.

**Lesson:** know which concurrency primitive your racing callers actually live in. "Add
a lock" is not the fix; "add the right kind of lock" is.

### The GUI and the executor did not share state

An operator reported "I don't see any templates in the graph" for reactions authored
with `side_effect_mode: "approved_list"`.

The mission editor decided whether to render the side-effect section at all by checking
`side_effect_template_id !== null`. But `approved_list` reactions store their data
entirely in `side_effect_template_options` and never set `side_effect_template_id`. Any
reaction authored programmatically — by a setup script rather than through the GUI's own
checkbox — rendered its entire side-effect block hidden: the checkbox, the mode selector,
and the template options.

The reactions **fired correctly the whole time**, confirmed by live verification. The
executor never reads that GUI state. Nothing was functionally broken; the configuration
was simply invisible and uneditable to the operator.

A second bug hid behind the first: unchecking "has side effect" only cleared
`side_effect_template_id`, so an `approved_list` reaction's checkbox would have refused
to stay unchecked — the same class of bug an adjacent code comment already documented
for a different case.

**Lesson:** two readers of the same data with independent interpretations of "is this
field set" will diverge, and the divergence is silent when one of them is a UI.
"It fires correctly live" is not evidence the GUI renders it correctly. For a system
whose premise is auditability, a config an operator cannot see is a real defect even
when the behaviour is right.

### The self-heal wrapper masked the real error

A countermeasure task failed with:

```
[FAILED: No Anthropic API key configured...]
```

That describes the *escalation path* failing. The actual error was an SSH
authentication failure — the shared first-hop service account had been deleted earlier
in the test pass, breaking every subsequent host's countermeasure relay.

The self-healer caught the SSH error, classified it, climbed the escalation ladder,
found no API key at the top, and reported *that*. The original error was consumed.

**Lesson:** an error-handling layer that replaces the original error with its own
failure is worse than no layer. Always reproduce the dispatched command by hand with
`-v` rather than trusting a wrapped task log. This has not been fully fixed, and is
listed below.

### `all` logic means two different things

A Gate's `all` logic accumulates: conditions can be satisfied by separate events over
time, and the Gate fires once the set is complete. A Trigger's `all` logic is
per-event: all conditions must match within the same evaluation.

A trainee typed two removal commands as two separate lines. The Gate (cumulative) was
satisfied. The Trigger-driven retaliation (per-event) never dispatched at all. Joining
them into one semicolon-separated line fixed it immediately.

Both behaviours are defensible in isolation and follow from level versus edge
triggering. Sharing the word `all` between them is the actual defect, and it is a naming
problem that produces a correctness problem.

---

## Design decisions

### The Decision node worked on paper before it worked in practice

The `Decision` node — a model picking from a closed set derived from graph wiring — is
the strictest safety design in the system. It is also, empirically, the mechanism that
gave the most trouble.

Across the `hospital-target` and `hospital-network` scenarios, the local model **never
once returned a valid choice.** Not a safety failure — the closed-set validation worked
exactly as designed and rejected every invalid answer — but a usability failure. A
mechanism that always falls back to its default is a default with extra steps and a
latency penalty.

Contested Ward's coaching Decision, on an 18-minute interval, was the first
`reevaluate`-style Decision across any scenario to return a valid choice on the first
try. It also correctly chose to stay silent given active trainee behaviour, which is the
right answer and the one a naive implementation would never pick.

The resolution was not to fix the Decision node. It was to stop reaching for it:
**`llm_narrative` reactions with `side_effect_mode: "approved_list"` and no Decision
node anywhere** became the proven default for "the model picks a varied real action".
That shape survived every subsequent round of iteration.

The mechanism is the same closed-set grammar. The difference is where it sits: a
side-effect pick inside a reaction that is going to fire regardless, rather than a
control-flow decision the whole mission waits on. If the pick is poor, the reaction still
fires with a reasonable action. If a Decision node fails, the mission stalls until its
timeout.

**Lesson:** the safety property and the reliability property are independent. The
closed-set design was sound and provably prevented invalid actions; it was the
*dependency structure* around it that made failures expensive. Put the fragile component
somewhere a failure is cheap.

### Regex out, real state in

An explicit design correction, from the project's own direction: *"the mission should
always rely on checking artifacts/states instead of regex when possible — this was part
of the design of the tool."*

Every Gate and Trigger in Contested Ward was rewired off `trainee_input` regex matching:

- **Removal and retaliation** now check `artifact_state` — the real `PersistenceRegistry`
  status, kept fresh by a 15-second `core.verify_persistence` heartbeat — instead of
  matching a removal command with a regex.
- **Discovery** now checks `semantic_event`, fed by a real two-way trainee chat and
  rule-classified, instead of matching a discovery command.

The difference is between detecting an *attempt* and detecting an *outcome*. A regex on
`rm /etc/cron.d/backdoor` fires when the trainee types something that looks like
eviction. An `artifact_state` check fires when the mechanism is actually gone.

This was verified in the most satisfying possible way: the tester killed the process via
`docker exec` directly, bypassing SSH and the suggested command entirely, and
`artifact_state` correctly detected it dead and dispatched retaliation. A regex-based
gate would have seen nothing at all.

It also, as noted above, surfaced three dormant platform bugs — because both condition
types had existed for days without ever being wired to anything real.

### Determinism first, everywhere

The same ordering recurs in four independent subsystems, and it was arrived at
separately each time:

| Subsystem | Deterministic first | Model only when |
|---|---|---|
| Semantic events | Regex pattern library | Line unrecognised; returns one label |
| Context condenser | Regex entity extraction + density scoring | Low confidence on a large document |
| Self-healer | Benign-failure table, error-signal gates | A real failure needs diagnosis |
| Mission report | Entirely deterministic | Never — no model in the module at all |

Each is cheaper, faster, inspectable, and reproducible in its deterministic path. The
model handles the residue. The semantic-event library also *grows* from unmatched lines
logged as suggestions, so the deterministic fraction increases with use.

### The report has no model in it

An earlier version fed mission facts into a local-model narrative pipeline — outline,
sections, review, PDF. It was removed:

> The local model both wedged Ollama under concurrent load and produced prose that
> barely referenced the real facts it was given directly.

Two independent failures. The concurrency one was practical: report generation competed
with the live mission for the same local model. The quality one was more important — a
model handed a complete factual record produced prose that drifted from it, which is
exactly the failure an after-action report cannot have.

The boundary now sits at the system edge. ALTRON produces facts deterministically and
completely; rubric scoring is a graph query over which branches reached `done`. An
instructor who wants narrative hands that document to a model of their choosing, outside
the audit path.

**Lesson:** when the output is a record of what happened, generated text is a liability
rather than a feature. Knowing where *not* to use a model is part of building with one.

### Answer keys must not touch the target's disk

A late addition: a `script_body` primitive for `core.shell` that runs an ephemeral
script instead of writing one to the target host.

The problem it solves is specific to training. A plan template that writes a setup or
countermeasure script to the host leaves that script on disk — where the trainee, who
has a shell on that host, can read it. The scenario's answer key leaks through its own
implementation.

`script_body` passes the script body through to execution without persisting it, so
there is nothing on disk to find.

**Lesson:** in an adversarial training system the trainee is a legitimate attacker
against your own tooling. Implementation artifacts are part of the attack surface.

### Backward compatibility as a constraint

The mission tree was added alongside an existing mission pipeline, not instead of it.
`node_kind` defaults to `"stage"`, so missions stored before the feature existed load and
run through the original pipeline unchanged. A legacy shim also folds the earlier
separate `recurring_decision` kind into `decision` with `reevaluate: True` on load, so no
stored mission broke when those concepts were unified.

Shipping a parallel execution path rather than migrating meant no existing scenario had
to be rewritten, at the cost of two code paths to maintain. For a system with live
scenarios already authored against the old shape, that was the right trade — but it is a
trade, and the second path is real complexity that will eventually need paying down.

### Reuse the established pattern, except when you shouldn't

Two decisions that look contradictory and are not.

**Reuse:** the bastion's upstream SSH leg reuses the paramiko password-auth client
pattern already established in the tunnel manager, and follows the same "blocking
paramiko work runs in a thread" convention. The mission event log follows the
per-entity-file persistence convention already used by the findings and credential
stores.

**Don't reuse:** the original plan assumed Gate event wiring could reuse the scheduler's
`fire_trigger()`. Reading the scheduler in full showed it fires one pre-registered task's
fixed instruction through a `run_callback(instruction, project) -> str` signature — not a
generic event bus. Retrofitting it would have meant every Gate owning a `ScheduledTask`
whose "instruction" had no sensible way to call back into gate evaluation. The executor
got its own lightweight signal intake instead:

> Simpler and more honest: this executor owns its own lightweight signal intake rather
> than being forced through machinery built for a different purpose.

The distinguishing question is whether the existing thing actually does what you need or
merely has a similar name. Both calls required reading the existing subsystem properly
instead of trusting the plan — which is the actual skill, and the plan was wrong in one
of the two cases.

---

## Platform interactions that are not bugs

Two behaviours that look like defects, are correct, and are easy to misdiagnose. Both
are documented because the misdiagnosis cost real time.

**Severe-tier countermeasures block each other.** The severe countermeasure takes `sshd`
down on a host for about five minutes. The countermeasure itself needs to SSH into that
same host. So firing two severe items on one host within five minutes means the second
one's retaliation fails with a `network_error` self-heal classification — regardless of
whether that item's own access is intact. Correct behaviour, confusing symptom: check
whether another severe countermeasure landed in the preceding five minutes before
assuming a broken relay.

**Removing an access-granting item breaks everything tested after it.** The first-hop
service account is shared by every host's countermeasure relay. Removing it "last among
this host's items" is not "last in the playthrough" — every subsequent host's retaliation
fails with an SSH auth error, surfaced as the misleading escalation-path message described
above. Recovery needs *every* affected container rebuilt, not just the one whose account
was deleted, because a host whose real persistence artifact was already killed during the
contaminated pass will still fail retaliation for an unrelated reason.

---

## Process lessons

**Design feedback is worth more than a passing test.** Watching a genuinely blind agent
play through Contested Ward produced a critique no test could: the trainee plays a network
admin, not an external attacker. Realistically they would already hold legitimate
credentials across their own network. The build made every pivot a "find the leaked
secret" puzzle regardless of whether the adversary had touched credentials at all — an
attacker-discovery framing bootstrapped onto an admin role.

That is a scenario-design error, not a code defect, and it was deliberately *not* fixed
mid-playthrough to avoid invalidating the run in progress. Noted for a future revision:
give the trainee legitimate cross-host credentials, and if credential theft is part of the
story, make it an explicit discoverable adversary action rather than the sole mechanism
for reaching every host.

**Parallel test agents need isolated identities.** Running multiple automated agents
against one live mission with a shared login produced a collision that was initially
reported as a platform bug and was not one. A separate run saw a background agent silently
scope-drift for over two hours. Both are now explicit preconditions: isolated credentials
per agent, and bounded checkpoints rather than long unattended runs.

**The state is rarely pristine.** `PersistenceRegistry` has no reset path and accumulates
duplicate entries across repeated scenario setup runs — harmless functionally, misleading
during verification. Resetting between runs means tearing down containers, deleting the
mission and team, and re-running setup exactly once. A missing cleanup path is a small
gap that costs time on every single test cycle, which is a reasonable argument for
building it.

---

## Known gaps

Listed because a documentation repo that only describes what works is marketing.

- **Self-heal error masking.** The escalation path's own failure replaces the original
  error in the reported message. Known, not fixed.
- **`all` logic ambiguity.** The same keyword means cumulative on a Gate and per-event on
  a Trigger. Worth a platform-level fix if the pattern spreads.
- **No `PersistenceRegistry` reset path.** Duplicates accumulate across setup runs.
- **Two mission execution paths.** The additive design was correct for compatibility but
  leaves `MissionPipeline` and `MissionTreeExecutor` both to maintain.
- **Decision node reliability with small local models.** Mitigated by preferring
  `approved_list`, not solved. A model below roughly 7B parameters should not be given a
  control-flow decision the mission blocks on.
- **Four scenario items with intentionally unfirable countermeasures.** Items whose
  removal deletes the very access their countermeasure needs. Correct by design — the
  countermeasure authenticates through the red team's own implant rather than a separate
  credential — but it means those items cannot retaliate, which is a design consequence
  rather than a bug and should be stated plainly.

---

## See also

- **[EXPLAINABILITY.md](EXPLAINABILITY.md)** — the logging layers and bounded model authority
- **[ARCHITECTURE.md](ARCHITECTURE.md)** — subsystems and the node graph
- **[MISSION_TREE.md](MISSION_TREE.md)** — the scenario engine these scenarios were built on
