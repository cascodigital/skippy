# Case study 4: Nineteen hours deaf

**Layer:** voice. **Date:** August 2026.

## What happened

The office voice satellite stopped detecting its wake word. Nobody noticed for 19 hours; it was found by accident
when someone tried to use it.

Every health check looked fine:

- the systemd service was `active`;
- the socket to Home Assistant was `ESTABLISHED`;
- the satellite reported `idle` and the microphone was open;
- the capture device was recording real audio (peaks of about ±8000 measured);
- Home Assistant had even sent two `run-pipeline` requests during the window.

Zero detections in 19 hours. An earlier runbook said that an established socket meant the satellite was fine. This
incident proved that wrong, and the runbook was corrected.

## How it was diagnosed

Each suspect was ruled out with a measurement, not by assumption:

| Suspect | How it was ruled out |
|---|---|
| Wake word add-on | Healthy, model present, and **another room using the same add-on was detecting normally**. That one history query eliminated the server side. |
| Playback volume at 0 (a known trap) | Mixer value at 100, volume guard log empty |
| Microphone muted by the audio session manager | 3 s capture with real peaks |
| Regression in the service unit | `diff` against the four previous backups: only the expected two flags changed |

The decisive signal was free. A small feature picks a random "I'm listening" sound on every detection and keeps the
last three choices in a JSON file. That file held a single entry, and the sound file's modification time matched the
service start. So the satellite had not detected anything since it started, without relying on the journal at all.

A `stop` plus `start` of the satellite fixed it; it detected again two minutes later.

## What watches it now

- A **local watchdog** on the satellite host (every minute) restarts the satellite when it sees no open pipeline
  run for too long. Deafness of that kind is repaired in about 5 minutes without anyone noticing.
- The **deadman** from [case study 3](03-backup-deadman.md) checks that this local watchdog keeps rewriting its state
  file. If the file is older than 10 minutes or the host does not answer, a push goes out.

## Known gap, documented

The original failure mode (an **open** pipeline run with no audio reaching the wake word engine) still passes the
local watchdog as `OK`. The documentation says so explicitly instead of claiming full coverage. The earlier
6-hour "no usage" alert stays, described for what it is: an alert on lack of use, not proof that the satellite is
alive.

## Lessons

- **Liveness of the transport says nothing about the function.** Socket up, process up and microphone up are all
  inputs; the output is a detection.
- **Use the cheapest discriminating comparison first.** A second room using the same server cut the problem in
  half with one query.
- **Cheap side effects make good probes.** A file that changes on every detection is a free counter.
- **Write down the gap.** A watchdog that does not cover a known failure mode is fine, as long as nobody believes it
  does.
