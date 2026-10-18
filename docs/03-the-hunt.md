# 3. The hunt

Dates below are from the observation window in early October 2026 (device clock, matched to collection host times). Full infrastructure identifiers appear defanged in chapter 6; device identifiers and cryptographic keys are omitted on purpose.

## 3.1 The box ships open

The firmware is an engineering build. The build is advertised to the buyer as a current Android release, but the real platform level is several versions older — a mismatch between the marketing version in the beacon and the actual SDK level. The security properties that a retail device sets to "locked" are all off:

- `ro.secure = 0`
- `ro.debuggable = 1`
- `ro.adb.secure = 0`
- build tags: `test-keys`; build type: `eng`

Root is not an exploit; it is a service. Two root binaries sit in the system partition dated to the image build, an `init` service file launches the root daemon at boot, and the process list shows several copies of the daemon already running with `init` as their parent. The TCP debug port (5555) was **closed**, matching the colleague's earlier external view. Access came locally: switching the USB mode to expose the debug interface let the on-device debugger return `uid=0` in the root SELinux context, with no on-screen authorization prompt, because `ro.secure` and `ro.adb.secure` were already zero.

## 3.2 Proving the client is alive without touching the operator

Before decrypting anything, we established that the implant is a running thing, not a dormant file. Three independent facts, each recorded on the device:

- **A privileged binder service answers.** The system service list exposes a plugin-manager service whose AIDL interface belongs to the implant package. Calling its "list process names" method returns the live system process list. Only implant code executing **inside** `system_server` can serve that call.
- **The classes are resident in the heap.** A heap dump of `system_server` (about 22.2 MB) contains the implant's class names — the registration state machine, the network/beacon helpers, the Base64/DES/XOR utility classes, the plugin-manager stub. The class list was verified by matching the on-device hash to the copy pulled to the analysis host.
- **The executable mapping traces to disk.** With root over the debugger, the full memory map of `system_server` shows a read-only and an executable mapping of a specific dalvik-cache image. The original framework jar for that image is gone from `/system/framework`, but its optimized OAT (an ELF of ~565 KB, carrying an embedded DEX of ~177 KB and ~103 classes in the implant's namespaces) is still present in dalvik-cache and still mapped executable.

That closes a chain: **OAT on disk → executable mapping in `system_server` → classes in the heap → binder service answering**. The reason an earlier `grep` for the jar name found nothing is that the on-disk path uses a framework-looking name, not the implant's package name.

## 3.3 The address is encrypted, not missing

A string search for the operator hostname — on disk and in the heap — returns nothing. The heap "URLs" extract came back empty (an empty-file hash), confirming the address is not held in cleartext in memory either.

The hostname is three layers deep inside the mapped image:

1. The bytes visible in the dump are **Base64**. That is the outer layer.
2. The client Base64-decodes them and applies a repeating-key **XOR**. The XOR key is a short literal that is present in cleartext in one of the implant's utility classes, in the same mapped image. The result is still not the address: it is another Base64 string.
3. That second Base64 is the **AES** ciphertext. The AES key is itself stored as a Base64 blob in a different implant class, run through the same XOR step to yield a 16-byte key. Decrypting the second blob with that key produces the whole URL at once.

We recovered the key material and reproduced the full URL from the mapped image offline. The point for defenders is structural: the cleartext address exists only while the client runs. On disk it is only ciphertext, which is why static search fails and why memory forensics plus key recovery were necessary.

## 3.4 Replaying the whole cycle in a sealed lab

To show the factory client still obeys its protocol — without ever contacting the operator — we built an isolated access point and pinned the operator hostname to localhost on the device (`hosts` entry plus a reverse port bridge to a local server that speaks only the client's own protocol). There is no route to the Internet in this setup, and the real operator address was never contacted.

On replay, in one tight sequence (device clock, around 00:02:44 on the lab day):

1. The client sent its registration beacon: an HTTP POST to the implant's check-in path, ~379 bytes of encrypted form data, with the device's own Dalvik user agent. The lab answered with the "registration accepted" status the client expects.
2. With registration accepted, the client's state machine asked for a task on the task path.
3. It downloaded the task archive (~1 KB) over HTTP.
4. It reported back on the report path: the target package was present, and the downloaded file's checksum matched.
5. The client decrypted the archive to its internal on-disk format (a task descriptor and a DEX, each re-encrypted on disk), loaded the delivered class, and executed its entry method.

The delivered code was ours: a benign marker that prints a known log line and paints an on-screen banner. Its appearance on the TV is the proof that the factory-installed client accepted a task from a server it believed was the operator and executed it. The marker talks to no server and only draws the overlay.

## 3.5 The other two C2 planes

The BadBox client is not the only thing calling home.

- **A live plugin/OTA chain.** A content provider on another resident app rewrites its base URL to a live operator API at startup (so editing stored preferences alone does not redirect it). That API delivers a plugin descriptor with a download URL, checksum, intervals, and trigger package list, and the plugin can **silently install APKs**. A separate factory OTA endpoint accepts a special script package: a ZIP whose command file is decrypted line by line and run through `sh` as root. The reach of that plane is **any APK, firmware, or root script** on the box. In our window one of these components installed a monitoring package on the device, time-stamped during the same afternoon the box was being read.
- **A residential proxy.** A resident payload turns the home connection into a sellable exit node. Its stored configuration carries a publisher tag (`chch_gms`), ASN 7418 (Telefónica Chile), country `CL`, city `Puente Alto`, and a list of **66 proxy gateways**. At one reading the proxy process held 25 concurrent outbound connections, 15 of them to a single large provider on low ports. Two other commercial proxy/bandwidth SDKs are present on the same image.

## 3.6 The node is down, the implant is not

In our capture the BadBox beacon kept firing, but DNS resolution of its operator hostname returned **NXDOMAIN (rcode 3)**, repeatedly, across the capture. The client is calling a node that is currently off the air. A colleague's earlier capture (August 2026) showed the same hostname resolving to a live address, with repeated connection attempts on a few-minute cadence. Because the address is compiled into the firmware, reviving the operation requires only that the operator bring a server back up on that name. The implant does not need to change.

Next: [Attack chain](04-attack-chain.md).
