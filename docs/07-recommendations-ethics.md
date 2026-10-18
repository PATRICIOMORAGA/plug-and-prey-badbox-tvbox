# 7. Recommendations and ethics

## Recommendations

**CSIRTs, regulators, and customs/market-surveillance teams.** Treat the device, not a single domain, as the indicator of compromise. Taking down one operator node does not remediate a box whose root, daemon, and client live in the shipped image. The useful lever is the hardware model and its firmware build, through import controls, recalls, and vendor accountability. The withheld reproduction material (keys, payloads, checksums) can be shared with competent authorities on request; do not re-fang the indicators to reconstruct targets from this document.

**Network defenders and ISPs.** Hunt the behavior described in chapter 6: an always-on consumer device that beacons on a fixed cadence to a hostname that is absent from the device's own storage, that answers an unknown binder service inside the system process, and whose home line shows many concurrent low-port connections consistent with proxy enrollment. An ISP seeing residential lines behaving as proxy exit nodes should treat that as a compromise signal, not as user traffic.

**Buyers and resellers.** A cheap no-name Android TV box that ships rooted and debuggable is not something a factory reset will clean. If a box must stay, isolate it on a guest VLAN with no access to the rest of the home network and no stored accounts; better, do not keep it on the network at all. Prefer devices from vendors that ship locked, signed, retail builds.

**Platform and ecosystem.** The on-device disabling of Play Protect and the silent-install plane show that a resident system component can neutralize the usual safety net. Detection has to assume the system image itself is hostile on these devices, rather than trusting on-device protection state.

## Method and ethics

- Original research by **Patricio Moraga** and **Anthony De La Paz**, prepared for a technical talk at **ekoparty 2026**. It is a continuation of, and companion to, the external-network findings presented in the paired deck.
- Collection combined on-device inspection over a serial console (unprivileged shell), rooted access over the local debugger, a `system_server` heap dump, static reverse engineering of the extracted dalvik-cache image, and a passive network capture.
- The full command-and-control cycle was reproduced only in a **sealed lab**: the operator hostname was pinned to localhost, the device had no route to the Internet, and a local server spoke only the client's own protocol. **The real operator server, and the live operator IP seen in the earlier capture, were never contacted.**
- The code executed by the replayed client was a **benign marker** we wrote; it contacts no server and only prints a log line and paints an on-screen banner.
- Network indicators are published **defanged** so defenders can hunt them. We do **not** publish the steps, keys, ciphertext, or payloads that would let a reader revive the operation or re-point the client. Key material was recovered to prove the obfuscation claim, and is withheld.
- No personal data of any device owner was collected. Device serial, MAC, and UUID are withheld.

Return to [the report index](../README.md).
