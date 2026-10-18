# 6. ATT&CK, indicators, limits

## MITRE ATT&CK

| ID | Technique | In this case |
|----|-----------|----------------|
| T1195.003 | Supply chain compromise: compromise hardware supply chain | Backdoor present in the factory image before first use |
| T1601.001 | Modify system image | Root daemon, removed framework jar, implant in dalvik-cache |
| T1543 | Create or modify system process | `init` service starts the root daemon at boot |
| T1055 | Process injection / in-process execution | Implant runs inside `system_server`, loaded from a framework image |
| T1140 | Deobfuscate/decode information | Beacon hostname is Base64 → XOR → AES, assembled in memory |
| T1027 | Obfuscated/encrypted files and information | C2 address stored only as ciphertext on disk |
| T1071.001 | Application layer protocol: web | HTTP register/task/report cycle |
| T1105 | Ingress tool transfer | Task archive downloaded, checksum-verified, decrypted, executed |
| T1554 | Compromise host software binary | Silent APK install via the plugin/OTA plane |
| T1059 | Command and scripting interpreter | OTA command package: lines decrypted and run via `sh` as root |
| T1562.001 | Impair defenses: disable or modify tools | Resident component turns Play Protect off before installing |
| T1090 | Proxy | Residential-proxy payload; home line as an exit node |
| T1657 | Financial gain | Proxy/bandwidth monetization of the owner's connection |

## Indicators

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
- A marketing OS version that does **not** match the real platform SDK level in the device's own beacon.
- A periodic HTTP beacon whose destination hostname is **not** present as a string anywhere on disk or in the heap.
- Outbound DNS for one hostname returning **NXDOMAIN** on a fixed cadence (a client calling a dead node).
- A process holding **many concurrent low-port connections** to one provider, with stored proxy config tagging a residential ISP and city.
- Play Protect verifier and upload settings flipped to `0` by on-device code.

**Withheld (reproduction material / PII, not locators):** the XOR / AES / DES key material, the decrypted payloads and task archive, payload checksums, and the lab device's own serial / MAC / UUID. These can be shared with competent authorities on request.

## What we did not get

| Question | Result |
|----------|--------|
| The real plugin payload the operator would serve | No — the check-in node was NXDOMAIN in our window; we replayed the protocol with a benign lab task only |
| Byte-for-byte dump of the anonymous DEX region in RAM | No — `/proc/<pid>/mem`, `maps`, `fd` were `Permission denied` without root in the serial window; the on-device root tap was pending |
| Cleartext C2 URL in the heap | No — the heap holds the classes, not the address (empty "URLs" extract); the address is produced only at call time |
| FEL / low-level flash dump | No — FEL mode enumerated, but the dump was blocked by USB-permission constraints in the VM |
| Operator civil identity | No |
| Confirmed LAN pivot or credential theft in our window | No — capability is present and demonstrated (root + remote code execution); we did not observe the activity |
| A product CVE matching these behaviors | No — this is a supply-chain backdoor and a class of insecure-by-default build, not a numbered vulnerability in a specific app |
| Contact with the real operator server | No — deliberately never contacted |

Next: [Recommendations and ethics](07-recommendations-ethics.md).
