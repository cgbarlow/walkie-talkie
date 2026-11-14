# Off-internet radio options (Meshtastic/LoRa and alternatives) for push-to-talk voice between NZ homes — as of November 2026

Researcher caveats that apply to the whole file:
- Page fetching was blocked in this session for meshtastic.org, rsm.govt.nz, gazette.govt.nz, meshtastic.discourse.group, rda.sliit.lk and meshed.kiwi. Findings from those hosts come from search-result extracts, not full-page reads, and are marked "(search extract)". The exact current NZ GURL table rows (433 MHz, 2.4 GHz, PRS conditions) could NOT be read first-hand and must be checked against the Gazette text before any design or commercial decision.
- Airtime figures labelled "computed" are my own calculations from the standard Semtech LoRa time-on-air formula, not sourced measurements.
- Unit costs, power draw and module prices were not researched from sources; see Gaps.

## 1. Meshtastic: can it carry voice at all?

### Takeaway
Not live voice. Meshtastic on sub-GHz LoRa is a ~1 kbit/s (LongFast) to ~22 kbit/s (ShortTurbo) shared text mesh; its only built-in audio feature is an experimental Codec2 module documented for SX128x 2.4 GHz radios. Short store-and-forward Codec2 voice clips are feasible (a third-party project, Voicetastic, does exactly this), but a 10 s clip takes roughly 20-30 s to deliver on the default LongFast preset and only becomes "a few seconds" on short-range presets that give up most of the range.

