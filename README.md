---
title: "Hypervolt Home 3 Pro: Local Access and Hardware Research"
subtitle: "Technical findings and proposed next steps | Community handover report"
date: "20 November 2026"
author: "Independent owner-led investigation (not affiliated with Hypervolt)"
geometry: margin=23mm
fontsize: 10pt
lang: en-GB
colorlinks: true
---

# Executive summary

**Objective:** obtain charging status, instantaneous power, current, voltage, energy and historical data locally, without depending on Hypervolt's cloud service or mobile application, while preserving all built-in electrical protections.

An installed **Hypervolt Home 3 Pro** was investigated using network observation, photographic hardware inspection and byte-for-byte analysis of its removable microSD card before and after commissioning. The principal findings are:

1. The charger uses **MQTT over TLS on TCP port 8883**, connecting to `broker.prod.hypervolt.co.uk`. Three IPv4 addresses observed/resolved during the investigation were `34.246.234.36`, `54.72.194.209` and `3.248.158.201`. They are observations, not a permanent allow-list.
2. A captured successful connection negotiated **TLS 1.2**, cipher suite `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` (`0xC02F`). The server presented a public Amazon-issued certificate for the broker. Replacement certificates issued by an unauthorised local CA were rejected by the charger. This establishes certificate validation, **not** proof of certificate pinning.
3. The communications daughterboard is marked **HVSANMARINO Rev E**, with an **Espressif ESP32-WROOM-32D**, Ethernet connector, microSD socket, a separate **Winbond W25Q128JV-family 16 MiB SPI NOR flash** (`U6`) and two unpopulated pad groups potentially useful for diagnostics. Their electrical pinout has **not** been established.
4. A **Sahasra 32 GB microSDHC** card contains a small FAT32 partition and a larger proprietary raw partition identified by the ASCII signature **`TM DATA`**. Factory-state storage was almost entirely blank. Commissioning populated approximately 81 kB of structured local records and created a 56-byte FAT32 file named **`ACCUM.DAT`**.
5. **2,834 structured records** were recovered from five 16 KiB storage pages. Every recovered record passed its **CRC-16/XMODEM** check. They carry measurement IDs, six-byte source identifiers, timestamps, and typed data, including plausible mains-voltage values and settings such as `wired` and the UK time zone/NTP configuration.
6. No readable cloud broker credentials, PEM certificate, private key or firmware image was identified in either SD-card image. This is **not** proof that none exist in binary or encoded form. The ESP32 module's firmware flash and the additional Winbond chip are the most useful unexamined storage targets.

**Status:** local telemetry storage has been substantially decoded; a supported local operational interface has **not** been identified. This report emphasises verified findings and actionable next steps rather than unsuccessful-tool history.

## Evidence classification

