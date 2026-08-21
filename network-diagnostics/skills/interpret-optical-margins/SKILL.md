---
name: interpret-optical-margins
description: >
  Translate a fiber/PON issue raised on an optical MARGIN metric back into the
  measured power and the transceiver's own alarm thresholds, in dBm. Use
  whenever an issue, outlier cluster, or change point names a metric ending in
  `_margin_low_db` or `_margin_high_db` — or whenever you are about to tell a
  user or their ISP anything about optical power. Margin is the surface we
  GRADE on; it is not a number anyone outside this system can act on, and
  quoting it to an ISP will not survive the conversation.
argument-hint: "[device-name-or-ip]"
allowed-tools: >
  Bash, Read, Grep, Glob, Skill,
  mcp__sprinter__find_device,
  mcp__sprinter__show_device,
  mcp__sprinter__network_issues,
  mcp__sprinter__issue_chart,
  mcp__sprinter__event_evidence,
  mcp__sprinter__timeseries_instant,
  mcp__sprinter__timeseries_range,
  mcp__sprinter__timeseries_analyze,
  mcp__sprinter__get_reference_doc
---

# Reading an optical margin issue

## Why this skill exists

Fiber issues are detected on **margin** metrics — `optical_rx_power_margin_low_db`
and friends. A margin is `measured_value − the optic's own alarm threshold`, so
"0 dB" means the transceiver is exactly at the limit it publishes for itself.

We grade on margin for one specific reason: **the fleet is not one platform.** A
GPON optic alarms on receive power at −28.8 dBm and an XGS-PON optic at −29.2,
with the overload ends 3 dB apart, and the model number does not predict which
you have. Analysis profiles are keyed by metric and target class with **no
per-device dimension**, so no single absolute band can be correct for both.
Margin normalises that away: both optics reach "0 dB of margin" at their own
limit.

That property is what makes margin the right thing to *detect* on and the wrong
thing to *report*. Nobody outside this system uses it. A subscriber cannot phone
their ISP and say "my margin dropped to 2.1 dB" — the ISP will ask what that
means, and any answer requires explaining our internal convention. What the ISP
expects, and what every optical-power troubleshooting guide on earth is written
in, is: **"receive power is −26.7 dBm and this ONT alarms at −28.8."**

So: detect on margin, **always report in dBm.**

## The rule

> **Never put a margin value in anything a human reads** — not the issue
> summary, not a report, not a chat answer, not an ISP escalation. Translate it
> first. Margin may appear as supporting detail *after* the dBm statement, never
> instead of it.

## Translation table

Every margin metric has a measured value and two published thresholds beside it
in VictoriaMetrics. All are plain gauges on the same device; no computation is
required — **read the real numbers rather than reconstructing them.**

| Issue metric                      | Measured value         | Alarm threshold                   | Warning threshold                   |
|-----------------------------------|------------------------|-----------------------------------|-------------------------------------|
| `optical_rx_power_margin_low_db`  | `optical_rx_power_dbm` | `optical_rx_power_alarm_low_dbm`  | `optical_rx_power_warning_low_dbm`  |
| `optical_rx_power_margin_high_db` | `optical_rx_power_dbm` | `optical_rx_power_alarm_high_dbm` | `optical_rx_power_warning_high_dbm` |
| `optical_tx_power_margin_low_db`  | `optical_tx_power_dbm` | `optical_tx_power_alarm_low_dbm`  | `optical_tx_power_warning_low_dbm`  |
| `optical_tx_power_margin_high_db` | `optical_tx_power_dbm` | `optical_tx_power_alarm_high_dbm` | `optical_tx_power_warning_high_dbm` |

The identity that defines them:

```
margin_low  = measured − alarm_low        (headroom above the floor)
margin_high = alarm_high − measured       (headroom below the ceiling)
```

Both are **monotone decreasing-is-worse**: a margin *falling* toward 0 is
deterioration on either tail, even though the underlying power is moving in
opposite directions. This is the one place margin actively misleads if you
report it raw — a "decreasing" high-margin means power is **rising**.

