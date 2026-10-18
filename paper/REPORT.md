# Plug and Prey: a factory-backdoored Android TV box and its BadBox C2

**Threat intelligence case study**
**TLP:WHITE** for this narrative · **Live infrastructure identifiers: withheld (TLP:AMBER)**

| | |
|--|--|
| **Authors** | Patricio Moraga · Anthony De La Paz |
| **Venue** | ekoparty 2026 |
| **Window** | 3–5 October 2026 (main on-device collection: 3 October 2026; lab replay: 5 October 2026; network capture: 4 October 2026) |
| **Status** | Implant assessed **active and persistent**; the BadBox check-in node was **NXDOMAIN** in our window but is baked into the firmware and trivially revivable |

**Abstract.** Original research into a mass-market Allwinner Android TV box that ships **already rooted and already implanted**. There is no infection step. The box runs an engineering build with its security flags off, a root daemon started by `init`, and a BadBox-class command-and-control client living inside `system_server`, loaded from a framework image whose source jar has been deleted. The client's operator address is not stored as text: it is Base64, XOR-folded, then AES-encrypted, and assembled only in memory at call time. This report tells the hunt — proving the client is alive, tracing it from disk to heap, decrypting the address, and replaying the whole command-and-control cycle in a sealed lab. Network indicators are published in **defanged** form so defenders can hunt them; cryptographic keys, decrypted payloads, checksums, and the lab device's own identifiers are withheld.

> **Handling.** Network indicators appear **defanged** (`ycxrl[.]com`, `hxxp://`, `85.17.155[.]53`), neutered so nothing here resolves, links, or executes; refang them only inside controlled defensive tooling. Nothing in this document is a key, a decrypted payload, or a reproduction guide.

---

## 1. Executive summary

We examined a mass-market Allwinner Android TV box, sold as a streaming device, and found it **backdoored from the factory**. Nothing had to be installed, phished, or sideloaded. Out of the carton the box runs an engineering build (`eng` / `test-keys`) with the security switches off (`ro.secure=0`, `ro.debuggable=1`, `ro.adb.secure=0`), a persistent root daemon started by `init`, and a BadBox-class command-and-control client living inside the Android `system_server` process.

That client is not an app. It is loaded from a dalvik-cache image whose original framework jar has been deleted from `/system/framework`, so an app list does not show it and a filename search for the jar comes back empty. It beacons to a hard-coded operator address over HTTP, and the address is **not stored as text**: it is Base64, XOR-folded with a short key, then AES-encrypted, and only assembled in memory at call time. A plain search of disk or of the live heap for the hostname returns nothing.

The hunt had four beats:

1. **Entry.** The box ships rooted. The TCP debug port was closed, but the on-device debugger entered as `uid=0` with no prompt once the USB mode was switched, because the ROM security flags were already off.
2. **The client in memory.** We proved the implant is live without touching the operator: a system binder service answers a privileged call that only in-process implant code can serve, the implant classes are resident in the `system_server` heap, and the executable mapping traces back to a specific dalvik-cache image on disk.
3. **The address is encrypted, not absent.** The beacon hostname is three layers deep (Base64 → XOR → AES). We recovered the key material and the full URL from the mapped image. The cleartext hostname exists only while the code runs.
4. **Replaying the whole cycle, safely.** In an isolated lab, with the operator domain pinned to localhost and no route to the Internet, the factory client registered, requested a task, downloaded it, verified its checksum, decrypted it, and executed the delivered code. The real operator server was never contacted.

Two more command-and-control planes share the box: a live plugin/OTA chain that can silently install APKs and run root shell scripts, and a **residential-proxy** payload that advertises the owner's home line as an exit node (publisher tag `chch_gms`, ASN 7418 / Telefónica Chile, city Puente Alto, 66 gateways; 25 concurrent connections at one reading, 15 to one large provider).

In our window the BadBox beacon was alive but its operator domain returned **NXDOMAIN**: the client keeps calling a node that is currently down. A colleague's August 2026 capture showed the same name resolving to a live address with repeated connection attempts. No operator identity was recovered. Network indicators are published here in defanged form; cryptographic keys, decrypted payloads, checksums, and device identifiers are not.

---

## 2. Why this is not a sideloaded-app infection

| Classic Android compromise | This box |
|----------------------------|----------|
| User sideloads or is tricked into installing an APK | Nothing is installed; the box ships compromised |
| Malware is an app with a package name | The C2 client runs inside `system_server`, loaded from a framework image, with no app entry |
| Root is gained via an exploit | Root is a factory daemon started by `init` at every boot |
| The C2 address is a string in the APK | The address is encrypted (Base64 → XOR → AES), cleartext only in memory at call time |
| A factory reset cleans it | The implant and root live in the system image and survive a data wipe |
| Play Protect warns on install | A resident component turns Play Protect off before it installs anything |