- **Observed / verified** — appears directly in the supplied images, packet captures, photographs or checksum-validated binary records.
- **Interpretation** — credible explanation of observed data but not yet experimentally established (for example, a measurement's units).
- **Unknown** — not yet evidenced, including firmware encryption settings, MQTT authentication method, pinout, local API availability and exact cloud telemetry contents.

# 1. Device and research boundaries

| Item | Finding |
|:--|:--|
| Product | Hypervolt Home 3 Pro EV charger |
| Main PCB | `HYPERVOLT IMOLA RevF` |
| Communications / processor PCB | `HVSANMARINO Rev E` (board marking) |
| Main communications MCU/module | Espressif `ESP32-WROOM-32D` |
| Wired networking | RJ45 Ethernet port present on daughterboard; device observed communicating by Ethernet |
| Removable storage | microSDHC, 32 GB, Sahasra |
| Separate non-volatile memory | Winbond `25Q128JV` family (`U6`), nominal 128 Mbit / 16 MiB |

All analysis described here concerns the researcher's own equipment and locally obtained copies. An isolated network segment prevented the charger from initiating connections to ordinary household devices. No conclusions are made about electrical safety certification, relay operation or firmware safety mechanisms. **Do not bypass load management, residual-current/PEN-fault protection, thermal controls or charging interlocks.** Physical board testing should be confined to a spare, fully isolated board and verified low-voltage supplies.

# 2. Cloud endpoint and network protocol

## 2.1 Identified destination

```
DNS hostname / TLS SNI: broker.prod.hypervolt.co.uk
Transport:              TCP
Service port:           8883
Application protocol:   MQTT, protected by TLS
Observed/resolved A:    34.246.234.36
                        54.72.194.209
                        3.248.158.201
```

The IP addresses occurred in captures or were supplied from resolution during the investigation; **use the hostname for identification** because cloud endpoints may change. The captured TLS ClientHello included the hostname as **Server Name Indication (SNI)**. Substantial encrypted application-data traffic appeared in the capture, demonstrating established encrypted sessions, although the underlying MQTT topics, payloads, credentials and commands were not decrypted.

The investigation used a separate Ethernet network behind a policy-controlled firewall. Packet captures were obtained at the boundary, with timestamped observations of DNS, TCP connection attempts and TLS exchanges. This layout is useful for future work because it permits passive inspection without allowing the charger to reach the home's other systems. The exact private IP addressing is deliberately omitted from this shareable report.

## 2.2 Verified TLS profile

A successful broker connection in the supplied `capture.zip` packet capture shows:

| Handshake property | Observed value |
|:--|:--|
| Negotiated version | **TLS 1.2** (`0x0303` in ServerHello) |
| Cipher suite | **`0xC02F` — TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256** |
| TLS SNI / certificate identity | `broker.prod.hypervolt.co.uk` |
| Leaf certificate issuer | **Amazon RSA 2048 M01** |
| Leaf subject / SAN | `broker.prod.hypervolt.co.uk` |
| Intermediate | Amazon RSA 2048 M01, issued by Amazon Root CA 1 |
| Additional presented chain certificate | Amazon Root CA 1 cross-signed by Starfield Services Root Certificate Authority - G2 |
| Leaf expiry in observed chain | **26 February 2027, 23:59:59 UTC** |

For reproducibility, the observed leaf certificate's **SHA-256 fingerprint** was:

```
d5:2a:d0:0a:7e:ba:f8:ef:37:6f:af:aa:8f:81:46:96:
48:ad:50:c7:e9:6b:7f:3a:e1:f1:89:8a:d6:e0:f4:37
```

This is a **snapshot fingerprint, not a suggested pinning value**; public server certificates rotate. One inspected successful TLS server flight contained `ServerHello`, `Certificate`, `ServerKeyExchange` and `ServerHelloDone`, with **no `CertificateRequest` message**. Thus mutual TLS client-certificate authentication was not evident in that particular exchange. MQTT-layer passwords, signed tokens or other authentication remain possible and unverified.

## 2.3 Behaviour when the presented certificate is untrusted

A controlled certificate-validation test transparently directed the charger's connection for the broker to an alternative TLS endpoint, which presented a substitute certificate using the expected broker hostname but signed by a **locally generated, untrusted certificate authority**. The charger terminated the TLS exchange, including an observed fatal **`access_denied`** alert, before any usable MQTT session was established.

**What that proves:** the charger does not simply accept arbitrary substitute certificates under these conditions. **What it does not prove:** the firmware uses certificate pinning. Ordinary trust-store validation, an embedded CA bundle, hostname/certificate policy, or other TLS checks could explain the result. The observed rejection is independent of whether the ESP32's firmware flash is encrypted. Some diagnostic libraries refer to a `tlsv1 alert` even when the actual negotiated/record protocol is TLS 1.2; that wording alone must not be interpreted as TLS 1.0.

# 3. Processor board and memory components

## 3.1 Electronics seen in photographs

- **`HYPERVOLT IMOLA RevF`** — larger main PCB carrying charger-related electronics.
- **`HVSANMARINO Rev E`** — upright communications daughterboard, containing the Wi-Fi-capable ESP32 module, Ethernet interface and microSD slot.
- **`ESP32-WROOM-32D`** — Espressif 32-bit dual-core-capable MCU module. This module family normally includes **4 MB of SPI flash inside the module package**; that flash is *off-chip relative to the ESP32 silicon*, not internal silicon flash. Verify the actual installed capacity before assuming an image size. [Espressif datasheet](https://documentation.espressif.com/esp32-wroom-32d_esp32-wroom-32u_datasheet_en.html).
- **`U6`: Winbond `25Q128JV` family** — identifiable eight-pin SPI NOR flash, **128 Mbit = 16 MiB**. It is separate from the SPI flash packaged within the ESP32 module. Its connections and content have not been examined, so **it is not established to be the ESP32 boot flash or to hold application firmware**. [Winbond datasheet](https://www.winbond-electronics.com/pdf-7d/w25q128jvsjq.pdf).
- **Unpopulated `J5`** — approximately 2 × 3 exposed pads, with pin-number markings. **Unpopulated `J2`** — a second group of five exposed pads beside the ESP32. Neither has a confirmed pinout. They may be manufacturing test, UART, SPI or other interfaces; pad shape alone is inconclusive.

The ESP32 ROM serial downloader normally depends on UART TX/RX, GPIO0 boot strapping and reset/enable signals, but a working connector has **not** been located on this PCB. A read of the eFuse security state has **not** been performed. Encryption, secure boot and download-mode availability remain unknown. They are separate from the certificate-validation behaviour described earlier. [Espressif flash encryption documentation](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/security/flash-encryption.html).

# 4. Memory card identity, size and partitioning

## 4.1 Physical card

| Property | Observed detail |
|:--|:--|
| Brand | **Sahasra** |
| Form factor / capacity | **microSDHC, labelled 32 GB** |
| Markings | **A1**, **U1**, **V10** and Class 10-style speed symbols |
| Country | **Made in India** |
| Rear marking | `601AAC3Y118` (printed marking, **not confirmed as a model/SKU**) |
| Full image length | **31,266,439,168 bytes** (about **29.12 GiB**) |

Two independent, equal-sized, bit-for-bit image files were created and retained: `hypervolt-original.img` before commissioning, and `hypervolt-firstboot.img` afterwards. Neither was modified for analysis.

## 4.2 Active MBR layout

The active partition table is **DOS/MBR**, using **512-byte sectors**.

| Region | MBR type | Start LBA | Sector count | Size | Absolute start |
|:--|:--|--:|--:|--:|:--|
| Partition 1 | `0x0C` FAT32 LBA | 63 | 1,044,162 | **534,610,944 B** (~509.8 MiB) | `0x7E00` |
| Partition 2 | `0x0D` proprietary/raw | 1,044,225 | 33,543,720 | **17,174,384,640 B** (~15.995 GiB) | `0x1FDE0200` |
| Unallocated tail | none | 34,587,945 onward | — | **13,557,411,328 B** (~12.63 GiB) | `0x41F8A5200` |

Partition 1 ends immediately before partition 2 begins. The card's nominal 32 GB capacity is **not** available as a single mounted volume. Most of its space is unoccupied or not populated in these captures.

## 4.3 Additional, inactive FAT32 formatting traces

The image also contains an apparently valid FAT32 boot sector at absolute offset **`0x400000` (4 MiB)**, with a backup boot sector at `0x400C00` and FAT structures near `0x4B9000` and `0x85C800`. Its declared geometry would extend to the end of the card and **overlap the active partitions**. It is therefore **not an additional active third partition**; residual manufacturer/card-formatting metadata is a plausible explanation. It was unchanged across commissioning. No concealed firmware was found there.

# 5. Card before commissioning

The *factory-state* sparse listing contains **633 retained bytes across 20 spans**, when uninterrupted runs of **256 or more `00` or `FF` bytes** are omitted. This is a *sparse-listing statistic*, **not** a statement that the entire image contains only 633 non-zero bytes. The 29.12 GiB image is nonetheless overwhelmingly blank/filler.

Recognisable content consists mainly of:

- Master boot record and partition entries;
- FAT32 boot/backup boot sectors, FSInfo and reserved FAT entries;
- Inactive/older FAT32 formatting traces described above;
- Proprietary partition header at **`0x1FDE0200`**:

```
54 4D 20 44 41 54 41 00 01 00 00 00 02 00 00 00
20 D6 FF 01 20 00 00 00
"TM DATA\0" ...
```

At offset `0x10` in that header, little-endian `0x01FFD620` equals **33,543,712 sectors**, precisely **eight fewer than the proprietary partition's 33,543,720 sectors**. Its purpose is unknown; it may describe usable capacity or reserved end sectors. Much of the rest of this partition was zero-filled, with `FF`-filled regions near the header. There were no pre-commissioning event records in the observed region and **no recognisable firmware image, cloud credentials, MQTT hostname or PEM certificate in the sparse listing**.

# 6. Changes after commissioning

## 6.1 Byte-by-byte comparison

A full comparison of the two **31,266,439,168-byte** images produced:

| Metric | Result |
|:--|--:|
| Individual changed bytes | **56,571** |
| Changed 512-byte sectors | **167** |
| Changed bytes in FAT32 | **52**, across 6 sectors |
| Changed bytes in proprietary `TM DATA` | **56,519**, across 161 sectors |
| Post-commissioning sparse output | **81,405 retained bytes across 23 spans** |

The first modified byte is the FAT32 volume metadata at absolute offset **`0x7E41`**. The partition tables and `TM DATA` signature remained unchanged.

## 6.2 A new FAT32 file: `ACCUM.DAT`

A short-directory entry appeared at **`0x8B600`**, naming **`ACCUM.DAT`**. It points to **FAT cluster 4** and declares a **56-byte** file. The data is stored around absolute offset **`0x8D600`**. FAT metadata indicates creation/modification on **19 November 2026 at approximately 15:24:20** (FAT timestamps have no timezone and generally two-second resolution).

The 56-byte data includes two little-endian integers:

- **496,110** (at file offset `0x10`), matching the observed value for telemetry ID `0x0070`;
- **890,851** (at file offset `0x18`), matching the observed value for telemetry ID `0x006F`.

Their interpretation as cumulative/accumulator values is **probable but not yet proven**. They are not established as watt-hours, charge counts or any other specific units. The final validated telemetry timestamp was **15:24:20.844 UTC**, closely matching the file's FAT timestamp.

## 6.3 `TM DATA`: page structure and record extraction

The newly populated proprietary partition contains **five sequential, fixed-spaced 16 KiB pages**, each beginning with a **64-byte metadata region** followed by variable-length records. The contiguous populated sparse span runs from image address **`0x1FDE0600` to `0x1FDF4144`** (end exclusive), a span of **80,708 bytes**.

| Page | Absolute page start | CRC-validated records |
|:--|:--|--:|
| 1 | `0x1FDE0600` | 576 |
| 2 | `0x1FDE4600` | 577 |
| 3 | `0x1FDE8600` | 575 |
| 4 | `0x1FDEC600` | 574 |
| 5 | `0x1FDF0600` | 532 |
| **Total** | | **2,834** |

Each record follows this empirically verified framing:

| Byte offset within record | Size | Meaning |
|:--|--:|:--|
| 0 | 4 | Constant record prefix **`03 03 01 00`** |
| 4 | 6 | Source / device identifier (two values observed) |
| 10 | 2 | Measurement or event ID, **little-endian** |
| 12 | 8 | Timestamp, **little-endian unsigned milliseconds** |
| 20 | 2 | Payload length, **little-endian** |
| 22 | Variable | Payload (1, 2, 4, 8 or larger number of bytes) |
| Last 2 | 2 | **CRC-16/XMODEM**, stored little-endian |

**Checksum rule:** initialise CRC to `0x0000`, polynomial `0x1021`, non-reflected; calculate over bytes **10 to the end of the payload** (that is, the event ID, timestamp, payload length and payload), excluding the four-byte prefix, six-byte identifier and the stored CRC field. **All 2,834 records passed**. Record lengths and boundaries are therefore much better established than the physical meaning of each measurement ID.

Two source identifiers appeared, distinguished by their final byte **`...:00`** (**2,172 records**) and **`...:01`** (**662 records**). The full six-byte values are deliberately not published in this handover in case they are device-specific. They *resemble* adjacent MAC addresses but have **not** been independently checked against the actual Ethernet MAC or controller identity. There were **261 distinct measurement/event IDs** in the decoded sample.

## 6.4 Clock behaviour and apparent restarts

Initial records in each of the first three pages have millisecond timestamps near zero (`227`, `192`, `201`) before real Unix-epoch millisecond timestamps appear. Three startup/initialisation sequences can be distinguished:

| Apparent initialisation | Approximate real time (UTC) | Interpretation |
|:--|:--|:--|
| Sequence 1 | **14:44:54**, 19 Nov 2026 | Records initially use a short relative timebase |
| Sequence 2 | **14:47:01**, 19 Nov 2026 | Relative clock starts afresh |
| Sequence 3 | **15:17:10**, 19 Nov 2026 | Relative clock starts afresh, with later continuous activity |

Records then continue through **15:24:20.844 UTC**. These are consistent with software restarts or new recording sessions; they do **not** independently prove power cycling, crash recovery or watchdog resets. A likely explanation is that the device logs before synchronising its wall clock and later switches to Unix time.

## 6.5 Recognisable configuration and measurements

| Event ID | Observed data | Assessment |
|:--|:--|:--|
| `0x0035` | `wired\0`, repeated three times | Ethernet/network-mode configuration |
| `0x8036` | `GMT0BST,M3.5.0/01,M10.5.0/02` and partial `pool.ntp.or...`, in several forms | UK time zone and NTP settings; some strings fragmented/overwritten |
| `0x006C` | Four-byte floats such as 241.8, 242.1, 243.6 | **Likely AC voltage**, unconfirmed |
| `0x00D0` | Floats around 242–243 on some occasions, with different start-up values | Possibly another voltage reading; unconfirmed |
| `0x0043` | Floats around 5.7–5.9 | Unknown |
| `0x006F`, `0x0070` | Eight-byte integer values, shared with `ACCUM.DAT` | Likely counters/accumulators, units unknown |
| `0x8003`, `0x8145`, `0x814D` | Long zero-filled payloads during startup | Unknown structures/configuration placeholders |

Many four-byte payloads plausibly represent IEEE-754 little-endian floats, while others are integers or short status values. **Do not infer engineering units merely from a plausible number.** A controlled charging session cross-checked against an independent meter or the manufacturer's UI would be needed to map IDs reliably.

# 7. Cloud credentials, privacy and physical access

Plaintext string searches over the **entire original and commissioned SD images** yielded no recognisable broker hostname, MQTT keyword, HTTPS endpoint, PEM private key or PEM certificate. Only UK time-zone/NTP strings appeared among the selected network-related patterns after commissioning. The short string `wired` was absent from that particular search because its minimum string length was six; it is, however, present in the decoded binary records.

This narrows the likely location of cloud configuration to firmware or another non-volatile store, but it **does not rule out** binary DER certificates, compressed/encoded data or encrypted material on the card. The ESP32 module flash and the board-level Winbond U6 chip have not been dumped or independently analysed.

The microSD is removable after physical access to the charger's enclosure. The proprietary record format is **not equivalent to encryption**: the stored records were recovered and CRC-validated without access to manufacturer-held keys. They contain times, device/source identifiers and electrical-related measurements. Those facts establish a **physical-access privacy consideration**; they do **not** demonstrate a remotely exploitable vulnerability, compromised cloud service or the presence of passwords on the card. For public publication, remove charger serial numbers, full unique identifiers, account details and any discovered credentials from images or files.

It is also important to distinguish **what is locally recorded** from **what is sent to the cloud**. The capture confirms encrypted MQTT traffic to the broker, but because application payloads have not been decrypted, it does not establish that every recovered SD record is transmitted or identify the exact content/retention policy of cloud telemetry.

# 8. Why local access matters

The owner's goal is not to defeat electrical safety systems or obtain another user's data. It is to **access measurements generated at their own property, integrate them with a local energy dashboard and avoid reliance on a commercial cloud for routine monitoring**.

There are legitimate reasons to challenge cloud-only access to monitoring and remote-control functions in purchased appliances: internet and service outages; provider closure or commercial-policy changes; future subscription requirements; loss of historical data; app obsolescence; unavailability of account provisioning; and uncertainty about the amount of usage telemetry retained remotely. A further concern is that a physically removable SD card stores readable, timestamped operating data.

**A critical factual qualification:** Hypervolt's published support guidance states that the charger can continue using previously retained settings while offline; app changes will not apply, and after a power cycle it reverts to default settings. Consequently, **this investigation does not establish that the charger cannot physically charge a car without internet access**. What is documented here is cloud-mediated access to the manufacturer's app functions and an absence, *so far*, of an identified supported local telemetry API. [Hypervolt offline guidance](https://support.hypervolt.co.uk/en/knowledge-base/troubleshooting-network-loss-common-causes-and-solutions).

In the UK, the consumer-connectable-product security regime has applied since **29 April 2024**, addressing baseline product security, but it does **not generally guarantee a local API or indefinite cloud-service availability**. Privacy and transparency obligations may arise under UK data protection law where usage telemetry relates to an identifiable household; the exact legal position depends on the processing, notices and lawful bases. Neither a privacy-law breach nor an absence of any lawful opt-out has been established here. [UK government PSTI guidance](https://www.gov.uk/guidance/regulations-consumer-connectable-product-security); [ICO guidance on consumer IoT information](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/online-tracking/guidance-for-consumer-internet-of-things-products-and-services/what-information-do-iot-products-use/).

A useful product-design principle would be **local read-only telemetry and offline core operation by default, with cloud-connected convenience features as an optional enhancement**. This is a project motivation and recommendation, not a claim about a currently mandated product requirement.

# 9. Recommended next steps (priority order)

1. **Publish this sanitised baseline and archive evidence.** Retain unchanged SD images, hashes, a minimal raw PCAP, decoded CSV and PCB photographs privately. Publish the decoder and record layout after removing unique device identifiers; do not publish authentication secrets, if discovered.
2. **Document offline functionality without opening the charger.** With all electrical protections in place, test normal permitted use during a planned, temporary internet disconnection: charging initiation/continuation, previously set schedules, current limits, status LEDs, local network discovery and behaviour after a normal reboot. Compare results against Hypervolt's own offline guidance. Do not experiment with safety-critical interlocks.
3. **Look for a supported local surface.** On the isolated segment, passively inventory DHCP, DNS, ARP, mDNS/SSDP and service announcements. The manufacturer's network setup instructions reference a device-local provisioning page at `hypervolt.energy` while connected to its setup hotspot; establish whether that is *only* provisioning or exposes a documented safe read-only status path. Its existence is **not proof** of a local operational API.
4. **Acquire a spare/donor communications PCB** marked **HVSANMARINO Rev E**, with the ESP32, flash and interfaces populated, rather than risking the commissioned charger. Match revision and silicon before buying; an uncommissioned or faulty board may be sufficient for offline study.
5. **Map the donor board electrically with all power removed.** Trace `J2`/`J5`, ground, supply rails, boot/reset/UART and U6 SPI pins. Verify 3.3 V compatibility and the destination of every test pad before connecting any programmer. Do not power a PCB through unverified signal lines.
6. **Obtain read-only flash evidence.** Once the electrical interface and supply are understood, record device/chip identifiers and security eFuse state; if permitted by the configuration, take separate binary backups of the ESP32 module flash and U6, with repeat-read SHA-256 verification. Never erase, overwrite, unlock, mass-erase or reconfigure security fuses as part of initial research. Flash encryption may render a raw dump unreadable.
7. **Analyse recovered firmware only if available in plaintext.** Identify storage partitions, build version, TLS CA trust material, MQTT broker references, authentication mechanism, message topics, update logic and possible local endpoints. Redact private keys and tokens; never use credentials from a donor device to impersonate it on the live service.
8. **Map event IDs by correlating independent measurements.** Re-examine the existing SD decoder against deliberately controlled charging sessions, known voltage/current/power conditions and an independent local meter. A local meter is also a practical, cloud-free route to power and energy monitoring even if firmware study does not yield an interface.
9. **Ask Hypervolt for a documented route.** Request details of local telemetry availability, an export endpoint, offline guarantees, data stored on SD/cloud, retention/deletion controls, the status of encrypted at-rest storage and what happens if the cloud service is withdrawn.

**End state sought:** a maintainable, read-only **LAN telemetry source** for integration into independent monitoring systems, with any optional control limited to fully understood, supported functions and **no interference with the certified charging/safety mechanisms**.

# 10. Outstanding questions

- Are the `J2` / `J5` pads connected to UART, SPI, JTAG or manufacturing test signals?
- What is U6 connected to, and is its 16 MiB content firmware, update staging, a second controller's storage or general data?
- Is ESP32 flash encryption or secure boot enabled? Is ROM download mode available read-only?
- Where are the device's cloud trust anchors and authentication material stored?
- Is broker authentication performed in MQTT payloads (such as CONNECT credentials), via client certificates or another layer? The examined TLS flight did not request a client certificate.
- Is a local read-only API, serial reporting stream or broadcast telemetry service implemented but undocumented?
- What are the physical meanings and units of the 261 distinct event IDs?
- What is the purpose of the two adjacent six-byte source identifiers? Do they match actual Ethernet/Wi-Fi MAC addresses?
- Are the SD telemetry records simply a local history, or are any/all uploaded to cloud services? What is the retention policy?
- Is the SD data protected at rest in newer firmware revisions, and what is the threat model for physical service access?
- What functions survive prolonged loss of internet, reboot while offline, app discontinuation or permanent backend closure?

# 11. Evidence and reproducibility

**Primary evidence used** (private working copies, not included in this public report):

- `hypervolt-original.img`, `hypervolt-firstboot.img` — full 31,266,439,168-byte SD images held by the owner.
- `hypervolt-original-sparse.hex.txt` and `hypervolt-original-sparse-index.csv` — pre-commissioning sparse retained-byte listing and exact original offsets.
- `hypervolt-firstboot-sparse.hex.txt` and `hypervolt-firstboot-sparse-index.csv` — post-commissioning counterpart.
- `all-byte-changes.csv` — exact per-byte before/after comparison, with location within the active partitions.
- `hypervolt_decoded_tmdata.csv` and `hypervolt_tmdata_decoder.py` — 2,834 CRC-validated records and the read-only decoding logic.
- `capture.zip` / included `hypervolt-firstboot.pcap`, and `hypervolt-secondboot.pcap` — captured network traffic including TLS handshakes. The successful TLS 1.2 server certificate chain was extracted from the former. Packet captures should be checked for identifiers before public release.
- Photographs of PCB markings, ESP32 daughterboard, SPI NOR chip and Sahasra SD card.

**External technical and product references** (consulted 20 November 2026):

1. [Hypervolt — Home 3 Pro Installation Guide](https://support.hypervolt.co.uk/en/knowledge-base/home-3-pro-installation-guide) — hardware installation, Ethernet and normal functional testing.
2. [Hypervolt — My Hypervolt is Offline!](https://support.hypervolt.co.uk/en/knowledge-base/troubleshooting-network-loss-common-causes-and-solutions) — retained offline settings, app limitations and reboot behaviour.
3. [Espressif — ESP32-WROOM-32D datasheet](https://documentation.espressif.com/esp32-wroom-32d_esp32-wroom-32u_datasheet_en.html) — MCU, interfaces, flash and download modes.
4. [Espressif — Flash Encryption (ESP32)](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/security/flash-encryption.html) — eFuse-controlled encryption and download-mode restrictions.
5. [Winbond — W25Q128JV datasheet](https://www.winbond-electronics.com/pdf-7d/w25q128jvsjq.pdf) — 128 Mbit/16 MiB SPI NOR flash.
6. [UK government — Consumer connectable product security regulations](https://www.gov.uk/guidance/regulations-consumer-connectable-product-security) — PSTI baseline requirements.
7. [ICO — What information do IoT products use?](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/online-tracking/guidance-for-consumer-internet-of-things-products-and-services/what-information-do-iot-products-use/) — personal data and telemetry in consumer IoT.

---

*Report date: 20 November 2026. Independent research based on one observed Home 3 Pro and one before/after commissioning dataset. Findings may differ across hardware and firmware revisions. Firmware or SD-card contents were not modified during the forensic analysis. No endorsement or affiliation with Hypervolt is implied.*