## What each tail actually means

The two tails are different faults with **opposite remediations**. Getting this
backwards sends a technician to the wrong end of the fiber.

| Tail                | Physically                    | Cause                                                                | Remediation                     |
|---------------------|-------------------------------|----------------------------------------------------------------------|---------------------------------|
| Rx margin **low**   | Too little light arriving     | Dirty or bent fiber, bad splice, failing OLT laser, over-long run     | Inspect/clean the fiber path    |
| Rx margin **high**  | Too much light arriving       | ONT sited too close to the OLT with no attenuator; saturating the Rx  | **Add an attenuator**           |
| Tx margin **low**   | ONT laser output collapsing   | Transceiver aging or failing                                          | Replace the ONT                 |
| Tx margin **high**  | ONT laser over-driving        | Transceiver out of spec                                               | Replace the ONT                 |

## Procedure

1. **Identify the metric and window.** From `network_issues` or the issue
   payload, note which margin metric fired and over what time range.
2. **Read the three real series** over the same window — measured value, alarm
   threshold, warning threshold — with `timeseries_range`. Use the SAME window
   as the issue, not "now": a resolved issue's numbers are in the past.
3. **Check `optical_ddm_parsed` for the same window.** If it read 0 at any
   point, the margins and thresholds for that period are **absent, not clean**,
   and any conclusion drawn across the gap is unsound. Say so rather than
   interpolating.
4. **State the finding in dBm**, with the threshold beside it, and name the
   tail's physical meaning from the table above.
5. **Sanity-check the arithmetic** the first time on a device: measured minus
   alarm threshold should equal the reported margin. A mismatch means the optic
   was swapped mid-window (the thresholds are per-transceiver constants, so they
   step when hardware changes) — which is itself the finding.

## Worked example

An issue fires: `optical_rx_power_margin_low_db`, change point, margin fell from
9.1 to 2.1 dB.

Read the real series over the issue window:

```
optical_rx_power_dbm             -19.7  ->  -26.7
optical_rx_power_alarm_low_dbm   -28.8      -28.8     (constant — same optic)
optical_rx_power_warning_low_dbm -27.9      -27.9
```

**Do not report:** "Optical margin degraded from 9.1 dB to 2.1 dB."

**Report:**

> Receive optical power on the ONT fell from −19.7 dBm to −26.7 dBm. This
> transceiver warns at −27.9 dBm and alarms at −28.8 dBm, so the link is now
> about 1.2 dB from its warning threshold and 2.1 dB from alarm. A 7 dB drop
> with no change to the ONT points at the fiber path — a dirty or bent
> connector, a failing splice, or a change upstream at the OLT.

That paragraph is quotable to an ISP verbatim. The first version is not.

## Notes and traps

- **The thresholds are read from the optic, not configured by us.** Never
  substitute a "typical GPON" figure from memory or a datasheet — the whole
  reason these metrics exist is that the fleet's optics disagree. Read the
  metric.
- **Thresholds are mapping-only** (`analyze: false`): they are constants, so
  they never raise issues themselves. A *step* in one is still meaningful —
  it means the transceiver was replaced.
- **`optical_alarm_active{channel=...}` is the optic's own verdict.** If it is
  1, the transceiver is asserting the alarm itself, which is stronger evidence
  than our band and worth leading with.
- **`optical_rx_power_dbm` and `optical_tx_power_dbm` also carry their own
  absolute health bands.** Those bands are the intersection of both optic
  generations' healthy regions — deliberately conservative, and a backstop for
  when the diagnostic block cannot be parsed. When they disagree with the
  margin, the margin is the more precise statement because it is measured
  against *this* optic.
- **Temperature and laser bias have no margin metrics.** Temperature is graded
  on an absolute band that is stricter than the optic's own (70/80 °C versus its
  90/95), and `optical_tx_bias_raw` is a vendor-scaled counter that is only
  meaningful as a trend. Do not invent margins for them.
