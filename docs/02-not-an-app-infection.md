# 2. This is not a sideloaded-app infection

The mental model most people carry for Android malware is an app: the user installs something shady, it asks for permissions, and an antivirus can list and remove it. None of that applies here.

| Classic Android compromise | This box |
|----------------------------|----------|
| User sideloads or is tricked into installing an APK | Nothing is installed; the box ships compromised |
| Malware is an app with a package name | The C2 client runs inside `system_server`, loaded from a framework image, with no app entry |
| Root is gained via an exploit | Root is a factory daemon started by `init` at every boot |
| The C2 address is a string in the APK | The address is encrypted (Base64 → XOR → AES) and only exists in cleartext in memory at call time |
| A factory reset cleans it | The implant and root live in the system image and survive a data wipe |
| Play Protect warns on install | A resident component turns Play Protect off before it installs anything |

Three consequences follow, and they shape the whole investigation:

- **An app inventory is blind to it.** The implant is not in the installed-package list. The framework jar that carries it has been removed from `/system/framework`; only its compiled dalvik-cache image remains, mapped executable inside `system_server`. A search for the jar name on disk comes back empty, which is exactly why it first looked "fileless."
- **A string search for the C2 is blind to it.** The operator hostname is not written anywhere as text — not on disk, not in the process heap. It is produced by decrypting an embedded blob only when the client is about to connect. "Grep for the domain" returns zero, and that absence is itself a finding, not a dead end.
- **Reinstalling the OS does not help the owner.** Root, the daemon that starts it, and the implant are part of the shipped image. A factory reset re-writes that image. Removing the threat means removing the box from the network, not cleaning it.

This is the BadBox pattern: a supply-chain backdoor, present before first use, hiding as system infrastructure rather than as a user-visible app. The rest of this report is the evidence trail that establishes each of those claims on the physical device, and the controlled replay that proves the client still obeys its protocol.

Next: [The hunt](03-the-hunt.md).
