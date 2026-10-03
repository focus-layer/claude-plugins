---
name: troubleshoot-cable-modem
description: >
  Read a cable modem's DOCSIS telemetry to decide whether a complaint originates on
  the cable side of the network, which carrier is at fault, whether the pattern is a
  plant fault or a collection artifact, what the customer actually felt, and what to
  tell the ISP. Use when the network's WAN is cable (DOCSIS), when an issue names a
  `docsis_*` metric (`docsis_uncorrectable_codeword_ratio`,
  `docsis_uncorrectables_rate`, `docsis_ds_snr_db`, `docsis_ds_power_dbmv`,
  `docsis_us_power_dbmv`, `docsis_channel_locked`), when the user mentions a modem,
  "signal levels", "SNR", "uncorrectables", "T3/T4 timeouts", "partial service", or
  asks "is it my ISP / the cable / the line", or when `triage-network-complaint`
  lands on the WAN bracket for a cable site. Read-only: it observes and reports, it
  never changes modem or ISP configuration.
argument-hint: "[network-id-or-name] [optional: modem name/ip] [optional: time window]"
allowed-tools: >
  Bash, Read, Skill,
  mcp__sprinter__ask_user,
  mcp__sprinter__get_reference_doc,
  mcp__sprinter__find_network,
  mcp__sprinter__list_networks,
  mcp__sprinter__show_network,
  mcp__sprinter__network_tech_stack,
  mcp__sprinter__find_device,
  mcp__sprinter__show_device,
  mcp__sprinter__show_probes,
  mcp__sprinter__network_issues,
  mcp__sprinter__network_device_events,
  mcp__sprinter__event_evidence,
  mcp__sprinter__issue_chart,
  mcp__sprinter__timeseries_instant,
  mcp__sprinter__timeseries_range,
  mcp__sprinter__timeseries_analyze,
  mcp__sprinter__traceroute_history,
  mcp__sprinter__isp_info,
  mcp__sprinter__ioda
---

# Troubleshooting the cable (DOCSIS) side

## Why this skill exists

The raw DOCSIS series actively mislead without the reading rules. On one fleet modem
the UI, the Grafana dashboard and the anomaly detector all agreed on 20,000
uncorrectable codewords per second on channel 25 and a 40 → 0 dB SNR cliff on the
DOCSIS 3.1 carrier. Neither was real: the first was a `rate()` reset artifact from a
firmware table quirk, the second was the modem printing `0 dB` while re-acquiring a
carrier. The real impairment underneath was modest and persistent, confined to the
highest-frequency carrier, and pointed at a specific physical cause a cable technician
can act on. Three surfaces agreeing is not corroboration when they read the same
series; this skill is the reading rules.

**Two hard rules:**

