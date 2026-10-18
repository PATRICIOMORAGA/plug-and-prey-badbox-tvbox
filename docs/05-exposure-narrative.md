# 5. Who is exposed

This case has no chat logs and no individual victims to quote. The affected party is the household — anyone who buys the box, plugs it into the home network, and uses it. The exposure is concrete and follows from what the device does, not from speculation. Where a statement is an inference rather than a recorded fact, it is marked.

**The home line becomes a proxy exit node.** The residential-proxy payload advertises the connection to a commercial proxy network, tagged to the owner's ISP (ASN 7418, Telefónica Chile) and city (Puente Alto), with 66 gateways configured. Whatever a third party routes through that exit — including activity the buyer would never choose — leaves from the buyer's IP address. In one reading the proxy process held 25 simultaneous outbound connections, 15 of them to a single large provider on low ports. This is the plane that produced the large background traffic volumes reported in the companion talk.

**Anything on the network is reachable from a rooted box.** The box runs as root and executes code handed to it by a server. A device with full privileges, sitting inside the LAN, is a foothold: it can see and reach other devices on the same network. We did **not** observe LAN pivoting or credential theft in our window (inferred capability, not confirmed activity), but the primitive — arbitrary code execution as root on a box that is always on and inside the home — is present and demonstrated.

**Accounts opened on or through the box inherit the risk.** If the owner signs into services on the device, those sessions live on a system that is rooted and that runs attacker-delivered tasks. We did not recover any account data and did not see credential capture (inferred risk, not confirmed activity); the point is that the trust boundary the user assumes — "it's just my TV box" — does not exist.

**Protection software is actively disabled.** Before installing, a resident component turns Play Protect off (both the on-device verifier and the upload-to-vendor check), and logs its own success in its code. So the usual safety net that might flag a silently installed APK is removed by the box itself.

**A reset does not fix it.** Root, the daemon, and the implant are in the shipped image. A factory reset re-writes them. For the owner, the only effective action is to take the box off the network.

The through-line: the buyer paid for a streaming device and received a general-purpose, always-on, rooted node inside their home that answers to three different remote planes. The harm does not require a targeted attack on the individual; it is the default state of the product.

Next: [ATT&CK, indicators, limits](06-attack-indicators-limits.md).
