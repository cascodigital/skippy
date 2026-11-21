# Engineering principles

These rules are injected into every technical AI session that operates Skippy, and they apply to the human
operator too. Each one exists because something went wrong without it. The examples are real.

## Observability

**1. Alert on silence, not only on errors.**
A job that never starts produces no error. Every important periodic outcome publishes a success signal (a
heartbeat sensor, a rewritten state file, a finished container), and an hourly deadman alerts when the signal is
missing. The deadman sends a weekly "still alive" message, so its own death is visible too.
*Example: [five days without a backup](case-studies/03-backup-deadman.md).*

**2. "Running" is not "working". Check the artifact.**
The success criterion is the thing the user consumes (a file, a snapshot, a notification, a detection), never
"the process is up" or "the build passed".
*Examples: [Up but crash-looping](case-studies/01-up-but-crash-looping.md), [nineteen hours deaf](case-studies/04-deaf-satellite.md).*

**3. Silence is never success.**
A job whose normal result is "nothing found" ends with an explicit marker. An n8n SSH step does not fail on a
non-zero exit code, so every SSH step is followed by a check of the exit code or marker, and "no output" counts as
a failure. A lint over all workflows enforces this; zero findings is the expected result.

**4. The alert path is part of the system.**
Alerts go out over two independent channels (e-mail and a phone push addressed by IP, which works with DNS down).
A miss counts as delivered if either one works. Retrying a channel that shares a failure with the problem does not
help.

**5. A noisy job can blind the whole system.**
Shared limits (n8n's global execution history, rate limits, log retention) mean one workflow that fails in its
normal state can evict everyone else's evidence. Before activating a high-frequency job, decide what its normal
failure state is and exit cleanly in that state.

## Diagnosis

**6. Separate what was measured from what is a hypothesis.**
Correlation leads to the experiment that could refute it, not to a recommendation. A pool of 54 voice phrases was
almost pruned on a correlation from four samples; the experiment refuted it. Without the instrument (a log, a
capture, a stack sample), "no conclusion" is a better answer than a plausible culprit.

**7. Measure on the production path.**
Measure from where the code runs, not from the laptop. An SSH handshake took 3 s from the dual-homed laptop and
200 ms from the host where n8n actually runs. When a tool already has its own metric (n8n execution time, Home
Assistant traces, pipeline debug), read it before building a benchmark.

**8. Measure where you are before acting on the network.**
Agents can run in a container on either host or bare metal on the laptop, and each leaves the network from a
different address. The first step of any network task is `hostname`, a container check and `$SSH_CLIENT`. Assuming
the location once turned a correctly working source-restricted key into a "broken key" diagnosis.

**9. Live state beats documentation.**
Documentation gives the architecture and the traps; current state comes from the live source (Home Assistant
states, `docker ps`, the n8n API). When they disagree, the live source wins and the documentation is fixed on the
spot.

**10. Absence is not proven by a name pattern.**
"It does not exist" requires a search by identity (label, content, port, banner), not by a name that was guessed. If
the documentation says something exists and it cannot be found, the default answer is "I did not find it".

## Testing

**11. Test with a control.**
For a branch that *holds* state, "nothing changed" proves nothing. Run two phases that differ in exactly one
variable, with the condition actually provoked rather than faked. Test scripts restore what they touched in a
`trap ... EXIT`.

**12. Triggering by hand tests the actions, not the trigger.**
A watchdog is proven by waiting for the real condition and seeing `last_triggered` change on its own. Edge triggers
(`to: "on"` with `for:`) never fire if the entity is already `on` after a restart, so they get a periodic sweep as a
safety net. A harmless variant of an action does not prove the action: a camera PTZ "stop" returns success without
moving, so camera moves are verified by comparing two frames against a control.

**13. Ask a question that can discriminate.**
"Can you hear it?" turned into "validated" while the measured distortion was 75%. Ask "is it clean?", use a stimulus
that exposes the defect (a sweep, real speech, not a pure tone), and when measurement and perception disagree, the
measurement wins.

**14. Changing a shared source means testing its consumers.**
List every consumer of a sensor, flag or template before changing it, fire each one with a real trigger, and ask
"what if its input freezes or disappears?". An edited consumer that was never exercised left an outdoor light on all
night.

## Change safety

**15. Writes are dry-run by default.**
Every state-changing wrapper prints what it would do; `--apply` takes a backup, writes, re-reads, validates and
prints the rollback. Configuration that has its own validator (DNS, Home Assistant) is validated after the write and
restored automatically on failure.

**16. Safety must be structural, because long scripts get pasted, not read.**
Audit and action never live in the same script. Scripts that change state are short, dry-run by default, and their
gate variable is initialised to false on its own line so an exception cannot leave it true. The part a human must
read is printed last, next to the action. Irreversible operations go through the vendor's portal, not a script.

**17. A non-idempotent write succeeds by status code, not by a parseable body.**
No retry without a read proving the first attempt did not happen. A retry after a body that failed to parse once
posted a duplicate public comment; the duplicate was deleted immediately and the part that could not be undone was
reported.

**18. Before removing a service, ask who polls it.**
Pollers spend most of their time disconnected, so an empty connection list or access log proves nothing. Two
containers were removed on that evidence; one was the only data source for the memory watchdog, the other fed 64
Home Assistant sensors. Removal now starts with a search of the docs, the n8n workflow dump, Home Assistant
integrations and the crontab. When a device is retired, every layer is checked (Home Assistant, n8n, DHCP
reservation, watchdog scripts) and the clean layers are reported too.

**19. Do not edit a script while it runs.**
bash reads scripts by offset. Editing a running backup script caused a syntax error after the snapshot had already
been written. Edit a copy and `mv` it into place.

**20. Order boot dependencies explicitly.**
Docker that starts before its data disk, or a container that starts before a network share, fails once and stays
failed. Declare the dependency (`RequiresMountsFor=`, a wait unit) and add a guard that refuses to start on the wrong
filesystem. Failing to start is better than starting on an empty directory.

**21. Warn before restarting anything central.**
Edge devices (a camera, a lamp, a voice satellite) may be restarted directly. Home Assistant core, a host, Docker or
a central service only after a warning and an explicit yes. Hot reloads are preferred over restarts.

**22. If a rule keeps being broken, take the case out of reach.**
When a documented rule is violated repeatedly, the fix is to make the violation impossible (a deterministic check,
a guard script), not to rewrite the rule more forcefully. The authentication setup of the CLI agents was "fixed" by
editing shell rc files about fifteen times, one agent undoing the previous one, until a guard script took ownership
of it.
