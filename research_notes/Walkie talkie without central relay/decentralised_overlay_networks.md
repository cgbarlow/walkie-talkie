# Decentralised / federated overlay networks and public infrastructure an ESP32-S3 push-to-talk device could ride on (as of November 2026)

Reading guide. "Cited Findings" are things a source said during this research session. Where the source was only seen as a search-result summary (the page itself could not be opened because the research sandbox blocked the host: iroh.computer, docs.iroh.computer, tailscale.com, signal.org), the bullet is marked **[snippet]**. Pages actually opened are marked **[fetched]**. "Inferences" are my reasoning; items there marked **[background]** come from general knowledge and were not re-verified this session, so treat them as leads to check, not facts.

Device assumptions used for sizing: Opus 16-24 kbit/s in 20 ms frames = 50 packets/s, roughly 3-5 kB/s on the wire with headers and AEAD tag, about 11-18 MB per hour of continuous talk per listener; one talker at a time, groups of 2-10.

## 1. DHT / P2P stacks (libp2p, iroh, Hyperswarm, OpenDHT/Jami, Tox, mainline DHT, Veilid, Yggdrasil, cjdns)

### Takeaway
iroh is the only stack in this group with a vendor-published ESP32 port (std Rust on ESP-IDF, PSRAM board), but its free public relays are explicitly not for production, so a product would pay n0 or self-host a relay, which is the thing the owner wants to avoid. Every other stack either has no microcontroller client (libp2p, Hyperswarm, OpenDHT/Jami, Veilid, Yggdrasil, cjdns) or only a CI build (c-toxcore), and all of them still depend on somebody's bootstrap nodes and, for peers behind hard NATs, somebody's relay.

