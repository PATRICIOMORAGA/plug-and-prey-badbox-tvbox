# 1. Executive summary

We examined a mass-market Allwinner Android TV box, sold as a streaming device, and found it **backdoored from the factory**. Nothing had to be installed, phished, or sideloaded. Out of the carton the box runs an engineering build (`eng` / `test-keys`) with the security switches off (`ro.secure=0`, `ro.debuggable=1`, `ro.adb.secure=0`), a persistent root daemon started by `init`, and a BadBox-class command-and-control client living inside the Android `system_server` process.

That client is not an app. It is loaded from a dalvik-cache image whose original framework jar has been deleted from `/system/framework`, so an app list does not show it and a filename search for the jar comes back empty. It beacons to a hard-coded operator address over HTTP, and the address is **not stored as text**: it is Base64, XOR-folded with a short key, then AES-encrypted, and only assembled in memory at call time. A plain search of disk or of the live heap for the hostname returns nothing.

The hunt had four beats:

1. **Entry.** The box ships rooted. The TCP debug port was closed, but the on-device debugger entered as `uid=0` with no prompt once the USB mode was switched, because the ROM security flags were already off.
2. **The client in memory.** We proved the implant is live without touching the operator: a system binder service answers a privileged call that only in-process implant code can serve, the implant classes are resident in the `system_server` heap, and the executable mapping traces back to a specific dalvik-cache image on disk.
3. **The address is encrypted, not absent.** The beacon hostname is three layers deep (Base64 → XOR → AES). We recovered the key material and the full URL from the mapped image. The cleartext hostname exists only while the code runs; it is on disk only as ciphertext.
4. **Replaying the whole cycle, safely.** In an isolated lab, with the operator domain pinned to localhost and no route to the Internet, the factory-installed client registered, requested a task, downloaded it, verified its checksum, decrypted it to the internal format, and executed the delivered code. The real operator server was never contacted.

Two more command-and-control planes share the same box: a live plugin/OTA chain that can silently install APKs and run shell scripts as root, and a **residential-proxy** payload that advertises the owner's home connection as an exit node (publisher tag `chch_gms`, ASN 7418 / Telefónica Chile, city Puente Alto, 66 proxy gateways). At one capture the proxy process held 25 concurrent connections, 15 of them to a single large provider on low ports.

In our observation window (early October 2026) the BadBox beacon was alive but its operator domain returned **NXDOMAIN** (DNS rcode 3): the client keeps calling a node that is currently down. A colleague's earlier capture (August 2026) showed the same name resolving to a live address with repeated connection attempts. The implant is therefore assessed **active and persistent**; the specific check-in node is **currently unreachable but trivially revivable**, since the address is baked into the firmware.

No operator identity was recovered. Network indicators are published in defanged form (chapter 6); cryptographic keys, decrypted payloads, checksums, and device identifiers are not.

Next: [This is not a sideloaded-app infection](02-not-an-app-infection.md).
