# Customer-hosted relays and fully managed platforms for "Grandad's radio" (no vendor-run relay fleet)

Research date: 14 November 2026. Prices are in US$ unless marked NZ$; where converted, the project's own rate of NZ$1.81 per US$1 is used (from `Walkie-Talkie Beyond the R1 Minimum Hardware.md`; not re-checked).

**Method caveat for the report writer.** In this session page fetches were blocked by the network allowlist (developers.cloudflare.com, livekit.com, aws.amazon.com, www.spark.co.nz and 2degrees.nz all failed to resolve). Every figure below therefore comes from search-result extracts of the cited pages, not from a full read of the page. Vendor prices should be re-read on the linked page before they go into a business case. Anything I could not source this session is in a Gaps list and marked "unverified recollection" where I am relying on memory.

**Load model used throughout (assumptions, from the brief):**

| Item | Value | Arithmetic |
|---|---|---|
| Units per family | 4 | given |
| Talk time | 20 min/day for the whole family channel (one talker at a time) | given; if it is 20 min per unit, multiply all voice figures by 4 |
| Wire rate | 46 kbit/s, 50 packets/s (20 ms frames) | given |
| Ingress per family per month | 0.207 GB | 1,200 s x 46,000 bit/s = 55.2 Mbit = 6.9 MB/day; x 30 |
| Egress per family per month (fan-out to 3 listeners) | 0.621 GB | 0.207 x 3 |
| Packets per family per month | 1.8 M in, 5.4 M out | 50 x 1,200 x 30; x 3 |
| Presence connection-minutes per family per month | 172,800 | 4 units x 43,200 min |
| Stored message | about 345 kB per family | 60 s x 46 kbit/s / 8 |
| Share of time a family is talking | 1.4% average; 8.3% if all talk falls in a 4-hour evening peak | 20/1,440; 20/240 |

---

## A1. Can an ESP32-S3 itself be the family's relay?

### Takeaway
Packet fan-out for 10 peers is a trivial load for an ESP32-S3; the limits are sockets and TLS memory, both configurable, and I found no Espressif benchmark that states them per connection. The practical blocker is not the chip but reachability of the home it sits in (see A2).