### Cited Findings
**iroh (n0)**
- n0 published "Running iroh on an ESP32": iroh needs an OS with TCP/IP and Wi-Fi so it runs on ESP-IDF/FreeRTOS (std Rust, not no_std); a plain `cargo add iroh` did not compile, a special iroh branch was needed "for now" with support in main promised later; the default `ring` crypto provider did not work on ESP32; success reported on an ESP32 with 4 MiB flash and 4 MiB SPIRAM **[snippet]** — [iroh blog](https://iroh.computer/blog/iroh-on-esp32)
- iroh's compatibility docs point to `iroh-esp32-examples` firmware and an `iroh-smart-fan` demo (a PSRAM ESP32 driving a fan over irpc) **[snippet]** — [iroh compatibility docs](https://docs.iroh.computer/compatibility), [smart fan post](https://iroh.computer/blog/an-iroh-powered-smart-fan)
- iroh reached 1.0 in 2026 (a 1.0.2 release post is dated 2026-07-07) **[snippet]** — [iroh 1.0.2](https://iroh.computer/blog/iroh-1-0-2)
- Public relays are free, unauthenticated, rate-limited, "for development, testing, and hobby projects", carry no SLA or uptime guarantee, and officially support only the latest stable iroh release; the docs advise against production use and point to paid Shared Relays, a dedicated relay, or self-hosting **[snippet]** — [public relays](https://docs.iroh.computer/iroh-services/relays/public), [rate limiting](https://docs.iroh.computer/relays/rate-limiting)
- Shared relays are a paid option (a 5 MB/s rate limit is listed among Pro plan terms); self-hosted `iroh-relay` has rate limiting off by default **[snippet]** — [shared relays](https://docs.iroh.computer/iroh-services/relays/shared), [shared relays blog](https://iroh.computer/blog/shared-relays), [managed hosting](https://iroh.computer/services/hosting)

**libp2p**
- The circuit-relay v2 spec defines per-circuit `duration` and `data` limits and reservation expiry but sets no mandatory values; a relay may apply none **[fetched]** — [circuit-v2 spec](https://github.com/libp2p/specs/blob/master/relay/circuit-v2.md)
- libp2p's 2025 annual report says c-libp2p and zig-libp2p "expanded into embedded and constrained environments" and names IoT as a focus **[snippet]** — [libp2p 2025 report](https://libp2p.io/reports/annual-reports/2025/)
- The c-libp2p repository itself describes the project as "still in active development", builds for Linux/macOS/Windows with a C11 compiler, and its README names no microcontroller target and gives no footprint numbers **[fetched]** — [Pier-Two/c-libp2p](https://github.com/Pier-Two/c-libp2p)
- nim-libp2p is getting a C static-library binding (2026 roadmap), not microcontroller-specific **[snippet]** — [VAC roadmap](https://roadmap.vac.dev/p2p/ift/2026q1-nimlibp2p-cbindings)
- `libcp2p` (RTrade) is a "libp2p-like" C project aimed at embedded, self-described work in progress, compatibility with real libp2p only aspirational, apparently stale **[snippet]** — [libcp2p](https://awesome.ecosyste.ms/projects/github.com%2Frtradeltd%2Flibcp2p)

**Tox**
- c-toxcore added an "initial ESP32 docker build" in CI and later changed it to "actually try to instantiate tox"; no evidence found of a working ESP32 Tox device, memory figures or example firmware **[snippet]** — [c-toxcore](https://github.com/TokTok/c-toxcore/blob/master/INSTALL.md), [changelog](https://src.opensuse.org/pool/c-toxcore/raw/branch/factory/c-toxcore.changes)
- Tox bootstrap nodes are a community-run list on the Tox wiki **[snippet]** — [Tox wiki nodes](https://wiki.tox.chat/_export/raw/users/runningnodes)

**OpenDHT / Jami**
- Jami clients use `bootstrap.jami.net:4222` (run by Savoir-faire Linux) as the default, configurable bootstrap node on the public OpenDHT network **[snippet]** — [Jami distributed network](https://docs.jami.net/user/jami-distributed-network.html)
- OpenDHT is a C++17 library; the project pitches it as "communication middleware for embedded systems" but no ESP32 port or footprint figure was found **[snippet]** — [opendht](https://github.com/savoirfairelinux/opendht)

**Hyperswarm / Holepunch (Keet)**
- HyperDHT's default bootstrap servers are "publicly served on behalf of the commons" (by Holepunch); a private DHT is possible with your own persistent node(s). The implementation is JavaScript (npm `hyperdht`, 6.29.x) **[snippet]** — [hyperdht on npm](https://npmjs.com/package/hyperdht)

**BitTorrent mainline DHT (BEP 44)**
- pkarr/Pubky use BEP 44 mutable items on the mainline DHT and cite "15 years of proven reliability and 10+ million active nodes"; values are limited to about 1000 bytes; records are dropped after hours or days unless republished **[snippet]** — [pkarr](https://github.com/pubky/pkarr), [Pubky mainline DHT](https://pubky.org/explore/technologies/mainline-dht/)

**Veilid**
- Veilid core is Rust (v0.5.x by late 2025/2026); the Veilid Foundation itself runs 2 bootstrap nodes and 4 headless nodes **[snippet]** — [Veilid metrics post](https://veilid.com/blog/privacy-respecting-network-metrics), [veilid-core](https://lib.rs/crates/veilid-core)

### Inferences
- iroh on an ESP32-S3 with 8 MB PSRAM is plausible (the n0 demo used a smaller board), and iroh's QUIC datagrams suit 20 ms Opus frames. But it does not remove the relay: about one connection in ten typically cannot hole-punch **[background]** and falls back to a relay, and n0's own terms rule out the free relays for a product. So iroh changes "vendor writes and runs a custom relay" into "vendor pays n0 or runs stock `iroh-relay`"; it is a maintenance reduction, not elimination.
- The mainline DHT is the longest-lived candidate for pure rendezvous (publishing "my current IP:port" under an Ed25519 key in under 1 kB). It is UDP, bencoded and simple enough to hand-write for ESP32, and nobody owns it. It gives discovery only; it does not relay audio, and its bootstrap hostnames (router.bittorrent.com and similar) are run by third parties with no commitment **[background]**.
- Public libp2p (IPFS) relays in practice apply tight defaults (about 2 minutes and 128 KiB per circuit in go-libp2p) intended only to coordinate hole punching **[background]**; the spec fetch confirms limits exist but not these values. They cannot carry a voice session.
- Hyperswarm, Veilid, OpenDHT, Yggdrasil and cjdns are Node.js / Rust-std / C++17 / Go / Linux-class codebases; fitting them on an ESP32 would be a from-scratch port.

### Gaps
- Could not open iroh pages: exact rate-limit figures for public relays, Shared Relay prices, and whether ESP32 support is now in iroh main (post-1.0) are unverified. iroh.computer and docs.iroh.computer need to be allowed in the session network settings to check.
- No measurement found of iroh flash/RAM use or connect time on ESP32-S3.
- No sources found this session on Yggdrasil or cjdns on microcontrollers, on their public-peer policies, or on Keet's 2026 status.
- No ProbeLab-style numbers gathered on hole-punch success rates or DHT lookup latency.

## 2. Anonymity networks (Tor onion services, I2P)

### Takeaway
Tor and I2P are unsuitable: latency is seconds rather than tens of milliseconds for onion-to-onion paths, both are TCP/stream oriented, and no ESP32 client was found.

### Cited Findings
- A 2020 study found most voice calls over single Tor circuits (over 85% of trials) had one-way delay under 400 ms — [arXiv 2007.04673](https://arxiv.org/pdf/2007.04673)
- Onion services use 6-hop circuits and interactive latency on Tor is reported as more than 5x direct paths (secondary summary of a PoPETs 2025 paper) — [PoPETs 2025-0029](https://crysp.petsymposium.org/popets/2025/popets-2025-0029.php)
- "TerminalPhone", a walkie-talkie that sends recorded Opus clips over Tor hidden services, reported roughly 2-3 s delay per chunk and its author called full duplex "absolutely terrible" (forum comment, anecdotal) — [HN discussion](https://hn.svelte.dev/item/47164270)
- Mumble as a hidden service: 1-2 s latency described as normal (2013 anecdote) — [tor-talk](https://lists.torproject.org/pipermail/tor-talk/2013-October/030739.html)
- A search for ESP32 Tor clients returned no port — [search result set, e.g. DarkHorse paper for context](https://arxiv.org/html/2307.02429v1)

### Inferences
- A 2-3 s delay is tolerable for store-and-forward voice messages but poor for live push-to-talk, and the device would have to host an onion service (both ends behind NAT), which needs a full Tor client. C Tor and Arti (Rust) are desktop-class **[background]**.
- Piggybacking a consumer product on a volunteer anonymity network that the product does not need for anonymity is also an acceptable-use concern.

### Gaps
- No I2P voice-latency source found. No authoritative statement found that an ESP32 Tor client does not exist; only absence of search hits.

## 3. Messaging networks as transport (Nostr, Matrix, XMPP, SimpleX, Session, Briar, Signal, Delta Chat/chatmail, Meshtastic MQTT, public MQTT brokers, SSB)

### Takeaway
None is a sound carrier for live audio, and the "free public" servers either forbid or do not promise production use. The best fit in this group is as a signalling or voice-message channel: Nostr has working ESP32 libraries and many independent relays; chatmail/Delta Chat now offers multi-relay failover for store-and-forward messages. Both remain other people's goodwill.

### Cited Findings
**Nostr**
- ESP32 libraries exist: `arduino-nostr` (lnbits; NIP-01 notes and NIP-04 DMs, MIT, ESP32 architecture) and `Nostrduino` (rewrite with more features, NWC) **[snippet]** — [arduino-nostr](https://github.com/lnbits/arduino-nostr/), [Nostrduino](https://awesome.ecosyste.ms/projects/github.com%2Friccardobl%2Fnostrduino)
- No Nostr-based push-to-talk hardware turned up; ESP32 walkie-talkies found all use ESP-NOW (local only) — [example](https://hackaday.io/project/205897-esp32-walkie-talkie)

**Matrix**
- matrix.org updated its Homeserver Terms in July 2025, added fair-usage language and announced progressive introduction of premium accounts; usage limits count media, message contents and HTTPS transfer **[snippet]** — [terms update](https://matrix.org/blog/2025/07/terms-update/), [homeserver pricing](https://matrix.org/homeserver/pricing/), [terms](https://matrix.org/legal/terms-and-conditions/)
- matrix.org's client directory lists no ESP32 client; MatrixRTC relies on to-device key sharing plus an SFU back end **[snippet]** — [clients](https://matrix.org/ecosystem/clients/), [MatrixRTC talk](https://fosdem.org/schedule/event/fosdem-2025-5034-matrixrtc-building-real-time-applications-on-matrix)

**XMPP**
- Public XMPP servers are volunteer-run; operators' main concern is bot spam and many gate registration with CAPTCHAs **[snippet]** — [Prosody public servers](https://www.prosody.im/doc/public_servers), [xmpp.party AUP](https://xmpp.party/tos-aup)

**SimpleX**
- Preset SMP relays are provided "as is", experimental, with no guarantee of availability or data retention; users can choose other operators or their own servers **[snippet]** — [SimpleX operators conditions (mirror)](https://forgejo.multed.com/android-apps-mirrors/SimpleX-Chat/blame/commit/50232fd1799f05c9db4f2fce917f797635e595d4/PRIVACY.md)

**Signal**
- Community sources say Signal does not allow third-party clients on its servers; one commenter could not find that wording in the terms. Unverified against Signal's own text **[snippet]** — [Privacy Guides thread](https://discuss.privacyguides.net/t/delta-chat-email-client/18239/29)

**Delta Chat / chatmail**
- November 2026: Delta Chat dropped unencrypted email; new profiles join the fastest-responding community-run chatmail relay and add two more in the background (three total), fetching from all and failing over on send **[snippet]** — [AlternativeTo news](https://alternativeto.net/news/2026/11/delta-chat-drops-unencrypted-email-adds-decentralized-onboarding-across-community-relays/)
- Chatmail relay software is FOSS and documented for self-hosting **[snippet]** — [chatmail relay docs](https://chatmail.at/doc/relay/)

**Meshtastic public MQTT**
- The project-run public server enforces a zero-hop policy and reduced location precision, and on 21 August 2024 removed the ability to subscribe to all topics after map data was observed being stored and tracked **[snippet]** — [Meshtastic MQTT docs](https://meshtastic.org/docs/software/integrations/mqtt/), [Notebookcheck](https://www.notebookcheck.net/Enhancements-coming-soon-to-Meshtastic-s-public-server.862237.0.html)

**Public MQTT brokers**
- HiveMQ says its public broker is public and shared, "not intended for private or production data", for quick temporary testing **[snippet]** — [HiveMQ public broker](https://www.hivemq.com/public-mqtt-broker/)
- Community listings: public brokers (test.mosquitto.org, broker.emqx.io) are for testing; any device can publish/subscribe to any topic **[snippet]** — [mqtt.org public brokers wiki](https://github.com/mqtt/mqtt.org/wiki/public_brokers), [EMQX comparison](https://www.emqx.com/en/blog/popular-online-public-mqtt-brokers)

### Inferences
- Nostr ephemeral events (kinds 20000-29999, not stored by relays) **[background]** over one WebSocket/TLS connection per relay would work as presence and signalling (publish a sealed "here is my address" blob to 3-5 relays). Sending 50 events/s of audio would be rate-limited or banned on most public relays and each event needs a Schnorr signature, which is heavy at 50/s on an ESP32. Sending a whole 5-30 s clip as one or a few events (tens of kB base64) exceeds typical relay message-size limits; clips would need a blob host (Blossom-type) **[background]**.
- All these transports are TCP+TLS (WebSocket, HTTPS, SMTP/IMAP, MQTT). Head-of-line blocking makes them poor for 20 ms frames but fine for "record, seal, send, play" voice messages with roughly 0.5-3 s latency. That matches the product's "last message kept 24 h" feature better than its live path.
- chatmail is the most interesting store-and-forward option: independent operators, automatic three-relay failover, E2EE mandatory. A device would need SMTP submit + IMAP IDLE over TLS and an Autocrypt/OpenPGP-compatible envelope, which is feasible on ESP32-S3 but a substantial port (Delta Chat core is Rust std) **[background]**. Relay operators' willingness to carry thousands of commercial devices is unknown.
- Matrix needs an account per device on some homeserver; matrix.org is moving toward metered/paid use, so a fleet would be exactly the "fair usage" case. Live audio in Matrix needs a LiveKit SFU that someone must host.
- Briar (Tor-based, Android), Session (own onion network, clients are desktop/mobile) and Secure Scuttlebutt (append-only logs, pub servers) have no embedded clients and no live-audio path **[background]**.

### Gaps
- Signal's terms could not be opened (signal.org blocked in sandbox); the "official apps only" restriction is unverified here.
- No official terms pages retrieved for test.mosquitto.org or broker.emqx.io.
- No source found on chatmail relay rate limits, message-size caps or delivery latency; none on Session/Lokinet, Briar or SSB status in 2026.
- No source retrieved for Nostr relay rate limits or NIP numbers for voice messages.

## 4. Mesh VPNs (Tailscale/DERP, Headscale, ZeroTier, Nebula, NetBird, WireGuard + rendezvous)

### Takeaway
A third-party, MIT-licensed Tailscale client for ESP32-S3 (MicroLink) exists and measures 30-150 ms round trip via DERP and 5-50 ms direct over Wi-Fi, which is good enough for push-to-talk. But the relays and control plane belong to Tailscale Inc., the client is unofficial, and a product would need a commercial arrangement; ZeroTier explicitly requires a commercial licence for devices built on it. Self-hosted equivalents (Headscale, Nebula lighthouse, NetBird) put a server back on the vendor.

### Cited Findings
**Tailscale on ESP32**
- MicroLink v2: ESP32-S3 with 8 MB PSRAM recommended; Wi-Fi build uses 116 KB SRAM and 950 KB flash statically, 42 KB of task stacks, about 1 MB PSRAM peak during coordination polling; implements ts2021 control protocol, WireGuard (ChaCha20-Poly1305), DISCO, DERP (up to 32 regions), STUN, MagicDNS; works with Headscale and Ionscale; MIT; "independent implementation … not affiliated with or endorsed by Tailscale Inc." **[fetched]** — [CamM2325/microlink](https://github.com/CamM2325/microlink)
- MicroLink's own table: boot to connected about 15-20 s on Wi-Fi; round trip via DERP 30-150 ms, direct 5-50 ms; symmetric NAT can block direct paths; default cap of 16 active WireGuard tunnels; the device must connect to its advertised preferred DERP region **[fetched]** — [CamM2325/microlink](https://github.com/CamM2325/microlink)
- Forks exist in the ESP Component Registry (`fugo101/microlink`, requires ESP-IDF 6.x; `gmo-pepabo/microlink`) and as an ESPHome component **[snippet]** — [fugo101/microlink](https://components.espressif.com/components/fugo101/microlink/versions/3.2.2/readme), [esphome-tailscale](https://github.com/Csontikka/esphome-tailscale), [Hackaday, May 2026](https://hackaday.com/2026/05/18/running-a-vpn-gateway-on-an-esp32/)
- Independent test on M5Stack CoreS3: a WAV download took about 9 s via DERP and 2.4 s after direct UDP was established **[snippet]** — [lilting.ch DERP](https://lilting.ch/en/articles/m5stack-cores3-tailscale-direct-voice-server), [lilting.ch direct](https://lilting.ch/en/articles/m5stack-cores3-tailscale-disco-direct-udp)

**Tailscale relays and plans**
- Tailscale docs: DERP servers "are reliable but have limited quality of service (QoS) characteristics", are slower than direct connections and may offer lower throughput; they forward only encrypted traffic **[snippet]** — [connection types](https://tailscale.com/docs/reference/connection-types), [DERP servers](https://tailscale.com/docs/reference/derp-servers)
- Tailscale Peer Relays (beta October 2025) let a customer's own node relay; fallback is DERP **[snippet]** — [peer relays](https://tailscale.com/docs/features/peer-relay), [blog](https://tailscale.com/blog/peer-relays-beta)
- Third-party summaries: in April 2026 the free Personal plan went from 3 to 6 users and paid plans bill per seat (Standard about $8, Premium about $18 per user/month); one aggregator describes the free plan as non-commercial. Not checked against Tailscale's own pages **[snippet]** — [costbench](https://costbench.com/software/ztna/tailscale-ztna/free-plan/), [activepieces](https://www.activepieces.com/blog/tailscale-pricing-plans-costs-and-features-2026)

**ZeroTier**
- ZeroTier and libzt are under BSL 1.1; free for internal business, academic and non-commercial use; "building closed-source apps and devices based on ZeroTier" requires a commercial licence; a newer release note allows embedding for "the home user" only **[snippet]** — [libzt licence](https://www.github.com/zerotier/libzt/blob/main/LICENSE.txt), [ZeroTier forum](https://discuss.zerotier.com/t/licensing-and-commercial-use/8310), [ZeroTier 1.16](https://www.zerotier.com/?p=6818)

**Nebula**
- Nebula (built at Slack, maintained by Defined Networking) is open source; Defined's managed service is free up to 100 hosts, then about $1 per host per month **[snippet]** — [Defined pricing](https://defined.net/pricing/), [comparison page](https://www.defined.net/compare/nebula-vs-netbird)

**WireGuard on ESP32**
- `trombik/esp_wireguard` is an ESP-IDF component (esp32, s2, c3) based on WireGuard for lwIP, self-described alpha; a NATed peer must use persistent keepalive; plain WireGuard has no rendezvous or relay of its own **[snippet]** — [esp_wireguard](https://components.espressif.com/components/trombik/esp_wireguard)

### Inferences
- Tailscale is the technically strongest "someone else runs the relay" option: UDP direct path when possible, global DERP fleet otherwise, and a working ESP32-S3 client whose memory use leaves room for Opus and audio. DERP is TCP/TLS, so relayed audio suffers head-of-line blocking on lossy Wi-Fi; a jitter buffer of 100-200 ms would be needed.
- The business problem is unchanged in kind: each family would need a tailnet (their own Tailscale account, which is a setup hurdle for this audience) or the vendor would own one big tailnet and pay per device/seat, and MicroLink speaks a reverse-engineered control protocol that Tailscale could break or block. Fifteen to twenty seconds from boot to connected is also slow for an appliance.
- "Bring your own tailnet / Headscale" could be offered as an advanced option at zero vendor cost.
- Nebula lighthouses, NetBird relays and Headscale are all servers the vendor (or user) must run; they only swap one small Go relay for another. No ESP32 Nebula, NetBird or ZeroTier client was found.
- WireGuard + public STUN + a rendezvous record (DHT or Nostr) is a credible relay-less design for the roughly 80-90% of home NAT pairs that allow UDP hole punching **[background]**; the remainder needs TURN or a relay.

### Gaps
- Tailscale's terms and pricing pages could not be opened (tailscale.com blocked in sandbox): whether embedding in a sold device is permitted, device-count pricing for tagged/IoT nodes, and any DERP fair-use clause are unverified.
- ZeroTier 2026 prices and whether ZeroTier's roots are usable by a self-built minimal client were not found. No sources on NetBird's hosted relay terms or Headscale on embedded.
- MicroLink's maintenance status (release dates, bus factor) is unknown; the GitHub page shows no tagged releases.

## 5. Public STUN/TURN

### Takeaway
Public STUN is free and adequate for discovering the device's public address; TURN is the one relay function that can be bought as a commodity. Cloudflare's 1,000 GB/month free TURN allowance, then $0.05/GB, would cover a very large fleet's fallback traffic, but it needs a Cloudflare account and a credential-minting endpoint, so it is "vendor-paid utility", not "no vendor infrastructure".

### Cited Findings
- Cloudflare Realtime: first 1,000 GB per month free, shared between SFU and TURN; then $0.05 per GB; TURN is billed on data sent from Cloudflare's edge to the TURN client, ingress is not billed **[snippet]** — [Cloudflare Realtime pricing](https://developers.cloudflare.com/realtime/pricing/), [TURN FAQ](https://developers.cloudflare.com/realtime/turn/faq/)
- Cloudflare's STUN at stun.cloudflare.com is free with no usage limit **[snippet]** — [TURN FAQ](https://developers.cloudflare.com/realtime/turn/faq/)
- Metered's Open Relay offers 20 GB/month of free TURN; it now requires signing up for a free account and API key "to prevent abuse" (older static shared credentials no longer apply); no formal SLA found for the free tier **[snippet]** — [Open Relay](https://metered.ca/tools/openrelay/)

### Inferences
- At about 15 MB per relayed talk-hour per listener, 1,000 GB is on the order of 60,000 relayed listener-hours a month. If only 10-20% of sessions need relay and a household talks minutes per day, the free tier plausibly covers tens of thousands of devices; above it, $0.05/GB is under a tenth of a cent per relayed hour.
- TURN needs short-lived credentials issued from a secret API token. That token cannot ship in firmware, so the vendor still needs a tiny stateless credential service (a Cloudflare Worker would do). This is far less to operate than a relay, but it is a central piece and a vendor dependency on Cloudflare's pricing.
- TURN on ESP32 means a small STUN/TURN client (UDP allocate, permissions, channel data); libraries such as libjuice/libpeer have been used on ESP32 **[background]**.
- Google's stun.l.google.com is widely used but undocumented and unsupported **[background]**; ship a list of several STUN servers and make it updatable by OTA.

### Gaps
- No Google statement on STUN terms or reliability found. Cloudflare's terms on using TURN outside WebRTC/browser contexts were not checked. Open Relay's rules on commercial fleets were not checked.

## 6. What central piece remains, who maintains it, and track records

### Takeaway
No surveyed option removes every central dependency: each needs bootstrap/rendezvous (someone's hostname), a relay for hard NATs, and the vendor's own OTA/update host. The realistic choice is which party carries the relay: a paid commodity (Cloudflare TURN, n0 relays, Tailscale), volunteers (Nostr relays, DHT nodes, chatmail), or the customer (a home hub or self-hosted relay).

### Cited Findings
- Mainline DHT: about 15 years in operation and roughly 10 million nodes (as cited by pkarr/Pubky) **[snippet]** — [Pubky mainline DHT](https://pubky.org/explore/technologies/mainline-dht/)
- Bootstrap dependence by project: Jami on `bootstrap.jami.net` (Savoir-faire Linux); HyperDHT on Holepunch-served defaults; Veilid on 2 Foundation bootstrap nodes; Tox on a community wiki list **[snippet]** — [Jami](https://docs.jami.net/user/jami-distributed-network.html), [hyperdht](https://npmjs.com/package/hyperdht), [Veilid](https://veilid.com/blog/privacy-respecting-network-metrics), [Tox wiki](https://wiki.tox.chat/_export/raw/users/runningnodes)
- Terms changing under users: matrix.org added fair-use limits and premium accounts in 2025; Meshtastic restricted its public MQTT server in 2024 after abuse; Metered's Open Relay moved from open credentials to account-gated; iroh public relays are rate-limited and support only the latest release **[snippet]** — [Matrix terms update](https://matrix.org/blog/2025/07/terms-update/), [Meshtastic](https://www.notebookcheck.net/Enhancements-coming-soon-to-Meshtastic-s-public-server.862237.0.html), [Open Relay](https://metered.ca/tools/openrelay/), [iroh public relays](https://docs.iroh.computer/iroh-services/relays/public)
- Free services withdrawn: Mozilla shut Firefox Send in 2020 after malware abuse and cost review; AWS closed the free Wickr Me on 31 December 2023 to focus on paid products — [Mozilla](https://blog.mozilla.org/en/products/firefox/firefox-news/update-on-firefox-send-and-firefox-notes/), [Computerworld](https://www.computerworld.com/article/3680554/amazon-shuts-wickr-me-encrypted-messaging-service-enterprise-service-remains.html)

### Inferences
Summary table (my synthesis of the sections above; "ESP32 client" reflects what was found this session).

| Option | Live PTT fit | ESP32 client | Who runs infra | Commercial piggyback | Vendor still runs |
|---|---|---|---|---|---|
| iroh + n0 relays | Good (QUIC/UDP, relay fallback) | Vendor demo, std Rust, PSRAM | n0 (public/paid) or self | Public relays: not for production; paid plans | Paid relay plan or own `iroh-relay`; OTA |
| Tailscale (MicroLink) | Good direct, fair via DERP (TCP) | Unofficial, MIT, S3 + 8 MB PSRAM | Tailscale Inc. | Unverified; likely needs paid/commercial deal | Tailnet admin or per-user accounts; OTA |
| WireGuard/own UDP + STUN + Cloudflare TURN | Good | STUN/TURN client to write; WireGuard alpha component | Cloudflare (paid utility) | Yes, priced | Credential minting function; rendezvous; OTA |
| Mainline DHT (BEP 44) rendezvous | Discovery only | None found; simple to write | Nobody / everyone | No terms; background noise-level load | Nothing for discovery; relay problem remains |
| Nostr relays | Signalling and short clips only | Yes (arduino-nostr, Nostrduino) | Many independent operators | Per-relay; audio streaming likely unwelcome | Ideally one own relay as anchor; OTA |
| chatmail / Delta Chat | Voice messages only (seconds) | None found | Community relay operators | Unknown for device fleets | Nothing in principle; OTA |
| Matrix / XMPP public servers | Signalling only; live needs SFU | None found | Volunteer or foundation servers | matrix.org now fair-use/premium | Accounts, likely own homeserver |
| Public MQTT brokers | Works technically, TCP | Yes (ESP-MQTT) **[background]** | HiveMQ, EMQX, Eclipse | Explicitly testing only | Would end up running a broker |
| libp2p / IPFS relays | Relay limits too tight | No MCU port found | Protocol Labs ecosystem | Relays meant for hole-punch coordination | Own relay |
| Tor / I2P | Poor (seconds) | None | Volunteers | Poor fit | — |
| Tox, Jami/OpenDHT, Hyperswarm, Veilid, Yggdrasil, cjdns, ZeroTier, Session, Briar, SSB | Varies | None usable found | Project or community | ZeroTier needs commercial licence | Port + bootstrap risk |

- Longevity ranking by my judgement **[background]**: SMTP/IMAP email, XMPP (1999 on) and the BitTorrent DHT (2005 on) have 15-25 year records; Tor (2002 on) and STUN/TURN as standards likewise. Matrix (2014), Nostr (2020-21), iroh (1.0 in 2026), Veilid (2023), chatmail (2023-24) are younger and either VC/foundation funded or volunteer-run. Tox and SSB have shrunk; a product should not bind to them.
- The OTA server, firmware signing and a pairing/onboarding flow remain with the vendor in every option. They are static-file hosting (CDN or GitHub Releases), not a live service.
- A layered design is the logical outcome: (1) same-LAN direct; (2) UDP hole punching using public STUN plus a rendezvous record on infrastructure nobody owns (mainline DHT) or many own (several Nostr relays); (3) fallback relay chosen per group from a list: commodity TURN, a family member's always-on device acting as relay (the Tailscale "peer relay" idea), or a self-hostable relay; (4) store-and-forward clips over the same rendezvous channel when live fails.

### Gaps
- No uptime or churn measurements were gathered for Nostr relays, DHT bootstrap hosts or public STUN servers.
- Nothing found on legal exposure (for example relay operators blocking a vendor's device fleet) beyond the terms quoted.

## 7. Precedents: shipped hardware that depends on public decentralised networks

### Takeaway
I found no shipped consumer voice device that relies on a public decentralised overlay for connectivity. The nearest precedents show two patterns: products that stay healthy because the network is the users' own radios (Meshtastic), and shared free back ends that had to be restricted once usage or abuse grew (Meshtastic's MQTT server, Helium's economics).

### Cited Findings
- Meshtastic devices work device-to-device over LoRa; the project's shared public MQTT bridge had to add zero-hop and location-precision limits and, in August 2024, remove wildcard subscriptions after data harvesting — [Meshtastic MQTT docs](https://meshtastic.org/docs/software/integrations/mqtt/), [Notebookcheck](https://www.notebookcheck.net/Enhancements-coming-soon-to-Meshtastic-s-public-server.862237.0.html)
- Helium: hotspot hardware is made by third parties, not the network's developer; measurement work found over half of hotspots relied on another hotspot to relay data and backhaul concentrated in ISPs; investors and press questioned low network revenue relative to growth **[snippet]** — [APNIC measurement post](https://blog.apnic.net/2022/02/09/measuring-a-decentralized-network/), [Blockworks](https://blockworks.co/news/where-is-the-revenue-helium-investors-inquire), [Fortune 2022](https://fortune.com/2022/09/27/embattled-helium-attempting-crypto-powered-mobile-network)
- Keet (Holepunch) is an app, not hardware; it relays through other call participants and depends on Holepunch-served DHT bootstrap nodes **[snippet]** — [forum summary](https://tianpan.co/forum/t/holepunchs-p2p-stack-building-unstoppable-apps-without-infrastructure/677), [hyperdht](https://npmjs.com/package/hyperdht)
- iroh's ESP32 smart-fan demo and the MicroLink ESPHome integrations are hobby/demo projects, not shipped products **[snippet]** — [iroh smart fan](https://iroh.computer/blog/an-iroh-powered-smart-fan), [esphome-tailscale](https://github.com/Csontikka/esphome-tailscale)
- A November 2026-era search for Nostr voice hardware found none — [search hit showing only ESP-NOW walkie-talkies](https://www.hackster.io/pokibon/esp32talkie-a-license-free-wi-fi-walkie-talkie-on-m5sticks3-e81d02)

### Inferences
- Products that survive on third-party networks either contribute capacity to them (each Meshtastic node is the network) or use them only for low-volume signalling. A fleet that only consumes relay bandwidth from volunteers is the pattern that gets rate-limited.
- Home-server products such as Umbrel/Start9 use Tor onion services or Tailscale for remote access **[background]**; that suits occasional admin access, not audio, and they are user-run boxes, which hints at a "home hub as family relay" option for this product.

### Gaps
- No sourced post-mortems found for hardware bricked by a decentralised network changing terms; no 2026 status found for Keet, Umbrel, Briar hardware or Nostr signing devices in a connectivity role.
- Sandbox network limits: iroh.computer, docs.iroh.computer, tailscale.com, signal.org and hn.jakechampion.name could not be opened, so claims from those hosts rest on search-result summaries only.
