# ReAWDL v3.6.1

**Intel AX210 — Enhanced Wi-Fi 6, freshness-aware PHY-rate reporting, and experimental native AWDL for macOS Sonoma**

*README updated November 22, 2026 · HE-PR1066 Sonoma Debug build PASS · Standalone Wi-Fi 6 branch build PASS · Release build and independent standalone runtime validation pending*

## Overview

ReAWDL is an experimental modification of AirportItlwm, originally developed to investigate native Apple Wireless Direct Link (AWDL) support on Intel wireless hardware. Extensive work on firmware integration, peer discovery, packet handling, and driver reliability also created opportunities to improve ordinary infrastructure Wi-Fi.

**ReAWDL v3.6.1 emphasizes everyday Wi-Fi 6 performance, reliable connectivity, and trustworthy link-rate reporting.** The HE-PR1063 through HE-PR1065 development sequence added independent Intel firmware-rate diagnostics and a runtime-tested freshness-aware display path. The follow-on HE-PR1066 source revision makes freshness-aware reporting the default, with detailed diagnostic data remaining optional. These changes build on the existing 802.11ax improvements without modifying Intel's firmware rate-control decisions.

Active development of new AirDrop/AWDL functionality is paused for the foreseeable future. Existing experimental AWDL functionality and previous reliability improvements remain in the project.

## What's New in v3.6.1 — November Wi-Fi 6 Update

### Native 802.11ax connectivity and HE rate decoding

- **Wi-Fi 6 on Intel AX210:** Native 802.11ax infrastructure connectivity demonstrated under macOS Sonoma using the modified AirportItlwm 2.4.0 driver.
- **80 MHz and 160 MHz operation:** Tested on 5 GHz networks. At 80 MHz, reported rates reached approximately **1,200 Mbps**; earlier limited 160 MHz testing reached approximately **2,401 Mbps** on a 2×2 connection.
- **Firmware-informed HE rates:** The initial Wi-Fi 6 reporting fix decodes firmware TLC rate notifications using MCS, channel width, spatial-stream count, and HE guard interval instead of assuming a fixed maximum.
- **Meaningful rate changes:** Testing observed valid HE80 combinations including **245 Mbps / MCS 3**, **1,020 Mbps / MCS 11**, and **1,200 Mbps / MCS 11**. The latter two are consistent with different HE guard intervals.

### HE-PR1063 — Independent firmware-rate witnesses

The first rate-probe build added optional, passive, Debug-only diagnostics under `-itlwm-herate`. It separately observes:

1. **TLC-selected initial rate** from Intel firmware notifications.
2. **Successful infrastructure single-frame TX rate** from firmware transmission completions.
3. **Aggregated TX rate** from Block ACK notifications, retained as raw diagnostic data pending validation.

In a live AX210 test, macOS continued to display **245 Mbps / MCS 3** while the successful single-frame completion witness reported a **1,200 Mbps / MCS 11** HE rate. During a subsequent LAN test, the qualifying single-frame completion counter increased by **150** while the TLC notification count stayed unchanged. This demonstrated that the TLC-based display could lag at least some actual successful transmissions; it did **not** establish that all traffic was running at 1,200 Mbps.

### HE-PR1064 — Freshness-aware rate display

The next experimental fix introduced the separate `-itlwm-hefresh` flag, used **together with** `-itlwm-herate` in the tested Debug build.

- Prefers a **recent, validated, successful infrastructure single-frame TX rate** when its sample is less than **three seconds old** and belongs to the current connection/firmware generation.
- **Falls back to the firmware TLC-selected rate** when a completion sample is stale, missing, or invalid; it does not manufacture a maximum PHY rate.
- Adds a corresponding firmware-derived MCS accessor without altering `ni_txmcs`, which is used elsewhere inside the driver.
- Retains the Block ACK rate witness for investigation rather than using unverified aggregated-rate data to drive the macOS display.
- Leaves firmware commands, rate selection, transmit queues, DMA handling, and AWDL behavior unchanged.

The first runtime test showed **1,200 Mbps with MCS 3** in macOS: the new transmit-rate path was active, but one MCS reporting interface was still returning the older TLC-derived index.