### Cited Findings
- The Meshtastic audio module is documented as for a device "that has a SX128x radio and operates on the 2.4 GHz ISM band", needs an I2S amp (MAX98357) and I2S mic, is "designed for channels with 2 kbit/sec of bandwidth or greater", and Codec2 is off by default (bitrate options such as CODEC2_1400) (search extract) — [Meshtastic audio module docs](https://meshtastic.org/docs/configuration/module/audio/)
- Default preset LongFast = 250 kHz bandwidth, SF11, about 1.07 kbps (search extract) — [Meshtastic blog: why your mesh should switch from LongFast](https://meshtastic.org/blog/why-your-mesh-should-switch-from-longfast/)
- ShortTurbo = SF7, CR 4/5, 500 kHz, 21,880 bps (third-party table citing Meshtastic docs) — [D-Central preset table](https://d-central.tech/meshtastic-modem-presets/)
- Meshtastic's published data rates are theoretical maxima that do not account for headers, hops or retransmissions (search extract) — [Meshtastic radio settings](https://meshtastic.org/docs/overview/radio-settings)
- Default hop limit is 3 (search extract) — [Meshtastic Android settings docs](https://meshtastic.org/docs/software/android/user/settings-radio-user/)
- A 2026 conference paper states Meshtastic "does not natively support voice" and is limited by low data rates and regulatory duty-cycle requirements (search extract) — [SLIIT repository paper](https://rda.sliit.lk/bitstreams/54cca157-cce6-4a2a-92d4-806014bdfc38/download)
- Voicetastic is "voice messaging over the Meshtastic mesh": whole voice messages sent as a unit with FEC and NACK recovery (3 s NACK window), codecs carried include Codec2 (3.2 down to 1.2 kbps, default 1.2 kbps), AMR-NB (4.75-12.2 kbps), Opus (6-64 kbps advisory). Chunk sizes by range class: short 215 B, medium 160 B, long 96 B, very long 48 B. Inter-packet pacing: SHORT_TURBO 150 ms, SHORT_FAST 250 ms, MEDIUM_FAST 500 ms, LONG_FAST 900 ms, LONG_SLOW 1800 ms, VERY_LONG_SLOW 3000 ms. Protocol v3; wiki last edited 13 June 2026. The page gives no end-to-end latency figure and does not describe multi-hop behaviour or live streaming — [Voicetastic Voice Protocol wiki](https://github.com/voicetastic/voicetastic-core/wiki/Voice-Protocol)
- A long-running Meshtastic forum thread on PTT voice exists, and a forum post notes SX1280 gives "higher data rate at cost of lower range" and 2.4 GHz "may not have duty cycle limitations in some countries" (search extract) — [PTT Voice Transmission thread](https://meshtastic.discourse.group/t/ptt-voice-transmission/356); [2.4 GHz voice thread](https://meshtastic.discourse.group/t/voice-communication-with-2-4ghz-net-possible/11406)
- Wellington's community mesh (LongFast, slot 20) reported >150 nodes and was "becoming unusably congested", prompting a planned move to a faster preset (search extract; date not confirmed) — [Wellington region thread](https://meshtastic.discourse.group/t/wellington-region-new-zealand/10171)

### Inferences
- Computed clip delivery times, single hop, no retries, using Voicetastic's chunk sizes and pacing plus computed LoRa time-on-air (16-symbol preamble, ~24 B header overhead assumed):

  | Preset | Packet airtime | 10 s clip @ Codec2 700 (875 B) | 10 s @ 1200 (1500 B) | 10 s @ 3200 (4000 B) | 5 s @ 1200 |
  |---|---|---|---|---|---|
  | LongFast (SF11/250k), 96 B chunks | ~1.13 s | 10 pkts, ~20 s | 16 pkts, ~33 s | 42 pkts, ~85 s | ~16 s |
  | MediumFast (SF9/250k), 160 B | ~0.49 s | ~6 s | ~10 s | ~25 s | ~5 s |
  | ShortFast (SF7/250k), 215 B | ~0.19 s | ~2 s | ~3 s | ~8 s | ~2 s |
  | ShortTurbo (SF7/500k), 215 B | ~0.10 s | ~1 s | ~2 s | ~5 s | ~1 s |

  Each extra hop roughly adds the same again (the relay must receive then re-send every packet on the same channel), and any lost chunk adds a 3 s NACK round.
- Live voice needs the channel continuously: Codec2 1200 bps already exceeds LongFast's ~1.07 kbps raw rate before headers, so live PTT on the default preset is arithmetically impossible, not just impractical. On ShortTurbo a single hop could carry Codec2 in near real time, but it is a different mesh from the one the community runs and has far less link budget.
- Putting 16-42 back-to-back packets on a shared community channel for one voice clip is the behaviour that congested meshes are trying to eliminate; the >150-node Wellington congestion report makes "ride the public mesh with voice" both unreliable and antisocial.
- Codec2 at 700-1200 bps is intelligible to normal-hearing listeners in good conditions but is a vocoder with a robotic timbre; for an elderly listener with hearing loss this is a material quality risk compared with the product's current 16-24 kbit/s Opus. I found no study on Codec2 intelligibility for hearing-impaired listeners (see Gaps).

### Gaps
- Meshtastic maximum payload size and maximum hop limit were not confirmed from the docs (fetch blocked). My recollection is ~233 bytes payload and max 7 hops; treat as unverified.
- Whether the Codec2 audio module is still shipped/maintained in 2026 firmware, or has been removed, was not confirmed; search only showed it documented as an SX128x feature with no recent release news.
- No measured end-to-end latency for Voicetastic or any voice-over-Meshtastic project was found.

## 2. Alternatives on LoRa / sub-GHz and other radios

### Takeaway
The only stack found that offers genuine real-time voice over LoRa is Reticulum's LXST (Codec2 700-3200 bps, Opus from ~4.5 kbps), and its own ecosystem only reports it working over a single LoRa hop at a "reasonable" bitrate. Purpose-built ESP32 + SX1262 Codec2 walkie-talkies exist as hobby projects and one LILYGO kit, proving point-to-point low-bitrate digital voice is buildable, but none carries Opus-quality audio, and none solves next-town range without elevated repeaters. Sidewalk is not in NZ; satellite direct-to-cell is a phone feature, not something an ESP32 can use.

### Cited Findings
- **Reticulum / LXST / Sideband:** LXST provides Codec2 "for very low bandwidth, providing intelligible voice between 700 bps and 3200 bps" and Opus profiles from ~4.5 kbps to ~96 kbps, and can switch codecs mid-stream; it ships an `rnphone` telephony program — [LXST on PyPI](https://pypi.org/project/lxst/0.4.2/); [LXST repo mirror](https://lavaforge.org/Reticulum-Things/LXST)
- Sideband 2.3.0 incorporates RNS 1.6.0, LXMF 1.3.0 and LXST 0.6.0, with voice calls working down to Android 8.1 — [Sideband 2.3.0 release notes](https://newreleases.io/project/github/markqvist/Sideband/release/2.3.0)
- The Reticulum MeshChat developer reports testing two-way audio calls over LoRa "with a single hop", working well "when a reasonable bitrate is configured on the RNode" (no numeric definition given) — [reticulum-meshchat README](https://github.com/markqvist/reticulum-meshchat)
- Ratspeak (another Reticulum client) offers experimental LXST voice calls limited to contacts-only, 0-hop — [Ratspeak](https://upd.dev/ratspeak/Ratspeak)
- **MeshCore:** a lightweight LoRa messaging alternative to Meshtastic — [CNX Software, July 2025](https://www.cnx-software.com/2025/07/19/meshcore-lightweight-alternative-to-meshtastic-for-lora-based-off-grid-messaging/). No official MeshCore voice feature was found; third-party clients advertise Codec2 voice *clips* (MeshCore SAR, posted March 2026; Meck "Voice Notes Over LoRa"; Sestriere) — [MeshCore SAR listing](https://hnshowcase.com/post/47246525/meshcore-sar-voice-maps-and-messaging-without-cell-coverage); [Meck wiki](https://github.com/pelgraine/Meck/wiki); [MeshCore FAQ](https://docs.meshcore.io/faq/?h=)
- **Purpose-built LoRa/FSK Codec2 walkie-talkies:** LILYGO T3-S3 MVSR is an ESP32-S3 board with SX1262 or SX1280, mic and speaker; its SX126x walkie-talkie sketch uses Codec2 at 3200 bps, 8 kHz 16-bit audio, and **FSK modulation** (not LoRa) at 914.6 MHz — [CNX Software, March 2025](https://www.cnx-software.com/2025/03/28/a-lora-based-walkie-talkie-meet-lilygo-t3-s3-mvsr-lora-voice-communication-kit/)
- sh123/esp32_loradv is an ESP32 handheld supporting Codec2 (700-3200 bps) and Opus, with LoRa or FSK modulation, built on E22 (SX1262/SX1268-class) modules — [esp32_loradv](https://github.com/sh123/esp32_loradv)
- dudmuck/lora_codec2 (STM32 + SX1262) tabulates Codec2 modes against LoRa SF/bandwidth and notes "latency across the radio link is due to LoRa packet duration" — [lora_codec2 readme](https://github.com/dudmuck/lora_codec2/blob/master/readme.md)
- LoRaTalk: experimental ESP32 + SX1262 Codec2 walkie-talkie with a browser PTT interface; no published performance — [LoRaTalk](https://github.com/codeWithMaaz-05/LoRaTalk-WalkieTalkie)
- **Wi-Fi HaLow (802.11ah):** industry sources put Australia/NZ HaLow operation in 915-928 MHz; an Australian trade article reports a 16 km HaLow link demonstration — [ACS Information Age, 2025](https://ia.acs.org.au/article/2025/how-to-connect-to-wi-fi-from-16km-away.html); [Wi-Fi Alliance HaLow overview](https://20524844.fs1.hubspotusercontent-na2.net/hubfs/20524844/Wi-Fi_CERTIFIED_HaLow_Technology_Overview_20211102.pdf). An NZ regulator-related slide deck notes NZ's band is 915-928 MHz (not the US 902-928) with a stricter out-of-band limit than Australia — [ISPANZ 2016 RSM slides](https://www.ispanz.org.nz/wp-content/uploads/2021/12/Remote-Spectrum-Management-May2016.pdf)
- **Amazon Sidewalk:** expanded beyond the US only to Canada (26 Feb 2026) and Mexico; EU "planned for 2026"; outside the US live testing is limited. No source lists New Zealand — [iPhone in Canada](https://www.iphoneincanada.ca/2026/01/28/amazon-sidewalk-arrives-in-canada-feb-26/); [Ring Sidewalk information](https://ring.com/support/articles/2a1ku/Amazon-Sidewalk-Information); [Semtech Sidewalk FAQ](https://www.semtech.com/amazon-sidewalk-faqs)
- **Satellite direct-to-device:** One NZ announced in February 2026 satellite data for select apps and voice calling via WhatsApp for eligible customers (iPhone 13 and later initially), needing line of sight to the sky; Spark launched a Starlink-powered service on 1 April 2026 — [One NZ media release](https://media.one.nz/satellite-data-whatsapp-calling); [Bill Bennett analysis](https://billbennett.co.nz/analysis-comparing-new-zealand-carriers-satellite-to-mobile/); a Starlink-focused blog says native voice calls are not commercially available as of Sept 2026 (conflicts in wording; One NZ's is app-based calling over the data service) — [starlink-news.com](https://starlink-news.com/2026/09/19/starlink-direct-to-cell-how-satellite-to-phone-works/)

### Inferences
- Honest verdict table (inference from the findings above plus general radio engineering; "live" = sub-second PTT):

  | Option | Live voice? | Quality for hard-of-hearing listener | Next-town range | Zero central infra? | Verdict |
  |---|---|---|---|---|---|
  | Meshtastic (stock, sub-GHz) | No | n/a | Only via community repeaters | Yes (but depends on others' nodes) | Text/alerts only; voice clips marginal |
  | Meshtastic + Voicetastic clips | No (10-30+ s per clip on LongFast) | Poor (Codec2 1.2k default) | As above, worse per hop | Yes | Possible "voicemail" fallback, not the product |
  | MeshCore | No (third-party clips only) | Poor | Needs repeater network | Yes | Same class as Meshtastic |
  | Reticulum + LXST over RNode | Yes, single hop, Codec2 | Poor-fair (Codec2); Opus needs >4.5 kbps link | Same-neighbourhood line-of-sight; multi-hop unproven | Yes | Best "true mesh voice" candidate, but hobby-grade; heavy stack (Python reference implementation) for an ESP32 product |
  | Custom ESP32 + SX1262 Codec2 (LoRa or FSK) | Yes, point-to-point | Fair at 3200 bps | Street/suburb scale indoor-to-indoor; no | Yes | Buildable same-neighbourhood link; fails "next town" |
  | SX1280 2.4 GHz LoRa/FLRC | Yes (higher bitrate) | Could carry low-rate Opus | Worse than sub-GHz through walls | Yes | Only for line-of-sight neighbours |
  | ESP-NOW / ESP-NOW LR | Yes (Wi-Fi-class rates) | Good (Opus) | Tens to a few hundred metres | Yes | Free on existing hardware; next-door only |
  | Wi-Fi HaLow | Yes (hundreds of kbps+) | Good (Opus) | Best sub-GHz candidate for km-scale with outdoor antennas | Yes | Technically the strongest; cost and NZ certification are the obstacles |
  | SA818/SA868 analogue FM on PRS | Yes | Good (analogue FM) | 1-5 km handheld-class | Yes | Works like a real walkie-talkie but no encryption/privacy, open channel, and a custom transmitter needs PRS type approval |
  | Amateur radio (M17, etc.) | Yes | Good | Excellent with repeaters | Yes | Every user needs a licence; encryption prohibited: non-starter for grandad |
  | LoRaWAN / Helium | No | n/a | n/a | No (gateways + network server) | Not a voice bearer |
  | NB-IoT / LTE-M / LTE Cat-1 | LTE-M/Cat-1 yes; NB-IoT no | Good | Nationwide | No: swaps your relay for a carrier, and you still need a server | Removes home Wi-Fi dependence, not the relay |
  | Amazon Sidewalk | No | n/a | n/a | No | Not available in NZ |
  | Satellite D2D | n/a | n/a | n/a | No | Phone-only; irrelevant to an ESP32 device |
  | DECT NR+ | Potentially | Potentially good | ~km mesh claimed by vendors | Yes | Unresearched for NZ; see Gaps |

- None of the radio options removes the relay for the product's headline promise ("the next town"). They either cover the same street/suburb, or depend on someone else's infrastructure (community repeaters, carriers), which is a different central dependency rather than none.

### Gaps
- QMesh, M17, FLRC voice projects, ESP-NOW long-range mode, DMR/dPMR modules, DECT NR+ (nRF91x1) and LoRaWAN/Helium were not individually sourced in this session; the table rows for them rest on general knowledge and are unverified.
- No numeric LoRa settings or measured latency for LXST voice over RNode were found.
- No pricing found for HaLow modules, SX1262 modules, or antennas; no power-draw figures sourced.
- DECT NR+ regulatory status in NZ (1880-1900 MHz) not found.

## 3. New Zealand regulation

### Takeaway
915-928 MHz is usable licence-free in NZ under the Short Range Devices GURL at up to 1 W EIRP (higher in 920-928 MHz with frequency hopping or digital modulation), with strict out-of-band emission limits (Special Condition 23) that RSM is actively reminding LoRa users about. Amateur bands are ruled out by the ban on obscured-meaning transmissions. Any product containing a radio transmitter needs a Supplier's Declaration of Conformity and R-NZ/RCM labelling. Whether encryption is permitted on SRD or PRS bands was not confirmed from primary text.

### Cited Findings
- The current SRD licence is the Radiocommunications Regulations (General User Radio Licence for Short Range Devices) Notice 2022 — [Gazette 2022-go3100](https://gazette.govt.nz/notice/id/2022-go3100); [RSM SRD GURL page](https://www.rsm.govt.nz/licensing/frequencies-for-anyone/short-range-devices-gurl)
- 2016 amendment: 915-928 MHz (previously 921-928) at up to 0 dBW (1 W) EIRP; 920-928 MHz (previously 921.5-928) at up to 6 dBW (4 W) EIRP (search extract) — [Gazette 2016-go446](https://gazette.govt.nz/notice/id/2016-go446); [Critical Comms summary](https://www.criticalcomms.com.au/content/radio-systems/news/nz-increases-available-bandwidth-for-srds-878225437)
- Special Condition 13 (2022 text): "Transmitters using e.i.r.p. greater than 0 dBW (1 W) must employ frequency hopping or digital modulation techniques" (search extract) — [Gazette 2022-go3100](https://gazette.govt.nz/notice/id/2022-go3100)
- Special Condition 23 (2022 text): unwanted emissions must not exceed -79 dBW (-49 dBm) EIRP within 800-915 MHz and -63 dBW (-33 dBm) EIRP within 928 MHz-1 GHz, 100 kHz reference bandwidth (search extract) — [Gazette 2022-go3100](https://gazette.govt.nz/notice/id/2022-go3100)
- RSM issued a LoRaWAN compliance reminder: devices in 915-928 MHz must comply with the SRD GURL including Special Condition 23; certain 868 MHz-range join transmissions are limited to 3 dBm (-27 dBW) EIRP — [RSM news item](https://www.rsm.govt.nz/about/news-and-updates/lorawan-and-compliance-with-general-user-radio-licence-gurl-conditions); [Critical Comms](https://www.criticalcomms.com.au/content/industry/news/rsm-new-zealand-issues-lorawan-licence-compliance-reminder-19176409)
- 2400-2483.5 MHz listed at 6.0 dBW (4 W EIRP) under Special Condition 13 in the 2015/2016 notices (current 2022 row not confirmed) — [Gazette 2015-go4359](https://gazette.govt.nz/notice/id/2015-go4359)
- PRS/CB: the Citizen Band Radio GURL (licence 228151) covers 26/27 MHz CB and 477 MHz PRS, no fee — [RSM Citizen Band Radio GURL](https://www.rsm.govt.nz/licensing/frequencies-for-anyone/citizen-band-radio-gurl); the 2011 notice includes channels whose use "is limited to telemetry and telecommand" — [2011 CB Radio notice](https://www.rsm.govt.nz/assets/Uploads/pdfs/gazette/43ec454362/radiocommunications-regulations-general-licence-for-cb-radio-notice-2011.pdf)
- A vendor guide says PRS use is licence-free provided the radio is type-approved and carries the R-NZ mark (secondary source) — [Mobile Systems NZ guide](https://mobilesystems.nz/blogs/radio-frequencies/uhf-cb-radio-nz-the-definitive-guide-to-nz-communications)
- Amateur: communications "must not be encoded for the purpose of obscuring their meaning" (except control signals for remote stations); NZART study notes list "secret cipher" messages as forbidden; a new Amateur GURL notice took effect 1 July 2025 — [ZL2GT regulations summary](https://zl2gt.nz/radio-regulations-summary/); [NZART study guide 2025](https://nzart.org.nz/wp-content/uploads/2025/10/NZART-Study-Guide-2025.pdf); [Amateur GURL Notice 2025](https://www.rsm.govt.nz/assets/Uploads/documents/gazette/radiocommunications-regulations-general-user-radio-licence-for-amateur-radio-operators-notice-2025.pdf)
- Product compliance: a supplier of a radio transmitter must "make a declaration of conformity; and label the product with the New Zealand radio label R-NZ"; levels of conformity 1-3 use the RCM, A1-A3 use R-NZ (A1 evidence of compliance, A2 valid test report, A3 accredited test report); "product" includes any assembly of modules forming a finished product; one SDoC covers variants only if changes are cosmetic and do not alter RF emission characteristics — [Radiocommunications (Compliance) Notice 2020](https://gazette.govt.nz/notice/id/2021-go52); [RSM steps to compliance](https://www.rsm.govt.nz/business-individuals/supplier-compliance/steps); [RSM compliance guide](https://www.rsm.govt.nz/assets/Uploads/documents/compliance-guide.pdf)
- RSM guidance also refers to supplier registration and a licence to supply for radio transmitting equipment (search extract) — [RSM step 4](https://www.rsm.govt.nz/business-individuals/supplier-compliance/steps/step-4)

### Inferences
- The SRD GURL extracts seen mention no duty-cycle limit for 915-928 MHz, consistent with NZ following the Australian/US-style power-limited model rather than the EU 1% duty-cycle model. This would make continuous PTT transmissions legal in principle at ≤1 W EIRP. Unverified against the full 2022 table.
- A LoRa or HaLow radio adds a second intentional transmitter to the product. Using a pre-certified module reduces test burden, but the finished product is still "the product" under the Compliance Notice, and FCC/CE module certificates do not automatically demonstrate compliance with NZ's stricter Special Condition 23 band-edge limits (the NZ band starts at 915, not 902 MHz). Expect an NZ/AU-specific test report for the sub-GHz radio; the Wi-Fi-only design avoids this.
- PRS is the only licence-free NZ band built for voice at useful power, but a home-built SA818 device is not a type-approved PRS radio, and an open shared channel cannot meet the product's end-to-end-encryption promise even if scrambling were allowed.

### Gaps
- Could not read the 2022 SRD GURL table: exact current rows for 433.05-434.79 MHz (commonly cited as 25 mW; unverified), 2.4 GHz, and whether any duty-cycle or voice restriction applies.
- No primary source found on whether encryption is permitted or prohibited on SRD GURL or PRS bands. Encrypted Wi-Fi/Bluetooth/LoRaWAN operate under the SRD GURL, which suggests no ban, but that is an inference.
- PRS power limit not confirmed (one community presentation cites 8.3 W max, radios typically 0.25-5 W) — [Onewhero CB presentation](https://www.onewhero.co.nz/wp-content/uploads/2025/10/CBRadioPresentation.pdf); whether data/digital voice is allowed on PRS voice channels not found.
- Applicable NZ radio standard and conformity level for a 915-928 MHz SRD transmitter not identified; likely testing cost not researched.

## 4. Real-world range and the NZ community meshes

### Takeaway
NZ has active community LoRa meshes (Wellington's Meshtastic mesh is the largest found, with >150 nodes and congestion problems; MeshCore networks exist in Tairāwhiti, Bay of Plenty and elsewhere), and a 2026 Massey/CRISiSLab study found city-scale Meshtastic messaging in Wellington achievable with terrain as the main limit. These are volunteer text networks; nothing found supports relying on them for a commercial voice product.

### Cited Findings
- Wellington mesh: LongFast, frequency slot 20, >150 nodes, "becoming unusably congested" (search extract) — [Wellington region thread](https://meshtastic.discourse.group/t/wellington-region-new-zealand/10171)
- "The Lower North Island already enjoys a very populous and active Meshtastic mesh" (event listing) — [Humanitix: Introduction to Meshtastic](https://events.humanitix.com/introduction-to-meshtastic)
- Massey/CRISiSLab paper (repository record dated 2026-05-21): Meshtastic deployed across Wellington's varied terrain; "terrain was the primary limiting factor" but "effective city-scale communications is achievable"; switching to MeshCore "would require the creation of a new repeater network" (search extract) — [Testing of a resilient community-driven communications network of LoRa devices in Wellington](https://rda.sliit.lk/entities/publication/5434cb7c-276a-4876-975e-ef5fa8a58064)
- Gisborne District Council: a Meshtastic proof of concept hit "technical limitations", after which MeshCore was adopted; "we have at least six backup modes of communication and MeshCore is now one of those" — [GDC media release](https://www.gdc.govt.nz/council/news/media-releases/back-up-comms-crucial-in-emergencies)
- MeshCore repeaters deployed on high ground in Bay of Plenty (Dec 2025) — [blowonthepie.co.nz](https://blowonthepie.co.nz/2025/12/07/meshcore-locations-bay-of-plenty); an NZ MeshCore community site exists — [meshed.kiwi](https://meshed.kiwi/)
- NZ community groups: Meshtastic Community New Zealand and an NZ Discord are listed — [awesome-meshtastic](https://github.com/ShakataGaNai/awesome-meshtastic/)
- Public node maps show only nodes that report position via MQTT, and drop nodes not heard for 7 days — [liamcottle/meshtastic-map](https://github.com/liamcottle/meshtastic-map)

### Inferences
- The emphasis in every NZ deployment on hilltop/high-ground repeaters implies indoor-to-indoor house links across a suburb are not dependable without them. A product that needs the buyer's family to site a rooftop or hilltop repeater has replaced "one small server" with "physical radio infrastructure per family".
- Coverage outside the Wellington region and a few MeshCore pockets cannot be assumed; a nationally sold product could not promise it.

### Gaps
- No node counts found for Auckland or Christchurch; no NZ-specific map figures (fetch blocked, maps are interactive).
- No measured indoor house-to-house LoRa range data for NZ suburbs; the Massey paper's distances, delivery rates and hop counts could not be read.
- No statement found from NZ mesh communities on commercial or voice traffic acceptability.

## 5. Hybrid designs and overall verdict

### Takeaway
No licence-free radio found can replace the internet relay for this product as specified: none delivers live, Opus-quality, encrypted voice between towns without elevated infrastructure. Radio is credible only as an optional same-neighbourhood or outage fallback, at the cost of an extra transmitter to certify and markedly worse audio.

### Cited Findings
- Real-time LoRa voice is reported only single-hop and at Codec2 bitrates — [reticulum-meshchat README](https://github.com/markqvist/reticulum-meshchat); [LXST](https://pypi.org/project/lxst/0.4.2/)
- Radio-link latency for LoRa voice is governed by packet duration — [lora_codec2 readme](https://github.com/dudmuck/lora_codec2/blob/master/readme.md)
- A finished product with an added transmitter is assessed as a whole for NZ compliance — [Radiocommunications (Compliance) Notice 2020](https://gazette.govt.nz/notice/id/2021-go52)

### Inferences
- Requirement-by-requirement: live latency (met only by point-to-point radio, HaLow, ESP-NOW, analogue FM); quality for hearing loss (only HaLow, ESP-NOW and analogue FM avoid low-rate vocoders); next-town range (none without repeaters or a carrier); privacy (fails on PRS and amateur); zero central infrastructure (true only for short-range options).
- The relay-burden concern is better addressed in the internet design itself (the relay only forwards end-to-end-encrypted frames, so it can be small, stateless and replaceable) than by changing bearer. That question belongs to the other research strands.
- If a radio fallback is wanted, the lowest-risk sequence is: ESP-NOW first (no new hardware, no new certification) for same-property use; then an optional SX1262 add-on carrying Codec2 voice clips and the offline/low-battery alerts, which are tiny payloads that LoRa handles well.

### Gaps
- Per-unit cost adder and power draw for SX1262/HaLow options not sourced.
- No user evidence on whether elderly listeners accept Codec2-grade audio.
