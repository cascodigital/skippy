# Case study 2: 68 seconds to press play

**Layer:** hosts and network, surfacing in voice and automation. **Date:** November 2026.

## Symptoms

Over two days, two unrelated-looking problems:

- A voice session hung silently inside the real-time model's connect call. No error, no log, the user heard nothing.
- Starting a playlist through the music server took about 70 seconds, and sometimes failed with
  `Timed out acquiring playback lock` and an AirPlay session that nobody claimed.

The obvious suspect was the music server. It was updated the same day. That changed nothing, and it was not the
cause.

## Measuring instead of guessing

The music server runs inside the Home Assistant OS VM. Its stuck threads were sampled from a sidecar container that
joined the add-on's PID namespace with `SYS_PTRACE`. Every stuck thread was inside urllib3's `create_connection`.

urllib3 (used by `requests`, `ytmusicapi` and `yt-dlp`) tries each address of a name in order and has no Happy
Eyeballs. Each request tried the IPv6 address first and waited for it to fail before falling back to IPv4.

A ten-second probe on the affected host confirmed it: for each global IPv6 address on the interface, `curl -6
--interface <addr>` either answered or timed out. One prefix was dead.

## Root cause

The ISP router had rebooted the day before the first symptom and received a new IPv6 prefix. It kept advertising
the old one as well, through Router Advertisements and a DHCPv6 lease valid for about five days, with no route
behind it. Every host on the LAN picked up addresses on both prefixes. Clients that try addresses in order paid a timeout on every new
connection.

## Fixes that did not hold

- Deleting the dead address on a host: the router put it back within 30 seconds.
- Preferring IPv4 per host through `gai.conf`: it worked on the two servers, but containers inside Home Assistant OS
  do not inherit the host's `gai.conf`, so the music server was not covered.

## Fix at the source

Router Advertisements and DHCPv6 were turned off on the ISP router. The router's write API is encrypted in its web
UI JavaScript, so the change is made by a headless browser script, with a matching rollback script and a
read-only check. A daily n8n job measures from the main host that the global IPv6 route has not come back.

Without a default IPv6 route, a global IPv6 attempt now fails immediately with `ENETUNREACH`, and glibc sorts IPv4
first (verified inside the music server's container). Local IPv6 is kept: the DNS host still advertises a unique
local prefix for Matter devices.

**Result: starting a playlist went from 68 s to 6 s.** The silent voice hang was the same cause.

## Lessons

- **Sample the stuck process before blaming a component.** The stack trace pointed at the network library in one
  measurement; updating the suspect would have proven nothing.
- **A fix that has to be applied per host is a workaround.** The real owner of the problem was the device announcing
  the bad prefix.
- **Turning something off is a decision to document.** The router setting, the reason, the values before the change
  and the rollback command are recorded together, so nobody turns IPv6 back on "to have IPv6".
