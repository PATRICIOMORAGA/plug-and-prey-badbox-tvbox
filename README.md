# Plug and Prey: a factory-backdoored Android TV box and its BadBox C2

**Threat intelligence case study** · ekoparty 2026
**Authors:** Patricio Moraga · Anthony De La Paz
**TLP:WHITE** (this text) · **Indicators defanged; keys, payloads, and device identifiers withheld**

Original research into an off-the-shelf Allwinner Android TV box that ships **already rooted and already implanted**. There is no infection step: an engineering build, a factory root daemon, and a BadBox-class client living inside `system_server` are present the moment the box leaves the carton. This report tells the full hunt — pulling the client out of memory, decrypting the address it calls, and replaying the whole command-and-control cycle in an isolated lab. Network indicators are published in **defanged** form (`ycxrl[.]com`, `hxxp://`, `85.17.155[.]53`) so that defenders can hunt them without this document resolving or executing anything. What is **not** published is the reproduction material: cryptographic keys, decrypted payloads and the task archive, payload checksums, and the lab device's own serial / MAC / UUID.

## Read the report

1. [Executive summary](docs/01-executive-summary.md)
2. [This is not a sideloaded-app infection](docs/02-not-an-app-infection.md)
3. [The hunt](docs/03-the-hunt.md)
4. [Attack chain](docs/04-attack-chain.md)
5. [Who is exposed](docs/05-exposure-narrative.md)
6. [ATT&CK, indicators, limits](docs/06-attack-indicators-limits.md)
7. [Recommendations and ethics](docs/07-recommendations-ethics.md)

The single publishable file is [`paper/REPORT.md`](paper/REPORT.md).

## Handling

This document is published openly so that authorities, defenders, and the public can review the case. Network indicators appear in defanged form, deliberately neutered so that nothing here is clickable, resolvable, or executable; refang them only inside controlled defensive tooling. Reproduction material — cryptographic keys, decrypted payloads, checksums — and the lab device's own identifiers are kept out of this public tree and can be shared with competent authorities on request. Do not re-fang the indicators or add keys, payloads, or device identifiers to this repository.

## Citation

Moraga, P., & De La Paz, A. (2026). *Plug and Prey: a factory-backdoored Android TV box and its BadBox C2.* ekoparty 2026. See [CITATION.cff](CITATION.cff).

Text is licensed under [CC BY 4.0](LICENSE).