### Cited Findings
- ESP-IDF's `LWIP_MAX_SOCKETS` accepts 1 to 253 with a default of 10; the practical ceiling is "determined by available heap memory at runtime" — [ESP-IDF lwIP Kconfig mirror](https://code.brunner.ninja/feedc0de/esp-idf/raw/branch/https_server_authmode_optional/components/lwip/Kconfig)
- `esp_https_server` default config sets `max_open_sockets` to 4 — [esp_https_server.h, ESP-IDF v5.2.2](https://github.com/espressif/esp-idf/blob/v5.2.2/components/esp_https_server/include/esp_https_server.h)
- Raising the socket limit alone is not enough: a forum report needed `LWIP_MAX_ACTIVE_TCP` raised to 32 as well before six browser clients worked — [esp32.com thread](https://esp32.com/viewtopic.php?p=45554)
- A CircuitPython port reported that it is "possible to run out of IDF memory with two TLS connections" on a non-PSRAM configuration, with memory use non-linear in socket activity (hardware-specific warning, not a measurement) — [CircuitPython PR 6021](https://github.com/adafruit/circuitpython/pull/6021)
- Espressif documents memory-saving mbedTLS options (for example not keeping the peer certificate after handshake) — [ESP-FAQ mbedTLS](https://docs.espressif.com/projects/esp-faq/en/latest/software-framework/protocols/mbedtls.html); a public ESP32-S3 sdkconfig shows `CONFIG_LWIP_MAX_SOCKETS=16` with `CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC=y` (mbedTLS allocating from PSRAM) alongside HTTPS and WebSocket servers — [example sdkconfig](https://git.commander1024.de/Commander1024/ESP32_Serial_Swiss_Army_Knife/raw/branch/devel/sdkconfig.defaults)
- The project's own measurements: fixed-point Opus encode on an S3 takes about 4 ms per 20 ms frame, decode 2 to 9% of a core, so CPU headroom exists — project doc `/home/user/walkie-talkie/Walkie-Talkie Beyond the R1 Minimum Hardware.md` (line 57), citing [micro-opus](https://components.espressif.com/components/esphome/micro-opus/versions/0.4.1/readme)

### Inferences
- Fan-out load on a base-station unit: 50 packets/s in, up to 9 x 50 = 450 packets/s out, 9 x 46 kbit/s = 414 kbit/s uplink while someone talks. This is far below what an S3's Wi-Fi can carry and below any NZ fibre uplink; on fixed wireless or Starlink it is still small. The relay never transcodes (payload is end-to-end encrypted), so the CPU cost is copy plus sendto.
- A TLS *server* for 9 peers is the expensive part (handshake RAM, certificates, renewal with no public CA unless a DNS name is used). The design already has per-unit key pairs and a group key, so a base station can skip TLS and authenticate control messages with the same keys over UDP. That removes the certificate and socket-count problems.
- The 24-hour message store (about 345 kB) fits in PSRAM or flash easily.
- New failure modes with a unit as relay: the whole family goes silent when that home loses power or internet, when the router reboots and the port mapping is lost, or when the base station is unplugged ("always plugged in" is a behavioural assumption about an elderly user's house). A second unit able to take over the role is needed, which means election logic in firmware.

### Gaps
- No official Espressif figure found for RAM per mbedTLS server session or handshake time on the S3 (unverified recollection: roughly 30 to 40 kB per session with default 16 kB record buffers, and around 1 to 2 s for an RSA-2048 handshake; measure with `heap_caps_get_free_size()`).
- No Espressif UDP throughput benchmark was retrieved this session.
- I found no shipping product example of an ESP32 acting as an internet-reachable relay for other homes; hobby web servers behind port forwards exist but I did not source one. ESP-IDF ships no UPnP-IGD/PCP client (unverified recollection: third-party Arduino libraries such as TinyUPnP exist).

---

## A2. Reachability of a home device in New Zealand (UPnP/PCP, CGNAT, IPv6, dynamic DNS)

### Takeaway
A meaningful share of NZ homes cannot accept inbound connections at all: fixed wireless (about 20% of fixed broadband), Starlink, and 2degrees dynamic-IP customers are behind CGNAT, and where the IPv4 address is public the mapping still depends on a router setting that varies by brand. Inbound hosting can therefore only be an optimisation, never the only path, unless a rendezvous or relay fallback exists.

### Cited Findings
**Access mix**
- Fixed wireless is about 20% of NZ's fixed broadband market and fibre 74% (article undated in the extract) — [iStart, "Kiwis take to skies for broadband"](https://istart.com.au/news-items/kiwis-take-to-skies-for-broadband/)
- Chorus: 1,147,000 fibre connections and 75.9% uptake across 1,499,000 passed addresses at end of June 2026 — [Chorus Q4 FY26 update via Kalkine](https://kalkinemedia.com/au/news/announcements/chorus-limited-achieves-115-million-fibre-connections-as-copper-network-phase-out-nears-completion-in-q4-fy26)
- The Commerce Commission reportedly found Starlink is now the largest rural broadband provider, ahead of Spark and One NZ — [iStart](https://istart.com.au/news-items/kiwis-take-to-skies-for-broadband/)

**CGNAT**
- 2degrees: "If you've got a ... Dynamic IP broadband connection (not a Static IP), then you'll be moved to CGNAT"; applies to personal broadband only; a static IP costs NZ$10/month for home broadband — [2degrees IP Addresses](https://2degrees.nz/help/broadband-help/ip-addresses/2degrees-ip-addresses)
- Spark Wireless Broadband shares a pool of addresses through CG-NAT instead of giving each connection its own IP — [Spark rural page](https://spark.co.nz/rural); Spark recommends adding a static IP for affected uses — [Spark service restrictions](https://www.spark.co.nz/help/internet/learn/service-restrictions/)
- Skinny does not offer dedicated IP addresses, and its wireless service has a CG-NAT help section — [Skinny 4G wireless IP addresses](https://www.spark.co.nz/help/wireless-4g-broadband/network-and-speed/ip-addresses/)
- Electric Kiwi says it does not use CGNAT on fibre — [Electric Kiwi broadband FAQ](https://www.electrickiwi.co.nz/faqs/broadband)
- "Many providers" are using CGNAT ahead of wider IPv6 rollout — [Canstar Blue NZ](https://www.canstarblue.co.nz/?p=86964)
- Starlink Residential and Roam IPv4 is behind CGNAT (100.64.0.0/10), blocking all inbound IPv4 — [natchecker Starlink guide](https://natchecker.com/blog/starlink-port-forwarding) (vendor blog; consistent with several others)

**IPv6**
- One NZ says it has deployed IPv6 across its fixed-line network with dual stack for compatible hardware — [One NZ IPv6 FAQ](https://one.nz/faq/about-one-nz-and-ipv6)
- The stock Starlink router supports no port forwarding or firewall rules for IPv4 or IPv6; inbound IPv6 is blocked with no UI to open it, and a third-party router in bypass mode is needed — [natchecker](https://natchecker.com/blog/starlink-port-forwarding), [DishyTech](https://www.dishytech.com/does-starlink-have-port-forwarding/) (DishyTech's claim that IPv6 removes the need for port forwarding conflicts with the inbound-filtering reports)

**Router port-mapping defaults**
- Google Nest Wifi / Google Wifi: UPnP is enabled by default and can be turned off in the Google Home app — [Google Nest help](https://support.google.com/googlehome/answer/6274337)
- Fritz!Box: firewall blocks all inbound by default; self-service port mapping (UPnP or PCP) is a per-device "Permit independent port sharing" option that the owner must enable, and AVM warns malware could use it — [AVM knowledge base](https://fritz.com/en/apps/knowledge-base/FRITZ-Box-4020/894_Setting-up-automatic-port-sharing), [Fritz!Box 7590 help](https://service.avm.de/help/en/FRITZ-Box-7590/015/hilfe_portfreigabe)
- General statement that UPnP is "often enabled by default on consumer-grade routers" — [TP-Link blog](https://www.tp-link.com/us/blog/2531/)
- Starlink router: no port forwarding of any kind — as above

**NAT traversal success without port opening (comparison point)**
- Tailscale reports from its own metrics that "more than nine out of 10" node-to-node connections end up direct after hole punching, but all connections start relayed via DERP and hard NAT or double NAT forces relay — [Tailscale NAT traversal improvements, part 1](https://tailscale.com/blog/nat-traversal-improvements-pt-1), [Tailscale connection types](https://tailscale.com/docs/reference/connection-types), [Tailscale peer relays post, Jan 2026](https://tailscale.com/blog/peer-relays-international-networks)
- Cloudflare's STUN service (`stun.cloudflare.com`) is free and unlimited — [Cloudflare TURN FAQ](https://developers.cloudflare.com/realtime/turn/faq/)

**Dynamic DNS**
- DuckDNS is a free service hosted on AWS; users report it "seems to go down quite often" (Mar 2025), with outages logged in Oct 2023 and reported in Aug 2025 (the latter from an unverified blog) — [StatusGator](https://statusgator.com/services/duck-dns), [Jellyfin forum](https://forum.jellyfin.org/t-reliable-duckdns-alternative), [Home Assistant community, Oct 2023](https://community.home-assistant.io/t/duckdns-issues-oct-10-2023-and-intemittently-on-oct-11-but-now-back-up/625032)
- deSEC is free and supported by common update clients — [Jellyfin forum](https://forum.jellyfin.org/t-reliable-duckdns-alternative)

### Inferences
- **What fraction of homes could host (rough estimate, not a measurement).** Start from access type: about 20% fixed wireless (CGNAT at Spark/Skinny) plus Starlink homes are out. Of the roughly 74% on fibre, 2degrees-group dynamic customers are out, and the rest need a router that honours UPnP/PCP for a device with no screen. A defensible planning range is 35 to 55% of NZ homes able to accept inbound UDP with no customer action; I could not find data to tighten it.
- **If only one home per family must be reachable:** with per-home probability p and n independent homes, P(at least one) = 1 - (1 - p)^n. At p = 0.5: 75% for 2 homes, 87.5% for 3, 93.75% for 4. At p = 0.35: 58%, 73%, 82%. So even in the best case roughly 1 family in 8 to 1 in 4 has no hostable home, and the product needs a fallback path for them. Independence is optimistic: families cluster on the same ISP and rural families on the same access type.
- The reachable home becomes a single point of failure, and it may not be the home with the "always plugged in" unit. Router replacement, ISP change or an ISP moving customers to CGNAT (as 2degrees did) silently breaks a working family, and the vendor has no visibility unless a central diagnostics service exists.
- Hole punching with a stateless rendezvous (free STUN plus a tiny signalling exchange) avoids asking anyone to open a port and works for most NAT pairs, but CGNAT-to-CGNAT pairs still need a relay. Some relay capacity is therefore unavoidable in any design that must work for every family.
- A vendor-owned DNS zone on a managed DNS API is safer than a free dynamic DNS service: it removes dependence on a hobby service, and the "DNS record" can just as well be a signed pointer stored in object storage that units poll.

### Gaps
- Spark fibre and One NZ fibre CGNAT status in 2026: no authoritative statement found (unverified recollection: both have historically given fibre customers a dynamic public IPv4). Geekzone threads were not retrievable.
- Default UPnP state on the Spark Smart Modem, One NZ modems, TP-Link and eero: not found (unverified recollection: TP-Link and eero ship with UPnP on; eero also supports PCP/NAT-PMP).
- IPv6 inbound firewall defaults per NZ router: not found beyond Starlink and Fritz!Box (unverified recollection: RFC 6092-style default-deny inbound is the norm, and PCP is the only standard way for a device to open an IPv6 pinhole).
- DuckDNS and deSEC terms of service and rate limits; Cloudflare DNS API limits on the free plan: not retrieved.
- No NZ-specific measurement of UPnP-enabled share or CGNAT share of households exists in what I found.

---

## A3. Security and liability of asking customers to expose a port

### Takeaway
The consumer-device record is poor: the largest incidents came from devices reachable from the internet with weak authentication, and from P2P "no port forwarding needed" SDKs whose central registration was spoofable. A design that opens ports via UPnP inherits the reputation and the attack surface, and the vendor carries it in customers' homes where it cannot patch the router.

### Cited Findings
- ThroughTek Kalay, CVE-2021-28372 (disclosed 17 Aug 2021 by Mandiant with CISA): CVSS 9.6; an attacker who knew a device UID could register as that device and then eavesdrop on audio/video and take over the device. ThroughTek advertised more than 83 million active devices and 1.1 billion monthly connections; Mandiant could not list affected devices. Fix: SDK 3.1.10 or later plus DTLS and AuthKey — [Mandiant / Google Cloud blog](https://cloud.google.com/blog/topics/threat-intelligence/mandiant-discloses-critical-vulnerability-affecting-iot-devices), [Help Net Security](https://www.helpnetsecurity.com/2021/08/17/cve-2021-28372/)
- Mirai (2016): spread by trying a list of more than 60 default username/password pairs; Level 3 saw it on at least 500,000 devices; Dyn estimated about 100,000 infected devices in the October 2016 attack; camera vendor Xiongmai admitted weak default passwords in its products were partly to blame — [CSO Online](https://www.csoonline.com/article/558517/chinese-firm-admits-its-hacked-products-were-behind-fridays-massive-ddos-attack.html), [Computerworld](https://www.computerworld.com/article/3135434/security/ddos-attack-on-dyn-came-from-100000-infected-devices.html)
- CallStranger, CVE-2020-12695: a flaw in the UPnP SUBSCRIBE callback usable for DDoS reflection, data exfiltration and internal port scanning; about 5.45 million UPnP devices were exposed to the internet; the advice was to disable UPnP or keep it off the internet — [Tenable](https://es-la.tenable.com/blog/cve-2020-12695-callstranger-vulnerability-in-universal-plug-and-play-upnp-puts-billions-of), [SecurityWeek](https://www.securityweek.com/callstranger-upnp-flaw-affecting-billions-devices-allows-data-exfiltration-ddos-attacks/amp/)
- AVM's own documentation warns that enabling self-service port sharing lets malware on that device open ports through the firewall — [AVM knowledge base](https://fritz.com/en/apps/knowledge-base/FRITZ-Box-4020/894_Setting-up-automatic-port-sharing)
- A Virus Bulletin 2024 paper exists on the security cost of "open by default" consumer networking — [VB2024 paper](https://virusbulletin.com/uploads/pdf/conference/vb2024/papers/Open-by-default-the-hidden-cost-of-convenience-in-network-security.pdf) (title only seen)

### Inferences
- The Kalay lesson applies directly to design B-style and rendezvous designs too: a P2P system with a central lookup is only as strong as the proof a device gives when it registers. Grandad's radio already has per-unit key pairs, so registration and lookup records can be signed; Kalay used a bare UID.
- An exposed base station's realistic risks: (1) DoS or battery/bandwidth drain from internet scanning of the open UDP port, (2) remote code execution through a parser bug on a device that sits inside the family LAN, (3) the unit used as a traffic reflector. Mitigations: answer nothing unless the first packet carries a valid MAC under a group-derived key (silent drop), random high port, no TCP listener, memory-safe parsing, signed OTA. With silent drop the port is indistinguishable from closed.
- Liability and support: security guidance from ISPs and router vendors tells consumers to turn UPnP off. A product that requires it on will collide with that advice, with router "security scan" warnings, and with managed mesh systems that hide the setting. In a product sold to non-technical grandparents, "log in to your router" is a support call the vendor will lose.

### Gaps
- Akamai's UPnProxy research (routers with injected NAT entries) and the Mirai peak figure of 600,000 were not sourced this session (the 600,000 figure appeared only on a low-quality Q&A site; unverified recollection is that it comes from the 2017 USENIX "Understanding the Mirai Botnet" paper).
- No NZ-specific legal analysis (Consumer Guarantees Act, Privacy Act 2020) of vendor liability for a device that opens a port was found.

---

## A4. Precedents: products where the customer's box is the server, and what the vendor still ran

### Takeaway
Every precedent I could source that sells to non-experts ended up running central rendezvous plus a relay fallback, and several charge a subscription or throttle to pay for it. "Customer hosts" reduced the vendor's bandwidth bill; it did not remove the vendor's always-on services.

### Cited Findings
- **Tailscale:** all connections start relayed through vendor DERP servers, which also carry the signalling for NAT traversal; more than 9 in 10 then upgrade to direct; hard NAT and double NAT stay relayed, and in 2025-26 Tailscale added customer-run "peer relays" as another fallback — [connection types](https://tailscale.com/docs/reference/connection-types), [NAT traversal improvements](https://tailscale.com/blog/nat-traversal-improvements-pt-1), [peer relays](https://tailscale.com/blog/peer-relays-international-networks)
- **Plex:** the media server is the customer's, yet Plex runs a Relay service for servers that cannot be reached directly; relayed streams are capped at 2 Mbit/s for Plex Pass / Remote Watch Pass subscribers (older forum quotes of the same article say 1 Mbit/s for free users) — [Plex Relay support article](https://support.plex.tv/articles/216766168-accessing-a-server-through-relay/), [Plex forum](https://forums.plex.tv/t/plex-bandwidth-limit-exceeded/571065)
- **Synology QuickConnect:** vendor-run QuickConnect server does LAN/WAN detection, then hole punching, then falls back to a Synology Relay Server "for any NAS not reachable using those methods"; relayed traffic is slower — [Synology QuickConnect White Paper](https://cndl.synology.cn/download/Document/Software/WhitePaper/Os/DSM/All/enu/Synology_QuickConnect_White_Paper_enu.pdf)
- **Home Assistant + Nabu Casa:** remote access is a vendor-run relay (SniTun, "End-to-End encryption with SNI proxy on top of a TCP multiplexer"); the home instance dials out and must stay connected; Nabu Casa states it cannot view the data; the subscription is EUR 7.50/month or EUR 75/year in the EU — [Nabu Casa remote access docs](https://support.nabucasa.com/hc/en-us/sections/26496346499997-Home-Assistant-remote-access), [Nabu Casa SniTun article](https://support.nabucasa.com/hc/en-us/articles/26510075061021)
- **Syncthing:** relays are run by volunteers and are "not managed or vetted by the Syncthing project", but each relay announces itself to a global relay pool server hosted by the project, and a forum snapshot put combined relay capacity at 10 Gbit/s with about 900 Mbit/s in use — [Syncthing relay server docs](https://docs.syncthing.net/users/strelaysrv), [relays.syncthing.net](https://relays.syncthing.net), [forum: relay network capacity](https://forum.syncthing.net/t/relay-network-capacity/11470)

### Inferences
- The common pattern has three tiers: (1) central identity/rendezvous, always vendor-run; (2) direct path when NAT allows; (3) relay fallback, vendor-run or volunteer-run. Only tier 3 carries media, and it is the tier vendors throttle (Plex) or charge for (Nabu Casa). For Grandad's radio the media is 46 kbit/s, so tier 3 is cheap; the cost that precedents could not escape is tiers 1 and 3 being *available*, not their bandwidth.
- Nabu Casa's relay has the same trust property as the current design: the relay forwards ciphertext and holds no keys. The owner's existing relay is already the "good" version of what these vendors converged on.
- Syncthing shows a volunteer relay pool is possible for a free, technical community. It is not a model a paid consumer product for grandparents can lean on: no SLA, relays self-report, and the project still hosts the pool and discovery servers.

### Gaps
- Not sourced this session (all unverified recollection): Apple HomeKit home hubs (remote access goes through iCloud, so Apple runs the rendezvous and relay); Matter/Thread border routers (local only; remote access comes from each ecosystem's cloud); Briar Mailbox (customer's spare Android phone reached over Tor onion services, so the "central" part is the volunteer Tor network; latency unsuited to live voice); Umbrel/Start9 (remote access by Tor or a Tailscale-type overlay); Mumble home servers (public server list run by the project; home hosting needs a port forward); Helium hotspots (many hotspots sat behind NAT and were relayed through other hotspots, with known performance penalties).
- Share of Home Assistant installs that pay for Nabu Casa, and current Syncthing figures for the percentage of connections relayed, were not found.
- Plex's 2025 move of remote streaming behind Plex Pass / Remote Watch Pass could not be confirmed from an official page, though the support-article extract mentions the Remote Watch Pass.

---

## A5. Optional "family hub" (Raspberry Pi Zero 2 W or router plugin)

### Takeaway
A Pi hub costs about as much as a walkie unit, adds a second product to support, and does not fix reachability, since it sits behind the same router and CGNAT as an ESP32 base station would.

### Cited Findings
- Raspberry Pi Zero 2 W in NZ: NZ$39.84 including GST at PB Tech (NZ$34.64 ex GST); NZ$34.99 at NZ Depot (sold out when seen) — [PB Tech](https://www.pbtech.co.nz/product/SEVRBP0368/Raspberry-Pi-Zero-2-W-with-Soldered-Male-Header-1G), [PriceSpy NZ](https://pricespy.co.nz/product.php?p=14539282), [NZ Depot](https://nzdepot.co.nz/product/raspberry-pi-zero-2-w-arm-64-bit-cortex-a53-cpu-512mb-sdram-2-4ghz-802-11-b-g-n-wireless-lan-bluetooth-ble-composite-video-and-reset-pins-via-solder-test-points-2/)
- The project's unit bill of materials is about NZ$35 to 50 with battery — project doc `/home/user/walkie-talkie/Walkie-Talkie Beyond the R1 Minimum Hardware.md` (lines 9, 216)

### Inferences
- Hub landed cost: board about NZ$40 plus SD card, power supply and case, so roughly NZ$70 to 90 per family (my estimate; accessories not priced this session). That is one to two extra walkie units of cost per family.
- What a Linux hub buys over an ESP32 base station: it can run the existing Go relay unchanged, a real TLS stack, and an outbound overlay client (a WireGuard-based mesh or Tor) that gets through NAT. But an overlay that gets through NAT is again someone's central coordination and relay service.
- What it costs: a Linux image to patch for years, SD-card wear and corruption on power cuts, a second OTA pipeline, and a second device for Grandad's household to keep powered. A router plugin (OpenWrt package) only reaches the small minority with open-firmware routers and is unsupportable at consumer scale.

### Gaps
- No sourced data on SD-card failure rates or field support cost of Pi-based appliances.

---

## B1. Managed platform options, transports and ESP32 support

### Takeaway
No mainstream managed platform accepts the product's current raw UDP protocol. The choices are WebRTC (Cloudflare Realtime SFU, LiveKit, Kinesis Video Streams), WebSocket (Cloudflare Durable Objects) or MQTT/pub-sub (AWS IoT Core, EMQX, HiveMQ, Ably, PubNub, Pusher), and the pub-sub group is priced per message, which makes it unsuitable for 50 packets/s audio but cheap for presence and control.

### Cited Findings
**WebRTC media platforms**
- Cloudflare Realtime SFU and TURN: $0.05 per GB egress, first 1,000 GB/month free, shared between SFU and TURN; traffic between TURN and SFU is not charged; STUN is free — [Cloudflare SFU pricing](https://developers.cloudflare.com/realtime/sfu/platform/pricing/), [TURN FAQ](https://developers.cloudflare.com/realtime/turn/faq/). One Cloudflare overview table instead says TURN alone has no free tier — [Realtime overview](https://developers.cloudflare.com/realtime/)
- LiveKit Cloud: Build free (5,000 WebRTC minutes, 50 GB); Ship $50/month (150,000 minutes, then $0.0005/min; 250 GB, then $0.12/GB); Scale $500/month (1.5 M minutes, then $0.0004/min; 3 TB, then $0.10/GB). A WebRTC minute is one participant connected for one minute. Upstream bandwidth is free — [LiveKit pricing](https://livekit.com/pricing), [LiveKit billing docs](https://docs.livekit.io/deploy/admin/billing/)
- LiveKit ESP32 SDK: supports ESP32-S3 and ESP32-P4 only, Opus audio with echo cancellation, built on Espressif's WebRTC components, published as `livekit/livekit` (0.3.10 latest seen), and explicitly "Developer Preview ... not ready for production" — [LiveKit ESP32 docs](https://docs.livekit.io/frontends/build/hardware/esp32/), [Espressif component registry](https://components.espressif.com/components/livekit/livekit), [LiveKit blog](https://livekit.com/blog/livekit-sdk-for-esp32-bringing-voice-ai-to-embedded-devices). One user reported signalling succeeding but the RTC connection failing after DTLS/TURN on IDF 5.5.3 (SDK tested on 5.4.x) — [LiveKit community](https://community.livekit.io/t/esp32-rtc-error/682)
- Amazon Kinesis Video Streams WebRTC (US-East): $0.03 per active signalling channel per month, $2.25 per million signalling messages, $0.12 per thousand TURN streaming minutes, plus AWS data transfer; no free tier — [KVS pricing](https://aws.amazon.com/kinesis/video-streams/pricing/)

**WebSocket compute**
- Cloudflare Durable Objects (Workers Paid plan): 1 million requests included then $0.15 per million; duration 400,000 GB-s included then $12.50 per million GB-s (billed at 128 MB); incoming WebSocket messages are billed 20:1 (100 messages = 5 requests); outgoing messages and incoming protocol pings are free; a hibernatable idle object incurs no duration charge; a non-hibernating `accept()` is billed for the whole connection — [Durable Objects pricing](https://developers.cloudflare.com/durable-objects/platform/pricing), [Durable Objects FAQ](https://developers.cloudflare.com/durable-objects/reference/faq)
- SQLite storage in Durable Objects became billable (announced Dec 2025, target no earlier than 7 Jan 2026) — [Cloudflare changelog](https://developers.cloudflare.com/changelog/2025-12-12-durable-objects-sqlite-storage-billing)

**MQTT and pub-sub**
- AWS IoT Core (US rates): $0.08 per million connection-minutes; messaging from $1.00 per million (down to $0.70 at volume), metered in 5 KB increments — [AWS IoT Core pricing](https://aws.amazon.com/iot-core/pricing/)
- EMQX Cloud Serverless: 1 million session-minutes free per month, then $2.00 per million; $0.15/GB traffic beyond 1 GB; up to 1,000 connections on the Serverless plan — [EMQX pricing docs](https://docs.emqx.com/en/cloud/latest/price/pricing.html)
- HiveMQ Cloud: Starter is usage-based on runtime and message volume with a 99.95% SLA; third-party sites quote $0.34/hour plus $0.80 per million messages; the free serverless plan allows 100 connections — [HiveMQ editions](https://www.hivemq.com/hivemq/editions/), [freetier.co comparison](https://freetier.co/compare/emqx-serverless-vs-hivemq-cloud)
- Ably: consumption pricing since 25 Oct 2024 at $2.50 per million messages, $1.00 per million connection-minutes, $1.00 per million channel-minutes, plus a $29/month Standard base; messages are counted per delivery — [budgetforge Ably teardown](https://www.budgetforge.dev/tools/ably-pricing-2026) (third party)
- Pusher Channels: plans from free (100 concurrent connections, 200k messages/day) to $49, $99, $299 and up, metered on peak concurrent connections and messages per day — [Ably's Pusher pricing page](https://ably.com/topic/pusher-pricing), [budgetforge](https://www.budgetforge.dev/tools/pusher-pricing-2026) (third party; top tier quoted as $499 or $1,199)
- PubNub: free, $98/month Starter, then quote-based; illustrative about $190/month at 1,000 MAU — [frontdeskreview PubNub](https://frontdeskreview.com/software/chat-api/pubnub/) (third party)

### Inferences
- Transport fit summary:

| Platform | Transport | Raw UDP? | ESP32 path | Fit for live audio | Fit for presence/control |
|---|---|---|---|---|---|
| Cloudflare Realtime SFU | WebRTC (DTLS-SRTP) | No | Espressif WebRTC stack; no Cloudflare ESP32 SDK found | Good, cheapest per GB | None (needs a Worker for signalling) |
| Cloudflare Durable Objects | WebSocket over TLS/TCP | No | ESP-IDF WebSocket client | Workable; TCP head-of-line blocking on lossy Wi-Fi | Good (hibernation) |
| LiveKit Cloud | WebRTC | No | Official SDK, developer preview | Good, but per-minute metering | Poor: always-on connection is billed per minute |
| KVS WebRTC | WebRTC, master/viewer | No | AWS C SDK (unverified) | Poor for a group: not an SFU | n/a |
| AWS IoT Core | MQTT over TLS | No | Well supported in ESP-IDF (unverified, not searched) | Poor: per-message billing | Good |
| EMQX / HiveMQ | MQTT | No | Standard MQTT client | Poor | Fine at small scale; connection caps |
| Ably / PubNub / Pusher | WebSocket or MQTT bridge | No | No first-party ESP32 SDK found | Poor | Costly relative to IoT Core |

- End-to-end encryption survives on all of them if the unit encrypts the Opus payload before handing it to the transport. An SFU forwards RTP without decoding, and a WebSocket or MQTT broker carries opaque bytes. The platform's own transport encryption then becomes a second layer.
- None of these platforms provides floor control, a 24-hour last-message store, or offline alerts as features. Those are still the vendor's code: a few hundred lines in a Durable Object or a Lambda, with no server to patch, but code to own.

### Gaps
- Agora, Daily, Azure Web PubSub, Firebase, Supabase Realtime, Fly.io, Golioth, Blynk and Particle pricing were not sourced this session. Unverified recollection: Agora and Daily bill audio per participant-minute (about $0.99 per 1,000 minutes at Agora), which has the same always-on problem as LiveKit; Fly.io is a container host that *does* accept raw UDP, but running the Go relay there is still running the relay; Golioth/Blynk/Particle are device-management platforms (OTA, fleet, telemetry), not media relays.
- AWS IoT Core Sydney-region rates, and whether each delivered message is billed as well as each published one (unverified recollection: yes, both are metered).
- HiveMQ, Ably, Pusher and PubNub figures are from third-party summaries, not vendor pages.

---

## B2. Cost per family and at 1,000 / 10,000 / 100,000 families, versus self-run VPS relays

### Takeaway
The voice load is so small that every viable option costs cents per family per month: about US$0.03 on Cloudflare's SFU, US$0.07 to 0.13 on Durable Objects, and well under US$0.01 on self-run VPSs. LiveKit is roughly US$1 per family even when units connect only while talking. Money does not separate the options; operating effort and protocol rework do.

### Cited Findings
- Unit prices used below are those cited in B1: Cloudflare SFU $0.05/GB after 1,000 GB — [Cloudflare](https://developers.cloudflare.com/realtime/sfu/platform/pricing/); Durable Objects $0.15/M requests, $12.50/M GB-s, 20:1 WebSocket ratio — [Cloudflare](https://developers.cloudflare.com/durable-objects/platform/pricing); LiveKit Scale $500 with 1.5 M minutes then $0.0004/min and 3 TB then $0.10/GB — [LiveKit](https://livekit.com/pricing); AWS IoT Core $0.08/M connection-minutes and $1/M messages — [AWS](https://aws.amazon.com/iot-core/pricing/); EMQX $2/M session-minutes, 1,000-connection cap — [EMQX](https://docs.emqx.com/en/cloud/latest/price/pricing.html); Ably $1/M connection-minutes, $2.50/M delivered messages — [budgetforge](https://www.budgetforge.dev/tools/ably-pricing-2026); R2 $0.015/GB-month, 10 GB free, free egress — [Cloudflare R2 pricing](https://developers.cloudflare.com/r2/platform/pricing/)
- The project already identifies a free Oracle tier or a small Vultr VPS as the relay host — project doc `/home/user/walkie-talkie/Walkie-Talkie Beyond the R1 Minimum Hardware.md` (line 247)

### Inferences (all arithmetic is mine, on the load model at the top)

**Option 1: Cloudflare Realtime SFU for voice (WebRTC), presence elsewhere**
- Per family: 0.621 GB x $0.05 = **$0.031/month** (NZ$0.056).
- 1,000 families: 621 GB, inside the free 1,000 GB = **$0**.
- 10,000: (6,210 - 1,000) x 0.05 = **$260.50/month**.
- 100,000: (62,100 - 1,000) x 0.05 = **$3,055/month**.
- Not included: signalling Worker, presence, message store (see option 2 or the IoT Core line).

**Option 2: Cloudflare Workers + one Durable Object per family, audio and presence over hibernating WebSockets**
- Requests: 1.8 M inbound audio frames / 20 = 90,000, plus about 3,000 for presence and control = 93,000 billable requests = $0.014 per family. (Sending 60 ms per message cuts the audio part to 30,000.)
- Duration: the object is awake while someone talks. 36,000 s/month x 0.128 GB = 4,608 GB-s = $0.058. If bursty talk keeps it awake twice as long: 9,216 GB-s = $0.115.
- Per family: **$0.07 to $0.13/month** (NZ$0.13 to 0.23). Idle presence connections cost nothing while hibernated; protocol pings are free.
- 1,000 families: requests (93 M - 1 M) x 0.15 = $13.80; duration (4.608 M - 0.4 M) x 12.5 = $52.60; plan $5: **about $71** (high case $129).
- 10,000: $139 + $571 + $5 = **about $715** (high case $1,291).
- 100,000: $1,395 + $5,755 + $5 = **about $7,155** (high case $12,915).
- Message store: 100,000 families x 345 kB = 34.5 GB, about $0.52/month in R2 or a few dollars in Durable Object storage. Negligible at any scale.

**Option 3: LiveKit Cloud (WebRTC, official ESP32 SDK in preview)**
- Always-on presence in a room: 172,800 min x $0.0005 = **$86 per family per month**. Not viable.
- Connect only while talking (all 4 units join for 20 min/day): 2,400 participant-minutes = $0.96 at the Scale overage rate, plus bandwidth 0.621 GB x $0.10 = $0.06: **about $1.02 per family** (NZ$1.85). This needs a separate always-on wake/presence channel.
- 1,000 families: 2.4 M min: $500 + 0.9 M x 0.0004 = **$860**.
- 10,000: $500 + 22.5 M x 0.0004 + (6,210 - 3,000) x 0.10 = **about $9,821**.
- 100,000: $500 + 238.5 M x 0.0004 + 59,100 x 0.10 = **about $101,810** (list price; enterprise terms would differ).

**Presence and control on a managed broker (pairs with option 1 or 3)**
- AWS IoT Core: 172,800 min x $0.08/M = $0.0138, plus about 10,000 small messages = $0.01: **about $0.024 per family**; $24, $240 and $2,400 per month at the three scales.
- Audio over IoT Core is not viable: 1.8 M publishes + 5.4 M deliveries = 7.2 M messages = $7.20 per family (assuming deliveries are billed).
- EMQX Serverless: 172,800 x $2/M = $0.35 per family, and the 1,000-connection cap stops at 250 families.
- Ably: connection-minutes alone $0.17 per family; audio at $2.50/M delivered messages would be $13.50 per family.

**Self-run VPS relays (the current design)**
- CPU: at the 8.3% evening-peak concurrency, 10,000 families means about 833 simultaneous talkers = 41,650 packets/s in and 125,000 out, 38 Mbit/s in and 115 Mbit/s out. Plain UDP forwarding in Go handles this order of load on one or two cores (engineering judgement; no benchmark sourced).
- Memory: 4 idle TLS control connections per family; at an assumed 10 to 20 kB each in Go, 10,000 families need 0.4 to 0.8 GB.
- Transfer quota is the binding limit on a small plan: 0.83 GB per family per month in plus out. A plan with 1 TB of transfer carries about 1,200 families, 2 TB about 2,400 (plan sizes assumed, not sourced).
- So one NZ$10/month VPS realistically carries **1,500 to 3,000 families**, and far more if the plan has generous transfer. Cost: 1,000 families NZ$10 to 20 (two boxes for redundancy); 10,000 about NZ$40 to 70; 100,000 about NZ$400 to 700. That is NZ$0.004 to 0.02 per family per month.

**Side-by-side (US$ per month, list prices seen 14 Nov 2026; VPS converted at 1.81)**

| Families | Cloudflare SFU (voice only) | Durable Objects (voice + presence) | LiveKit (on-demand) | AWS IoT Core (presence only) | Self-run VPS |
|---|---|---|---|---|---|
| 1,000 | $0 | $71 to $129 | $860 | $24 | about $6 to $11 |
| 10,000 | $261 | $715 to $1,291 | $9,821 | $240 | about $22 to $39 |
| 100,000 | $3,055 | $7,155 to $12,915 | $101,810 | $2,400 | about $220 to $390 |

- At a retail price of, say, NZ$100 per unit, even the dearest sensible option (Durable Objects, NZ$0.23 per family per month) is under NZ$3 per family per year. Infrastructure cost should not drive the decision.

### Gaps
- VPS plan specifications and transfer allowances at the NZ$10 price point were not sourced; neither was a Go UDP forwarding benchmark.
- WebRTC overhead (SRTP, RTCP, ICE keepalives) would raise the 46 kbit/s wire rate somewhat; not quantified.
- Whether Workers/Durable Objects charge anything for WebSocket data volume (unverified recollection: no bandwidth charge).
- How long a Durable Object stays billable after the last message in a talk burst; the 2x "high case" is my allowance for it.

---

## B3. How much maintenance disappears with a managed platform, and what remains

### Takeaway
A managed platform removes the server-shaped work (OS patching, capacity, DDoS exposure of a fixed IP, failover) but not the product-shaped work (accounts, pairing, floor control and store logic, OTA, monitoring), and it adds a firmware protocol rewrite plus exposure to the provider's pricing and roadmap.

### Cited Findings
- LiveKit changed its pricing model in 2025 from connection-based metering toward agent-session minutes, and published posts titled "towards a future-aligned pricing model" and "the end of participant minute" — [LiveKit blog](https://livekit.com/blog/towards-a-future-aligned-pricing-model), [LiveKit blog](https://livekit.com/blog/the-end-of-participant-minute), [pricingsaas change log](https://pricingsaas.com/companies/livekit/diffs/2025Q3)
- Ably replaced peak-based pricing with per-minute consumption pricing on 25 Oct 2024 — [budgetforge](https://www.budgetforge.dev/tools/ably-pricing-2026)
- Cloudflare began charging for Durable Object SQLite storage in 2026 after a free period — [Cloudflare changelog](https://developers.cloudflare.com/changelog/2025-12-12-durable-objects-sqlite-storage-billing)
- Cloudflare has data centres in Auckland and Christchurch; before Auckland opened, all NZ traffic went via Sydney. A network performance incident at the Auckland facility was logged in May 2026 — [Cloudflare blog: Auckland and Melbourne](https://blog.cloudflare.com/auckland-melbourne/), [DataCenterNews Asia](https://datacenternews.asia/story/cloudflare-expands-a-nz-footprint-with-four-new-data-centres), [incident record](https://pingoru.io/providers/cloudflare/incidents/1688852)
- The LiveKit ESP32 SDK is a developer preview with API changes expected — [Espressif component registry](https://components.espressif.com/components/livekit/livekit)

### Inferences
- **Disappears:** OS and Go runtime patching, TLS certificate renewal on a relay host, firewalling and DDoS absorption for a public IP, capacity planning and sharding families across relays, regional failover, backups of relay state.
- **Remains:** device identity and pairing, group membership and re-keying, floor-control and last-message logic (now as platform code), offline/battery alert rules, signed OTA hosting, billing/usage monitoring, incident response when the platform has an outage, and support diagnostics. Platform credentials become a new secret to protect.
- **Added:** firmware must move from the custom UDP protocol to WebRTC or WebSocket. WebRTC on an S3 brings DTLS, ICE and SRTP stacks (RAM, flash, handshake time per talk burst) and a dependency on a preview-grade SDK or Espressif's stack. WebSocket keeps firmware simple but puts voice on TCP.
- **Lock-in and price risk:** three of the vendors above changed their pricing model in the last two years. The mitigation is to keep the device protocol provider-neutral (plain WebSocket or standard WebRTC/WHIP, standard MQTT) and keep relay endpoints in signed, updatable config, so a move back to a self-run relay is a config push, not a firmware rewrite. Durable Objects code is Cloudflare-specific but small.
- **Latency:** Cloudflare terminates in Auckland/Christchurch, so the device-to-edge leg is domestic; where the Durable Object or SFU session is placed is Cloudflare's choice. Anything hosted only in Sydney adds a trans-Tasman round trip of a few tens of milliseconds, which is immaterial for push-to-talk.
- **Data sovereignty:** because audio is end-to-end encrypted and the relay holds no keys, what leaves NZ is ciphertext plus metadata (who talked when, IP addresses, battery state). That metadata is still personal information under the NZ Privacy Act, and sending it offshore needs disclosure in the privacy policy; it is the same position as the current VPS unless that VPS is in NZ.

### Gaps
- No measured NZ-to-edge latency figures were found for Cloudflare, LiveKit or AWS. LiveKit Cloud's region list and AWS's NZ region status were not sourced (unverified recollection: AWS opened an Auckland region in 2025, and its service coverage for IoT Core was not checked).
- Platform SLA and outage history were not compared.
- No legal source was consulted on Privacy Act cross-border disclosure (IPP 12).

---

## B4. Irreducible central services for any commercial version, and the lowest-maintenance host for each

### Takeaway
Even a perfect peer-to-peer voice path leaves five central functions that only the vendor can provide; all five can be static files or pay-per-use functions with no server to patch, which is where "no central relay or similar" is realistically achievable.

### Cited Findings
- ESP-IDF OTA uses two firmware slots with automatic rollback and supports signed images, fetched by the device over HTTPS from any URL — [ESP-IDF OTA docs](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/ota.html) (as cited in the project doc)
- Cloudflare R2: $0.015 per GB-month, 10 GB free, 1 M Class A and 10 M Class B operations free per month, then $4.50 and $0.36 per million; egress to the internet is free — [Cloudflare R2 pricing](https://developers.cloudflare.com/r2/platform/pricing/)
- The project's control service is "optional by design: devices cache their last settings, so talking still works if the control service is down" — project doc `/home/user/walkie-talkie/Walkie-Talkie Beyond the R1 Minimum Hardware.md` (line 259)
- Precedents all kept central identity/rendezvous even when media went direct (Tailscale DERP and coordination, Synology QuickConnect server, Syncthing pool and discovery, Nabu Casa) — sources in A4

### Inferences
| Central function | Why it cannot be decentralised | Lowest-maintenance host | Rough cost |
|---|---|---|---|
| Firmware OTA distribution | Only the vendor holds the signing key; units must find new images | Static signed image plus a signed manifest in object storage behind a CDN; units poll daily | 2 MB image x 400,000 units = 800 GB per release; free egress on R2 |
| Firmware signing | Root of trust for every device | Offline key or hardware token on the vendor's desk; never in the cloud | Hardware token, one-off |
| Provisioning and pairing | A new unit must learn its family's keys and members | Local-first: BLE or QR pairing from a phone or between units, as in the current laptop-script design; optional claim-code function for remote adds | Zero to cents |
| Rendezvous / presence / config | Units behind NAT must find each other and learn changes | Signed per-family record in object storage, or one hibernating Durable Object per family, or MQTT on IoT Core | $0.02 to $0.13 per family per month |
| Offline, battery and lost-device alerts to family phones | Someone must notice that a unit has gone quiet; a silent unit cannot report itself | Scheduled function that checks last-seen times and sends email or push through a notification API | Cents |
| Support diagnostics | Vendor must see why a family's units cannot connect | Units post a small signed health report to object storage or a logging endpoint | Cents |
| Relay fallback for CGNAT-to-CGNAT pairs | Hole punching fails for some NAT pairs | Managed TURN/SFU (Cloudflare, per GB) or a Durable Object | See B2 |

- OTA polling arithmetic: 400,000 units checking a manifest once a day is 12 million reads a month, about 2 million over R2's free Class B allowance, so about $0.72/month; a CDN cache in front removes even that.
- GitHub Releases would work for an open-source hobby build, but for a commercial fleet a vendor-controlled bucket and domain avoids dependence on a third party's rate limits and terms (not sourced; see Gaps).
- The practical reading of the owner's goal: "no relay fleet to maintain" is achievable; "no central services" is not. The smallest honest footprint is static storage plus one serverless function set plus a per-GB managed relay fallback, all on one provider account, with the device protocol kept neutral so the self-run Go relay remains a drop-in fallback.

### Gaps
- GitHub Releases terms and rate limits for device fleets, and push-notification options (APNs/FCM directly, or a managed notification API) and their pricing, were not researched.
- No source found for typical support-contact rates of consumer IoT devices, which would size the diagnostics need.