### HE-PR1065 — MCS reporting alignment

HE-PR1065 corrects that mismatch by updating the separate `getMCS()` entry points in both the legacy and Skywalk AirportItlwm reporting interfaces, complementing the already-updated `getMCS_VHT()` paths. Under eligible freshness conditions, macOS can obtain the MCS from the same selected firmware sample used for the PHY rate.

**Source checks and Sonoma Debug compilation passed.** After installation, the developer reported maximum PHY-rate reporting and a successful reconnect at that rate after changing the router to another channel **without disabling and re-enabling Wi-Fi**. This is encouraging runtime evidence for the reporting fix, not a proof of permanent peak throughput or broad router compatibility.

### HE-PR1066 — Freshness reporting built in

In v3.6.1, the freshness fix is **default-on in source** for supported Intel AX210 infrastructure connections using 802.11ax, in both Debug and Release configurations. It no longer requires `-itlwm-hefresh` or the diagnostic `-itlwm-herate` boot flag to collect the compact successful-TX evidence used for reporting. The code retains the three-second sample validity window, connection-generation and channel-width checks, and TLC fallback when recent completed-TX evidence is absent. **No firmware TX-rate-control behavior is changed.**

The detailed firmware witness/IORegistry data is still optional in compatible Debug builds via `-itlwm-herate`. A new rollback flag, **`-itlwm-hefreshoff`**, disables freshness-aware reporting and restores TLC-only reporting in HE-PR1066 builds. The earlier `-itlwm-hefresh` flag was required in HE-PR1064/1065 but is unnecessary in HE-PR1066.

**Release status:** The complete HE-PR1066 source passed **21/21 structural checks**, **13/13 policy tests**, and **Sonoma14.4 Debug compilation** on the developer’s Mac. The independent eight-file `wifi6-he-fix` adaptation also passed **23/23 source checks**, **13/13 policy tests**, **12/12 decoder tests**, and a standalone Sonoma14.4 Debug compilation; it was published as commit `38a9bb4`. HE-PR1065 has the clearest documented on-device rate-reporting observations. Independent runtime testing of the standalone patch and a Release-build validation are still pending. The full HE-PR1066 source was published to the project master branch as commit `220e258`, with the generated firmware source excluded from version control.

## Performance and Validation

| Observation | Result / qualification |
|---|---|
| Primary tested adapter | Intel AX210 (2×2) |
| OS / driver | macOS Sonoma; modified AirportItlwm 2.4.0 |
| PHY mode | 802.11ax (Wi-Fi 6) |
| 5 GHz, 80 MHz | Up to ~1,200 Mbps reported; MCS 11 demonstrated |
| 5 GHz, 160 MHz | Up to ~2,401 Mbps reported in earlier limited testing |
| Firmware-rate differential | TLC display 245 Mbps vs. successful single-frame witness 1,200 Mbps |
| Controlled local ping (20 packets, 1-second interval) | 0% loss; 1.666 ms average; 2.168 ms maximum |
| Router channel change after HE-PR1065 | Rejoined with maximum reported rate; no manual Wi-Fi toggle needed in the observed test |
| Qualified aggregate-rate samples | Not yet observed in the HE-PR1063 runtime capture (`BACount = 0`) |

Earlier wireless analysis also observed strong signal quality (approximately **−33 dBm**, with a reported noise floor near **−96 dBm**) and promising Speedtest results relative to the previously used Broadcom BCM94360NG under those particular conditions. These comparisons were not controlled, repeatable head-to-head throughput benchmarks.

**PHY link rate is not real-world download, upload, or file-transfer throughput.** A successful 1,200 Mbps single-frame completion is meaningful firmware evidence, but it is not proof that all frames—or aggregated high-volume traffic—used that rate.

## Reliability and Driver Improvements

The release retains prior work in:

- Transmission queue management, packet ownership, and deferred processing.
- DMA mapping, cleanup, and safer completion handling.
- Firmware command tracking, error handling, and recovery.
- Reset handling, generation checks, and stale-operation safeguards.
- Targeted bounds checks, input validation, and memory-safety improvements.
- Native macOS Wi-Fi interface integration.
- Experimental AWDL synchronization, scheduling, and peer discovery.