Three consequences shape the whole investigation: an app inventory is blind to the implant; a string search for the C2 returns zero (and that absence is a finding); and reinstalling the OS does not help the owner, because root, the daemon, and the client are part of the shipped image. This is the BadBox pattern — a supply-chain backdoor present before first use, hiding as system infrastructure rather than as a user-visible app.

---

## 3. The hunt

Dates are from the observation window in early October 2026 (device clock, matched to collection-host times). Full infrastructure identifiers appear defanged in chapter 8; device identifiers and cryptographic keys are omitted on purpose.

### 3.1 The box ships open

The firmware is an engineering build. The buyer is shown a current Android release, but the real platform level is several versions older — a mismatch between the marketing version in the beacon and the actual SDK level. Retail-locked properties are all off: `ro.secure=0`, `ro.debuggable=1`, `ro.adb.secure=0`, `test-keys`, build type `eng`.

Root is a service, not an exploit. Two root binaries sit in the system partition dated to the image build; an `init` service file launches the root daemon at boot; and the process list shows several daemon copies already running, parented to `init`. The TCP debug port (5555) was **closed**, matching the colleague's external view. Access came locally: switching the USB mode to expose the debug interface let the on-device debugger return `uid=0` in the root SELinux context, with no on-screen prompt, because the security flags were already zero.

### 3.2 Proving the client is alive without touching the operator

Three independent facts, each recorded on the device:

- **A privileged binder service answers.** The system service list exposes a plugin-manager service whose AIDL interface belongs to the implant package. Its "list process names" method returns the live system process list. Only implant code executing inside `system_server` can serve that call.
- **The classes are resident in the heap.** A `system_server` heap dump (~22.2 MB) contains the implant's class names — the registration state machine, network/beacon helpers, Base64/DES/XOR utilities, the plugin-manager stub — verified by matching the on-device hash to the copy pulled to the analysis host.
- **The executable mapping traces to disk.** With root over the debugger, the full memory map of `system_server` shows read-only and executable mappings of a specific dalvik-cache image. The original framework jar is gone from `/system/framework`, but its optimized OAT (an ELF of ~565 KB, carrying an embedded DEX of ~177 KB and ~103 implant-namespace classes) is still present and still mapped executable.

That closes a chain: **OAT on disk → executable mapping in `system_server` → classes in the heap → binder service answering**. An earlier `grep` for the jar name found nothing because the on-disk path uses a framework-looking name, not the implant's package name.

### 3.3 The address is encrypted, not missing

A string search for the operator hostname — on disk and in the heap — returns nothing. The heap "URLs" extract came back empty, confirming the address is not held in cleartext in memory. The hostname is three layers deep inside the mapped image:

1. The bytes in the dump are **Base64** (the outer layer).
2. The client Base64-decodes them and applies a repeating-key **XOR**. The XOR key is a short literal present in cleartext in one of the implant's utility classes, in the same mapped image. The result is still not the address: it is another Base64 string.
3. That second Base64 is the **AES** ciphertext. The AES key is itself a Base64 blob in a different implant class, run through the same XOR step to yield a 16-byte key. Decrypting the second blob with that key produces the whole URL at once.

We recovered the key material and reproduced the full URL offline. The structural point: the cleartext address exists only while the client runs. On disk it is only ciphertext, which is why static search fails and memory forensics plus key recovery were required.

### 3.4 Replaying the whole cycle in a sealed lab

