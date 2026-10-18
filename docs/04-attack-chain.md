# 4. Attack chain

```
Factory image (eng / test-keys, security flags off)
  → root daemon started by init at every boot
  → BadBox client resident in system_server (loaded from a framework dalvik-cache image)
  → encrypted beacon (Base64 → XOR → AES) to the operator host over HTTP
     ├─ register  → task  → download  → verify checksum  → decrypt  → execute delivered code
     ├─ plugin/OTA plane: silent APK install; root shell scripts via OTA command package
     └─ residential-proxy payload: home line advertised as an exit node
```

Three independent command-and-control planes share one device, and they do not depend on each other:

1. **BadBox client** (`system_server` resident). Register → request task → download → checksum → decrypt → run. Delivers and executes arbitrary code inside the system process. Its check-in node was down in our window; the client keeps beaconing.
2. **Plugin / OTA plane** (live). A content provider pins a live API that hands out plugins and can silently install APKs; a factory OTA endpoint runs root shell scripts delivered as an encrypted command package. Reach: anything, as root.
3. **Residential proxy** (live). The owner's connection is enrolled as an exit node, tagged to a Chilean ISP and city, with dozens of gateways and many concurrent connections.

The device the buyer sees — a streaming launcher — is the cover. The business sits underneath it, in the system image, and starts before any user action. Taking down one operator node (as happened to the BadBox name in our window) does not remove the client, the root, or the other two planes. Removing the threat means removing the box from the network; a factory reset re-writes the compromised image.

Next: [Who is exposed](05-exposure-narrative.md).
