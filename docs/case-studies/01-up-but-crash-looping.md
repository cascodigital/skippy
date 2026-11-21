# Case study 1: Up for four days, crashing all along

**Layer:** containers and automation. **Date:** November 2026.

## Context

A scrobbler container on the secondary host is the only thing that feeds the music listening history. The music
curation skill ([rolo](https://github.com/cascodigital/rolo)) and the kitchen TV playlists depend on that history.
It runs as a Node.js process with a container memory limit of 512 MB.

## What happened

For about four days (16 to 20 November) no new listens arrived. `docker ps` showed the container `Up` the whole
time. Nothing alerted.

Inside the container, the upstream streaming site had changed its web player. The YouTube client library logged
`Failed to extract signature decipher algorithm` and the Node process grew past its default heap (about 256 MB under
that container limit) and was killed, over and over, while the container itself never left the `Up` state.
Container state was "running". The service produced nothing.

This was the second silent failure of the same service. Earlier, an image update had made a config field mandatory;
without it the container stayed up and ran only its retention cleanup task. That one led to pinning the image
version.

## Fix

- Gave Node a real heap and the container room for it: `NODE_OPTIONS=--max-old-space-size=1024`, container limit
  1.5 GB. Measured peak after the change: about 460 MiB at startup.
- Kept the image pinned, so an automatic update cannot change the schema again.

## The watchdog, rewritten to measure output

An n8n workflow runs a watchdog script every 6 hours. It does not ask Docker whether the container is up. It
checks what the service is supposed to produce:

- the container exists and is running (`CONTAINER_DOWN` otherwise);
- the log has no config validation errors (`CONFIG_BROKEN`, the earlier failure);
- the log has no Node out-of-memory events in the last hour (`CRASH_LOOP`, the new one);
- the expected "polling started" line is in the recent log;
- authentication failures (`401`/`403` in a status field) are reported as `AUTH_BROKEN`, separately from the
  scrobbling service being down (`OK_LB_FORA`), which is not a local fault;
- the most recent listen recorded at the scrobbling service is recent enough (`STALE` otherwise).

The script always prints one line with a single status, which the n8n Code node validates.

Two bugs in the first version of the watchdog are worth recording. A bare `403` pattern matched the milliseconds of
a log timestamp, so the matcher now ignores debug lines and only counts status, code or error fields. And upstream
`Failed to validate token due to non-auth error` messages looked like an auth problem but were the remote API
returning 502.

## Lesson

**Container state is not service health.** A watchdog for a service should check the artifact the service
exists to produce, and name the failure mode it found (`CRASH_LOOP`, `AUTH_FAIL`, upstream down) instead of a
generic "unhealthy".