To show the factory client still obeys its protocol without contacting the operator, we built an isolated access point and pinned the operator hostname to localhost on the device (a `hosts` entry plus a reverse port bridge to a local server that speaks only the client's protocol). No route to the Internet existed; the real operator address was never contacted.

On replay, in one tight sequence (device clock, around 00:02:44 on the lab day):

1. The client sent its registration beacon — an HTTP POST to the implant's check-in path, ~379 bytes of encrypted form data, with the device's Dalvik user agent. The lab answered with the "registration accepted" status the client expects.
2. With registration accepted, the client's state machine requested a task.
3. It downloaded the task archive (~1 KB) over HTTP.
4. It reported back: the target package was present, and the file's checksum matched.
5. It decrypted the archive to its internal on-disk format (a task descriptor and a DEX, each re-encrypted on disk), loaded the delivered class, and executed its entry method.

The delivered code was ours: a benign marker that prints a known log line and paints an on-screen banner. Its appearance on the TV is proof that the factory client accepted a task from a server it believed was the operator and executed it. The marker talks to no server.

### 3.5 The other two C2 planes

- **A live plugin/OTA chain.** A content provider on another resident app rewrites its base URL to a live operator API at startup, so editing stored preferences alone does not redirect it. That API delivers a plugin descriptor (download URL, checksum, intervals, trigger packages) and can **silently install APKs**. A separate factory OTA endpoint accepts a special script package — a ZIP whose command file is decrypted line by line and run through `sh` as root. Reach: **any APK, firmware, or root script**. In our window one of these components installed a monitoring package during the same afternoon the box was read.
- **A residential proxy.** A resident payload enrolls the home line as a sellable exit node. Its stored configuration carries a publisher tag (`chch_gms`), ASN 7418 (Telefónica Chile), country `CL`, city `Puente Alto`, and **66 proxy gateways**. At one reading the proxy process held 25 concurrent outbound connections, 15 to a single large provider on low ports. Two other commercial proxy/bandwidth SDKs are present on the same image.

### 3.6 The node is down, the implant is not

In our capture the BadBox beacon kept firing, but DNS resolution of its operator hostname returned **NXDOMAIN (rcode 3)**, repeatedly. The client is calling a node that is off the air. The colleague's August capture showed the same hostname resolving to a live address with repeated connection attempts on a few-minute cadence. Because the address is compiled into the firmware, reviving the operation requires only that the operator bring a server back up on that name; the implant does not change.

---

## 4. Attack chain

```
Factory image (eng / test-keys, security flags off)
  → root daemon started by init at every boot
  → BadBox client resident in system_server (loaded from a framework dalvik-cache image)
  → encrypted beacon (Base64 → XOR → AES) to the operator host over HTTP
     ├─ register  → task  → download  → verify checksum  → decrypt  → execute delivered code
     ├─ plugin/OTA plane: silent APK install; root shell scripts via OTA command package
     └─ residential-proxy payload: home line advertised as an exit node
```

Three independent planes share one device. The streaming launcher the buyer sees is the cover; the business starts before any user action, in the system image. Taking down one operator node (as happened to the BadBox name in our window) removes neither the client, the root, nor the other two planes. A factory reset re-writes the compromised image.

---

## 5. Who is exposed

This case has no chat logs and no individual victims to quote. The affected party is the household. Where a statement is an inference rather than a recorded fact, it is marked.

**The home line becomes a proxy exit node.** The residential-proxy payload advertises the connection to a commercial proxy network, tagged to the owner's ISP (ASN 7418) and city (Puente Alto), with 66 gateways configured. Whatever a third party routes through that exit leaves from the buyer's IP. In one reading the proxy process held 25 simultaneous outbound connections, 15 to one large provider on low ports. This is the plane behind the large background traffic volumes reported in the companion talk.

**Anything on the network is reachable from a rooted box.** The box runs as root and executes code handed to it by a server. We did **not** observe LAN pivoting or credential theft in our window (inferred capability, not confirmed activity), but the primitive — arbitrary code execution as root on an always-on box inside the home — is present and demonstrated.

**Accounts opened on or through the box inherit the risk.** Sessions on the device live on a rooted system that runs attacker-delivered tasks. We did not recover account data and did not see credential capture (inferred risk, not confirmed activity). The assumed trust boundary — "it's just my TV box" — does not exist.

**Protection software is actively disabled.** Before installing, a resident component turns Play Protect off (both the on-device verifier and the upload-to-vendor check) and logs its own success.

**A reset does not fix it.** Root, the daemon, and the implant are in the shipped image; a factory reset re-writes them. For the owner, the only effective action is to take the box off the network.

---

## 6. Sampling and scope note

This is a **single-device case study**, not a fleet survey. All on-device findings come from one physical unit of one model. The network capture is a short passive window (about nine minutes on 4 October 2026) and shows the beacon behavior and NXDOMAIN responses, not a full traffic census. The earlier live-resolution data point is from a colleague's separate August 2026 capture and is cited as corroboration, not re-measured here. Figures such as "66 gateways," "25 concurrent connections / 15 to one provider," and the ~22.2 MB heap size are exact readings from this unit at the moment of collection; they describe this device and should not be generalized to all units of the model without further sampling.

---

## 7. MITRE ATT&CK

| ID | Technique | In this case |
|----|-----------|----------------|
| T1195.003 | Supply chain compromise: hardware | Backdoor present in the factory image before first use |
| T1601.001 | Modify system image | Root daemon, removed framework jar, implant in dalvik-cache |
| T1543 | Create or modify system process | `init` service starts the root daemon at boot |
| T1055 | Process injection / in-process execution | Implant runs inside `system_server` |
| T1140 | Deobfuscate/decode information | Beacon hostname Base64 → XOR → AES, assembled in memory |
| T1027 | Obfuscated/encrypted information | C2 address stored only as ciphertext on disk |
| T1071.001 | Web protocols | HTTP register/task/report cycle |
| T1105 | Ingress tool transfer | Task archive downloaded, verified, decrypted, executed |
| T1554 | Compromise host software binary | Silent APK install via the plugin/OTA plane |
| T1059 | Command and scripting interpreter | OTA command package run via `sh` as root |
| T1562.001 | Impair defenses | Resident component disables Play Protect before installing |
| T1090 | Proxy | Residential-proxy payload; home line as exit node |
| T1657 | Financial gain | Proxy/bandwidth monetization of the owner's connection |

---

## 8. Indicators

Network indicators are **defanged**: neutered so nothing here resolves, links, or executes. Refang them only inside controlled defensive tooling.

**Network (defanged):**

| Plane | Indicator | Notes |
|-------|-----------|-------|
| BadBox check-in | `ycxrl[.]com` | paths `/terminal/client/eventinfo`, `/terminal/client/register`, `/terminal/client/apiInfo`, `/terminal/client/plugreport`; Dalvik user agent; channel `A10501` |
| BadBox resolved address | `85.17.155[.]53` | from the Aug 2026 capture (SYN :80, ~6–7 min cadence); **NXDOMAIN** in our Oct window |
| Plugin / ad API (live) | `api.fivira[.]cc`, backup `api.firico[.]cc` | path `/appapi/ad/sdk`; channel `CHH5005`; silent-install plane |
| Plugin CDN | `xfile.kkoper[.]cc` | plugin ZIP delivery |
| Factory OTA | `hxxp://uota.uota[.]xyz:2082` | path `/api/common/ota/update/v1`; root script/APK delivery |
| netservice | `business.dnsyc[.]top`, `heartbeat.dnsyc[.]top` | image-planted installer chain |
| Residential proxy | `nice-protect[.]com` | publisher tag `chch_gms`; ASN 7418 (Telefónica Chile); gateways `sNNN:6000` (66 configured) |
| Bandwidth-sell SDK | `wss://thewsoral[.]com` | exit-node monetization |

**Host / implant fingerprint:**

- Implant namespaces `com.jar` / `com.log.sloth`; carrier framework `xlogd.jar`; dalvik-cache `system@framework@xlogd.jar@classes.dex` with the **source jar absent** from `/system/framework`.
- Binder service `PlugManager`, AIDL `com.jar.aidl.IPlugManager`, answering a "list process names" call from inside `system_server`.
- Stage directory `/data/system/Corejava/` (`e.l`, `node/`); prefs `/data/system/shared_prefs/open_preference.xml` (`slothRegisterStatus`).
- ROM: `eng` / `test-keys`, `ro.secure=0`, `ro.debuggable=1`, `ro.adb.secure=0`; root daemon started by `init` via `sudaemon.rc`.

**Behavioral hunts (no locator needed):**

- An Android TV box that **ships** rooted, debuggable, and `test-keys`, with daemon copies parented to `init`.
- A marketing OS version that does not match the real platform SDK level in the device's own beacon.
- A periodic HTTP beacon whose destination hostname is **not** a string on disk or in the heap.
- Outbound DNS for one hostname returning **NXDOMAIN** on a fixed cadence.
- A process holding **many concurrent low-port connections** to one provider, with stored proxy config tagging a residential ISP and city.
- Play Protect verifier/upload settings flipped to `0` by on-device code.

**Withheld (reproduction material / PII, not locators):** the XOR / AES / DES key material, the decrypted payloads and task archive, payload checksums, and the lab device's own serial / MAC / UUID. These can be shared with competent authorities on request.

---

## 9. What we did not get

| Question | Result |
|----------|--------|
| The real plugin payload the operator would serve | No — the check-in node was NXDOMAIN; we replayed with a benign lab task only |
| Byte-for-byte dump of the anonymous DEX region in RAM | No — `/proc/<pid>/mem`, `maps`, `fd` were `Permission denied` in the serial window; the on-device root tap was pending |
| Cleartext C2 URL in the heap | No — the heap holds the classes, not the address; it is produced only at call time |
| FEL / low-level flash dump | No — FEL mode enumerated, but the dump was blocked by USB-permission constraints in the VM |
| Operator civil identity | No |
| Confirmed LAN pivot or credential theft in our window | No — capability demonstrated (root + RCE); activity not observed |
| A product CVE matching these behaviors | No — this is a supply-chain backdoor and an insecure-by-default build, not a numbered app vulnerability |
| Contact with the real operator server | No — deliberately never contacted |

---

## 10. Recommendations

**CSIRTs, regulators, and market-surveillance teams.** Treat the device, not a single domain, as the indicator. Taking down one operator node does not remediate a box whose root, daemon, and client live in the shipped image. The lever is the hardware model and its firmware build — import controls, recalls, vendor accountability. The withheld reproduction material (keys, payloads, checksums) is available to competent authorities on request.

**Network defenders and ISPs.** Hunt the behavior in chapter 8: an always-on consumer device beaconing on a fixed cadence to a hostname absent from its own storage, answering an unknown binder service inside the system process, with a home line showing proxy-enrollment traffic. Residential lines behaving as proxy exit nodes are a compromise signal, not user traffic.

**Buyers and resellers.** A cheap no-name Android TV box that ships rooted and debuggable is not something a factory reset will clean. If a box must stay, isolate it on a guest VLAN with no access to the rest of the home network and no stored accounts; better, keep it off the network. Prefer vendors that ship locked, signed, retail builds.

**Platform and ecosystem.** On-device disabling of Play Protect and the silent-install plane show a resident system component can neutralize the usual safety net. Detection must assume the system image itself is hostile on these devices.

---

## 11. Method and ethics

- Original research by **Patricio Moraga** and **Anthony De La Paz**, for a technical talk at **ekoparty 2026**, companion to the external-network findings in the paired deck.
- Collection combined on-device inspection over a serial console (unprivileged shell), rooted access over the local debugger, a `system_server` heap dump, static reverse engineering of the extracted dalvik-cache image, and a passive network capture.
- The full command-and-control cycle was reproduced only in a **sealed lab**: the operator hostname pinned to localhost, no route to the Internet, a local server speaking only the client's protocol. **The real operator server, and the live operator IP from the earlier capture, were never contacted.**
- The code executed by the replayed client was a **benign marker** we wrote; it contacts no server and only prints a log line and paints a banner.
- We do **not** publish the steps, keys, ciphertext, or payloads that would revive the operation or re-point the client. Key material was recovered to prove the obfuscation claim and is withheld.
- No personal data of any device owner was collected. Device serial, MAC, and UUID are withheld.

---

## 12. Timeline and publication

| Date (2026) | Event |
|-------------|-------|
| August | Colleague's external capture: the BadBox hostname resolves live; repeated connection attempts on a few-minute cadence |
| 3 Oct | Main on-device collection: serial console, rooted debugger access, `system_server` memory map and heap dump, dalvik-cache image extracted; a monitoring package observed installed by the plugin/OTA plane the same afternoon |
| 4 Oct | Passive network capture (~9 min): BadBox beacon active, operator hostname returns NXDOMAIN |
| 5 Oct | Sealed-lab replay of the full register → task → download → verify → decrypt → execute cycle; benign marker executed on the device |
| On release | Published openly so authorities, defenders, and the public can review it; network indicators published defanged, while keys, payloads, checksums, and device identifiers are withheld and available to competent authorities on request |

---

## 13. References

- MITRE ATT&CK, Enterprise matrix and technique catalogue. <https://attack.mitre.org>
- CISA / FBI and international reporting on the **BadBox** supply-chain compromise of Android TV boxes. <https://www.cisa.gov>
- Google Threat Analysis Group and partner reporting on pre-installed Android TV backdoors (BadBox / Peachpit family). <https://blog.google/threat-analysis-group/>
- Android Open Source Project, `system_server`, `init`, and verified-boot documentation. <https://source.android.com/docs/security>
- Allwinner / Android TV FEL (firmware-exfiltration/boot) mode background. <https://linux-sunxi.org/FEL>
- OWASP Mobile Application Security guidance (obfuscation, insecure defaults). <https://owasp.org>

---

## Citation

Moraga, P., & De La Paz, A. (2026). *Plug and Prey: a factory-backdoored Android TV box and its BadBox C2.* Threat intelligence case study, ekoparty 2026. Public edition publishes network indicators defanged and withholds keys, payloads, and device identifiers while the threat persists.

---

*Document version 1.0 (2026-10). One file, written to be published as-is. If you mirror it, do not re-fang the indicators or add keys, decrypted payloads, checksums, or device identifiers.*