1. **Fetch the reference first.** `get_reference_doc(name: "docsis-metrics-reference")`
   carries the metric table with current bands, the platform roster (which modems print
   codeword totals, and each platform's quirks), the physics, the pattern → cause →
   customer-effect table, the **artifact catalog**, and the PromQL cookbook. Do not
   grade from memory; the bands and the artifact list change.
2. **Always read per carrier.** Pin `channel_id`; never average across carriers. The
   single most diagnostic fact on the cable side is *which carriers* share a pattern.

## Procedure

### 1. Find the modem

- Resolve the network (`select-network` if the user did not fix one). Call
  `network_tech_stack`; **Connection technology: `cable`** names the gateway. If it
  reads `unknown` and the gateway is a plain router, this is **bridge mode**: sweep
  `find_device(network_id=…, device_class="cable_modem")` and `"cable_gateway"`. The
  modem is usually at `192.168.100.1` and is a *different* `device_id` from the gateway.
  Every query below uses the **modem's** `device_id`.
- `show_device` on the modem: vendor/model, and under the probes / `identificationLog`
  the monitoring rule name. Look the rule up in the reference's *Supported platforms*
  table: it tells you whether this platform prints **codeword totals** (→ the ratio
  grade applies) and which quirks to expect. If no DOCSIS series exist at all, follow
  *When metrics are missing* in `wan-metrics-reference` (credentials are load-bearing
  for cable) and stop here with that finding.

### 2. What did Focus Layer already detect?

`network_issues` over a window **wider** than the complaint (24 h minimum; a week for
"it's been slow lately"). Keep the DOCSIS rows (`probeType: pt_scripted`, metric
`docsis_*`) and the network's loss / DNS / HTTP rows. Then, per the reference's *Reading
DOCSIS issue rows*:

- **Group by `dimensions.channel_id`.** One channel with many rows is one bad carrier.
- **Prefer the long `mean_shift` row over the burst rows** on the same channel — since
  the level-score change it is the headline for a persistently bad carrier.
- **Discard artifact rows before reading severity:** `segment_after_mean=+Inf` (ratio
  denominator stalled), SNR rows with `segment_after_mean=0.00` (pre-fix SNR-0), and
  any uncorrectable rate in the tens of thousands per second (the mirrored-column reset;
  compare to the carrier's ~50k/s total). Say you discarded them and why.
- Note which grade the rows carry: `docsis_uncorrectable_codeword_ratio` (platforms
  with totals) or `docsis_uncorrectables_rate` (per-second fallback). On a platform
  with totals, per-second rows are from before the total series existed.

`event_evidence` on the surviving rows when you need the payload; `issue_chart` to see
the shape.

### 3. Read the carriers (only what step 2 left open)

Use the reference's cookbook queries with `timeseries_range` (`step` 5m, `max_points`
~60, `aggregation: max`; `drop_repeated: true` on lock / up gauges). In this order:

1. **Inventory:** `count by (modulation, direction) (sprinter_docsis_channel_locked{…})`
   — how many SC-QAM and OFDM/OFDMA carriers, and whether any are missing right now.
2. **Loss per carrier:** the `> 0`-guarded ratio division (platforms with totals) or the
   per-second rate (without). Beside it, `rate(docsis_codewords_total)` for the same
   carrier as the sanity bound. The bare derived name returns nothing.
3. **Margin per carrier:** the correctable ratio. High correctable with low
   uncorrectable is a warning, not loss.
4. **SNR per carrier** — a **gap** is "not reporting", never 0.
5. **Downstream power across all carriers at once** (instant): look for a **tilt** (the
   highest-frequency carrier several dB below the lowest) and for which tail.
6. **Upstream power trend** over days (`step` 1h): the pre-failure signature.
7. **Lock ratio and operational/boot state** with `drop_repeated` — partial service vs
   registration loss.

For a step in one series, `timeseries_analyze` on one pinned channel (`direction:
increase_bad`, `significant_value: 0.0001` for a ratio; `decrease_bad`, no floor, for
SNR).

### 4. Match the pattern

Use the reference's *Pattern → cause → what the customer felt* table. Write down, in
this order: **which carriers** (ids, modulation, and whether they are the
highest-frequency ones), **what they show** (ratio / SNR / power, with numbers and the
band), **the cause class** the table gives, and **the artifact check you ran**.

### 5. Did the customer feel it?

Correlate over the same window with what the network itself measured, and say which
statements are **measured** and which are **inferred**:

- The anchor ladder `sprinter_loss{target_class=~"tc_default_gateway|tc_first_isp_hop|tc_google_dns"}`
  and any loss / DNS / HTTP issues from step 2. Loss on the ISP hop and 8.8.8.8 with a
  clean gateway during a DOCSIS event is the measured customer effect.
- `docsis_connectivity_operational_up` / `docsis_boot_operational_up` dropping is a
  measured outage; carriers dropping out is a measured **partial service**.
- Throughput is **not measured** on most networks. "The connection was capped while the
  OFDM carrier was out" is an inference from partial service; state it as one.
- Loss below ~1% from uncorrectables is hidden by TCP for browsing and streaming and is
  felt by VoIP, video calls and gaming in proportion; say which apps.

`ioda` / `isp_info` when the pattern is plant-wide (all carriers) to see whether the
ISP has a regional event.

### 6. Report

**Audience.** Default `operator`; the finding's owner is the ISP whenever the pattern is a
plant fault, so the `escalation` paragraph is **appended automatically** in that case;
`end-user` on cue or offer. Rules and the full worked example for this exact case are in
`get_reference_doc(name: "report-audiences")` — fetch it the first time you render.

Follow the skeleton: **Verdict → Evidence → What we could not see → Next action.**

- **Verdict.** Up or degraded or down; which carriers; since when; owner (ISP plant, in-home
  wiring, modem, or "cable side eliminated").
- **Evidence.** Per carrier: ratio (or per-second rate on platforms without totals) with its
  rung, correctable ratio, SNR, power, re-acquisition count; the SC-QAM carriers' state as
  the contrast; power tilt; upstream trend; what the network's own probes showed over the
  same window.
- **What we could not see.** No throughput probe (say "inferred from partial service"); the
  issue rows you discarded as artifacts and why; any pre-window baseline you lacked;
  credentials missing.
- **Next action.** Whose move: ISP technician (with the paragraph below), in-home wiring
  check (splitters, unused ports, fittings), or look inward because the cable side is clean.

**`escalation` paragraph** — their vocabulary, no internal names, quotable verbatim:

> Technicolor CGM4981 (XB8), CM MAC 4C:D7:4A:25:75:44. Downstream OFDM carrier (ch 193,
> ~900 MHz): uncorrectable codeword ratio 7e-4 sustained (peaks 2e-3), correctable ratio
> 0.46, MER 39 dB when locked, carrier re-acquired 5× in 13 h (MER unreported ~10 min, then
> carrier absent ~10 min). All 28 SC-QAM carriers (ch 1–28, 256-QAM): 0 uncorrectables, MER
> 40+ dB. Downstream power −6 to −7 dBmV across the band; upstream 40–48 dBmV, stable.
> Measured at the modem's status page once a minute, 5 Sep 2026 03:00–21:00 PDT. The
> pattern is frequency-selective at the top of the downstream band: please check the drop,
> fittings and any splitter or amplifier for high-frequency roll-off or ingress.

**`end-user` paragraph** (on cue, or offered in one line when the request reads as
non-technical) — answers "is it my provider, my house, or my device", translates every
number into an effect, names one action; see the reference doc for the specimen.

## Traps

- **Do not report a `poor` band from `wan-metrics-reference` alone.** That doc grades
  the *worst* channel with `min()`/`max()`/`sum()` and cannot tell you which carrier or
  whether it was an artifact. It is the right tool for triage's yes/no; this skill is
  the right tool for the diagnosis.
- **Never average across carriers.** The bad OFDM carrier vanishes into 32 clean ones.
- **Never read a rate faster than ~50k/s per carrier as loss**, and never read a 1-minute
  `rate()` on codeword counters (the page refreshes every ~2 min on some firmware).
- **A gap is not a zero, and a zero is not a measurement.** SNR 0 = not reporting; a
  vanished channel series = the carrier dropped out of the modem's page.
- **A high correctable ratio is margin, not loss.** Report it as the leading indicator.
- **Cable is the one WAN technology where credentials are load-bearing.** Missing
  DOCSIS series with `HTTP_LOGIN_REQUIRED` and no `ct_http_credentials` channel means
  "add the modem's admin credentials", not "unsupported".
- **Bridge mode:** the box `network_tech_stack` calls the gateway is not the box that
  emits. Query the modem's `device_id`.
- **An IP is not an identity.** `192.168.100.1` exists on every cable network; always
  carry `network_id`.
- **Device link notes (replaced hardware).** When the ISP swaps a modem or an RMA unit arrives,
  the new box is a new device with a new `device_id`, and an operator may record that it
  replaced the old one. Tools then say so:
  - `find_device`, `list_devices` and `show_device` carry a `device_link` field. On the old
    box it reads `replaced by <new id> at <time>`; on the new box it reads
    `replaced <old id> (serial ...) at <time>; metrics for <new id> before <time> were copied from <old id>`.
  - The timeseries tools add a `device link: device <new id> replaced ...` note (in
    `device_link_notes`, or as leading lines) when you query the new box's `device_id`.

  What to do with it: use the **new** box's `device_id` for current state and for history.
  When the note says history `were copied`, a query on the new id already covers the time
  before the swap, and a change at `<time>` is the hardware swap, not a fault. When it says
  history `are being copied` instead, the copy is not finished: query the **old** id for the
  time before the swap. Never report the old box as down or missing: it was replaced.

## Hand off when a change explains the window

If `network_device_events` returns a `device_changed`, `replaced_by` or `replaces` row inside
the window you are analysing — or a device-link hint says "replaced by" / "were copied from" —
the two halves of that window are not the same device-configuration and must not be read as
one trend. Hand off to **`compare-before-after-change`** (`Skill`), which pins the change time
and compares matched windows either side of it. Say which change you found; do not average
across it.