The latest HE reporting improvements are intentionally **observational**: they do not force a higher Wi-Fi transmission rate or alter Intel firmware's adaptation policy.

## Demonstrated Functionality

| Capability | Current status |
|---|---|
| Native macOS Wi-Fi interface and infrastructure connectivity | Working on tested AX210 |
| Wi-Fi 6 / 802.11ax | Confirmed |
| 5 GHz / 80 MHz and 160 MHz | Confirmed in limited configurations |
| HE MCS 11 and 80 MHz rates up to ~1,200 Mbps | Observed |
| 160 MHz reported PHY rate up to ~2,401 Mbps | Observed in earlier testing |
| Firmware TLC rate decoding | Implemented and runtime-observed |
| Independent single-frame completion rate witness | Runtime-observed |
| Freshness-aware HE rate/MCS reporting (HE-PR1065) | Runtime-tested with promising results |
| Default-on freshness reporting (HE-PR1066) | Sonoma Debug compilation passed; standalone runtime and Release verification pending |
| Accurate aggregate-based rate reporting | Not yet validated |
| Normal-interval local network latency | Good in tested conditions |
| Native `awdl0` interface and AWDL firmware session scheduling | Demonstrated |
| AWDL management transmission and multicast reception | Demonstrated |
| Apple peer discovery | Demonstrated intermittently |
| Complete native AirDrop file transfer | Not demonstrated |

## Boot Arguments

### New default-on firmware reporting (HE-PR1066)

**No special boot argument is required for freshness-aware Wi-Fi 6 reporting** in the HE-PR1066 source revision. On the supported AX210/802.11ax infrastructure path, the driver automatically prefers a validated successful single-frame completion observed within three seconds; it falls back to the firmware TLC-selected rate if the recent sample is missing, invalid, or too old.

These flags are **optional**, not prerequisites for normal Wi-Fi 6 operation:

- **`-itlwm-herate`** — enables extended firmware-rate witnesses in compatible **Debug** builds, including raw TLC notifications, successful single-frame TX completion evidence, and raw Block ACK rate observations. Intended for troubleshooting; not required for the default reporting fix.
- **`-itlwm-hefreshoff`** — explicitly disables freshness-aware reporting in **HE-PR1066 builds** and uses the older TLC-only display path. Useful for controlled comparisons and rollback.
- **`-itlwm-diag`** — separate, opt-in broader driver/AWDL diagnostics. Not required for Wi-Fi 6.
- **`-awdlexp`** — experimental AWDL firmware integration where supported. Independent of the Wi-Fi 6 rate-reporting fix.

**Earlier HE-PR1064/1065 Debug binaries still require both `-itlwm-herate` and `-itlwm-hefresh` to activate the experimental freshness path.** The default-on behavior is a source change for HE-PR1066 and does not retroactively change those installed binaries. `-itlwm-hefresh` is no longer required in HE-PR1066. Only add flags that the specific kext version supports; retain existing unrelated OpenCore arguments.

### Checking optional firmware witnesses

When `-itlwm-herate` is enabled with a compatible Debug kext:

```bash
kmutil showloaded 2>/dev/null | grep -i AirportItlwm
ioreg -l -w0 | grep -E '"HEPR106(3|4)' | sort
system_profiler SPAirPortDataType | \
  grep -E 'PHY Mode:|Channel:|Transmit Rate:|MCS Index:' | head -4
```

The `HEPR1063TlcRaw`, `HEPR1063SingleRaw`, and `HEPR1063BARaw` fields are raw firmware witnesses. `HEPR1064LastSource` is `0` for TLC fallback and `1` for a recent qualified single-frame sample. IORegistry publication is periodic, so those properties may lag individual reporting calls. Raw rates are not automatically sustained-throughput measurements.

## Known Limitations

