# Layer 1: Hardware

**What it is.** Commodity and second-hand hardware, chosen for low power draw and for being replaceable.

| Role | Device | Notes |
|---|---|---|
| Main host | Orange Pi 5 (Rockchip RK3588, 8 cores ARM64, 8 GB RAM, NVMe) | Runs most containers and n8n. Container data lives on a 4 TB USB hard disk. Sits in the living room and also serves as that room's voice satellite. |
| Secondary host | Dell OptiPlex 7070 (Intel i5, 16 GB RAM, NVMe + internal SATA disk) | Runs the Home Assistant VM under KVM, DNS/DHCP, the AI console and the office voice satellite. Holds the media library and shares it to the main host over SMB. |
| Voice satellites | 2x eMeet OfficeCore M1A USB speakerphones | One per room. Hardware echo cancellation makes full-duplex conversation possible (talking over the assistant). |
| Radio | USB Zigbee coordinator | Passed through to the Home Assistant VM. |
| Workstation | Laptop (x86) | Edits the shared working tree; never the writer for git. |
| Clients | Android phone, Wear OS watch, Android TV boxes, Chromecast, network audio receiver | The phone and watch run the native voice client; the TVs run [skippy-tv](https://github.com/cascodigital/skippy-tv) and [kitchen-cast](https://github.com/cascodigital/kitchen-cast). |
| Travel | Raspberry Pi 3 | Plugged into any hotel or rental network, it opens a WireGuard tunnel home and publishes its own Wi-Fi. |

**Depends on.** Power and the ISP router. The secondary host sits on a smart plug that is kept out of Home
Assistant on purpose, because Home Assistant runs on that same host.

**What depends on it.** Everything. The split between an ARM board and an x86 box is visible in every layer
above: images must exist for both architectures, MCP servers are built per architecture, and some data formats
(PostgreSQL data directories) are not portable between them.

## Decisions

- **Two hosts, not one bigger one.** Splitting by role keeps the house (Home Assistant, DNS) on the machine with
  more RAM and fewer moving parts, and lets each host be the fallback for a service the other one runs.
- **USB speakerphones instead of DIY microphone arrays.** Conference speakerphones ship with echo cancellation
  and a noise gate tuned for speech. That removed the hardest part of building a voice satellite.

## Traps learned

- **A device that returns success can still be mute.** The speakerphone's ALSA playback volume came up at 0%
  after setup. `aplay` exited 0 and nothing came out. It is now the first thing checked when a room goes silent
  without errors.
- **`rms=0` in silence is the hardware noise gate, not a dead microphone.** Measured before chasing a hardware
  fault that did not exist.
- **The speakerphone's own volume buttons change a firmware gain Linux cannot see,** and it resets on USB
  re-enumeration. Volume is therefore enforced in ALSA by a small guard script, not left to the buttons.
- **One switch port negotiated only 100 Mb/s.** Under heavy traffic the secondary host dropped off the network.
  The NIC speed is now part of the host checklist.
- **Replugging the Zigbee dongle while the VM was running took the whole Zigbee network offline.** The
  persistent passthrough config matched the device by vendor and product ID, but libvirt resolves that to a bus
  address once, when the VM boots, and never again. A udev rule now triggers an idempotent detach, attach and
  Zigbee2MQTT restart whenever the dongle reappears (tested end to end: bridge back in about 30 s), and an
  automation alerts if the bridge stays offline for 5 minutes. Making the dongle mandatory for VM boot was
  rejected: it would trade "Zigbee down" for "the whole house down".