- **Experimental scope:** HE-PR1065 on-device results and HE-PR1066 Debug compilation are preliminary and focused on the AX210 under Sonoma; compatibility with other adapters, routers, and macOS versions remains unverified.
- **Freshness is conditional even when enabled by default:** A successful single-frame completion is only used for a short validity window. During idle periods, the display may fall back to the older TLC rate, which itself can lag firmware adaptation.
- **Persistent maximum is not a guarantee:** A 1,200 Mbps display across channel changes is encouraging but does not prove continuous live-rate accuracy, sustained throughput, or correct downward adjustments; all require further tests.
- **Rate/MCS queries are separate:** macOS may request them at slightly different times. A sample expiring between requests can still cause a momentary mismatch, even after HE-PR1065.
- **Aggregated TX:** The Block ACK structure exposes a `tx_rate` field, but qualifying live samples and exact encoding have not yet been validated. High-throughput, aggregate-heavy behavior still needs measurement.
- **Actual throughput:** No comprehensive sustained LAN throughput benchmark, cross-router compatibility suite, or prolonged sleep/wake validation is claimed for the new freshness fix.
- **160 MHz:** Functional in limited tests, but router- and environment-dependent; 80 MHz remains a practical configuration.
- **Wi-Fi 6E / 6 GHz:** No claim of validated 6 GHz operation is made by this release.
- **AWDL / AirDrop:** Complete AirDrop transfers remain unfinished. Peer discovery and advertising can be intermittent; recent testing identified dependence on an active 5 GHz infrastructure connection for advertising in the current experimental implementation. Earlier source snapshots exist for separate investigation. Station-3 work should not be mistaken for finished AWDL unicast support.
- **Sleep/wake:** Reliable post-wake AWDL advertising and repeated long-term Wi-Fi reliability still need further validation.

The project is best understood as an enhanced experimental Intel Wi-Fi driver with optional AWDL research functionality—not a finished AirDrop implementation.

## Included Utility — `intel_wifi_hardware_firmware.py`

The read-only macOS Python utility reports the Intel adapter model, loaded firmware, negotiated wireless standard, operating channel and bandwidth, and available PHY link-rate information. It is intended to describe actual observed operating capabilities rather than theoretical marketing specifications. It avoids displaying personal directory paths, Wi-Fi network names, and MAC addresses, and can include additional ReAWDL diagnostics when available.

## Future Development

Future updates may address Wi-Fi reliability, firmware diagnostics, compatibility, security hardening, and support for additional hardware. The immediate technical opportunities are to validate aggregate-rate telemetry, improve rate/MCS sampling consistency across idle periods and reconnects, and measure actual sustained throughput without sacrificing stable ordinary Wi-Fi behavior.

Active development of new AWDL/AirDrop features remains paused. Existing experiments may be maintained, but further releases are not contingent on completing AirDrop.

## Credits & Acknowledgments

**BringBackPhysicalMedia (GitHub)** — Project direction, extensive testing, experimental design, troubleshooting, performance evaluation, and validation of driver changes.

**KGP (GitHub)** — Intel driver engineering references that informed improvements involving DMA buffer mapping, packet ownership and cleanup, deferred queue recovery, firmware command lifetimes, reset safeguards, and Apple-compatible peer-to-peer interface lifecycle management.

**OpenIntelWireless** — The original AirportItlwm project and its contributors provided the foundational Intel Wi-Fi driver on which ReAWDL was developed.

**Linux AirDrop Project — `airdrop-mt7921`** — Protocol research that informed investigation of AWDL MAC identity synchronization, peer-to-peer operation, radio channel alignment, synchronization timing, and the distinction between peer discovery and successful data transfer. Its implementation targets different hardware.

**Artificial Intelligence — AI-assisted development** — AI coding assistants contributed substantial experimental project-specific code, tooling, source analysis, debugging support, firmware research, and documentation. Human-directed testing, observations, and engineering decisions guided the work.

## Closing Notes

ReAWDL began as an effort to bring native AWDL functionality to Intel wireless hardware. Although complete AirDrop functionality remains unfinished, the project has produced useful driver reliability improvements, experimental native-interface work, and a new set of firmware-based Wi-Fi 6 reporting capabilities.

The November HE-PR1063–1066 work illustrates the project's current direction: investigate with independent firmware evidence, make narrowly scoped changes, preserve rollback checkpoints, and prioritize the everyday wireless experience.

**Dedicated to my Mom.**
