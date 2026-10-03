---
name: compare-before-after-change
description: >
  Compare a device's or network's behaviour before and after a change, and say whether the
  change helped, hurt, or did nothing. Use when the user asks "did the upgrade help", "did
  the firmware update help", "is it better since we replaced the modem", "since we replaced
  the router", "is the new one better than the old one", "before vs after", "after the
  update it got worse", "compare last week to this week", or names a date and asks what
  changed around it. Also use proactively when a window under analysis contains a
  `device_changed` event, a `replaced_by` / `replaces` row, a device-link hint ("replaced
  by", "were copied from"), or a `software_rev` change — mention the change rather than
  reporting the two halves as one trend. Read-only: this skill measures and reports, it
  never reboots, reconfigures, or writes anything.
allowed-tools: >
  Bash, Read, Grep, Glob, Skill, WebSearch,
  mcp__sprinter__network_issues,
  mcp__sprinter__network_device_events,
  mcp__sprinter__show_device,
  mcp__sprinter__find_device,
  mcp__sprinter__timeseries_range,
  mcp__sprinter__timeseries_analyze,
  mcp__sprinter__get_reference_doc
---

# Compare before and after a change

A device's software version changed, or device P was replaced by device S. The question is
always the same shape: **did the thing we changed make the measurements better, worse, or
neither?** The method is always the same too — pin the change time T, pick matched windows
either side of it, and compare like with like.

This is the mirror image of `when-did-this-start`: that one has a symptom and looks for its
onset, this one has a known change and looks for its effect. Fetch it with
`get_reference_doc(name: "when-did-this-start")` when the user's framing is actually
"something got worse, when?" rather than "did this change help?".

## 1. Open with `network_issues`

Always, before any raw series. It is catalog-tuned and timestamped, so it orients the whole
comparison in time and tells you whether the detectors already named something around T. Ask
for a window that **spans both sides** of the change, not just the "after" half — an issue
that was already firing before T is the single most common reason a change gets blamed for
something it did not cause.

## 2. Recognise the pattern

Either the user asks about a change, or you find one inside a window you were already
analysing. In the second case **say so proactively**: a `device_changed` row in the middle of
a trend means the two halves are not the same device-configuration and should not be averaged
together.

## 3. Pin T

```
network_device_events(device_id=…, network_id=…,
                      kinds=["device_changed", "replaced_by", "replaces"],
                      start=<well before the suspected change>)
```

`kinds` is a **list**, not a comma-joined string. Pass `start` explicitly: the tool's default
window is the last 24h, so a change from last week simply will not appear.

**Read the bound, do not guess it.** A `device_changed` row's own timestamp is when the new
value was *first seen*; the sentence also carries `last seen as <old> at <T0>`, the last
observation still carrying the old value. So the change happened in `(T0, row-timestamp]` —
which can be a wide bracket if the device is polled slowly.

**Event times carry the network's offset.** `network_device_events` renders each row in the
network's timezone with a numeric offset (e.g. `2026-10-02T02:54:46-07:00`), while
`sprinter_boot_epoch_seconds` is epoch seconds and the examples below are written in UTC (`Z`).
Compare them as instants, never by clock reading. To use T as a `timeseries_range` bound, pass
the timestamp exactly as returned — offset included; never drop the offset or re-label it `Z`.

**Narrow it with the reboot.** A software-version change almost always coincides with one:

```
timeseries_range(query='sprinter_boot_epoch_seconds{device_id="…"}', …)
timeseries_range(query='sprinter_uptime_resets{device_id="…"}', …)
```

**Do NOT pin `uptime_of`.** Its value is set by whichever monitoring rule read the uptime, so
it is platform-specific: `snmp_system_uptime` emits `uptime_of="system"`, the Starlink dish's
`starlink_dish_signal` emits `uptime_of="starlink"`, and other platforms will add their own.
Pinning `uptime_of="system"` returns an **empty series** on every device that is not SNMP —
which the next rule then tells you to report as "the reboot time could not be established".
That is a false negative on a device whose reboot is sitting right there, and it costs you the
single most useful narrowing of T. Query without the label and read the value back from the
returned series; use `uptime_of=~".+"` if you need an explicit matcher.

Read the VALUE, not the presence: `sprinter_boot_epoch_seconds` is the boot instant, so a step
in its value IS the reboot, to the second. Pick the reboot that falls inside the bracket. These
are kept ~90 days, far longer than the 14-day event log, so they often outlive the event that
sent you looking.

**An absent reboot means `unknown`, not "no reboot".** `sprinter_boot_epoch_seconds` has a
documented silent-failure mode: a duplicated `uptime_of` label kills the recording rule, and
it kills `sprinter_uptime_resets` quietly. If the series is simply missing, say the reboot
time could not be established — do not conclude the device never rebooted.

A date the user gives you is a starting point for the search, never T itself.

**The moment T is bounded, check it against step 1's issues.** Do not defer this to step 7:
if an issue straddles the bracket, no window either side of T is comparable, and every
measurement you take afterwards is wasted. The overlap is usually already visible in the
`network_issues` output you opened with — compare each issue's `startTime`/`endTime` against
the bracket before going further.

Measured 2026-10-01: a printer's change bracketed to `(22:54Z, 04:30Z]` sat **entirely
inside** a 7-hour `lan_wide_connectivity_loss` affecting 9 of 16 devices on that network,
the printer among them. Any before/after comparison there measures the outage, not the
change. That is a "cannot tell" verdict, and it is available for free at step 1.

### Changes older than 14 days

The event log is kept 14 days. Past that, recover the boundary from the info series:

```
timeseries_range(query='max by (software_rev) (sprinter_deviceInfo{device_id="…", network_id="…"})',
                 drop_repeated: true)
```

Four things to know before you trust it:

- **The value is a constant `1`.** A version change shows up as one series *ending* and
  another *starting*, never as a value moving. With `drop_repeated` each surviving point is
  that version's first sample, which is the onset you want — but do not go looking for a
  transition in the numbers.
- It exists mainly for devices with a scripted monitoring probe (pinged targets also get a
  thinner version of it), so absence is not evidence.
- The boundary lags the real change by up to one collection interval.
- Empty values flap, so ignore transitions into and out of `""`.

Aggregating with `max by (software_rev)` deliberately drops `device_name`, `device_address`
and `device_mac`, so a rename or a new lease does **not** break the series — that caveat
applies only if you read the raw series instead.

## 4. Check it is a real change, not a labelling disagreement

Identity fields currently ping-pong between sources: `iPhone` ↔ `iPhone / iPad`, `Apple` ↔
`Apple, Inc.`, `OpenSSH 10.3` ↔ `25.6.0`. Around 108 such transitions a day across the fleet
are **not** real changes.

**Run this check every time, before any window arithmetic.** It is cheap, and skipping it
is how you end up writing a confident upgrade analysis of a spelling change.

```
timeseries_range(query='max by (<field>) (sprinter_deviceInfo{device_id="…"})',
                 start=<~7 days before T>, end=now, drop_repeated: true)
```

`<field>` is the label matching the changed field: `software_rev`, `model` (for
`product_name`), or `vendor`. Aggregate by **both** when two fields changed in the same pass.

**Read the series COUNT, not the values.** The metric's value is a constant `1`, so nothing
"transitions" — what tells you is how many series come back:

- **One series** → the old value is genuinely gone. Consistent with a real change; continue.
- **Two or more, overlapping in time** → both values are live simultaneously and the device
  is flapping between sources. **Stop.** This is a labelling disagreement.
- **Two or more, but the old one ENDS near T and the new one STARTS there** → a real change.
  That is the shape an upgrade makes.

Corroborate with the reboot from step 3. The combination is what decides it:

| Series over 7 days | Reboot near T | Read as |
|---|---|---|
| old ends, new starts | yes | a real change — proceed |
| old ends, new starts | no | probably real, say the reboot could not be confirmed |
| both live throughout | no | a flap — stop |
| both live throughout | yes | a flap that happens to coincide with a reboot — still stop |

Other tells, once the series check is ambiguous:

- the "new" value is a prefix, suffix or re-spelling of the old one (`Apple` → `Apple, Inc.`);
- the value is not a version at all (`Y`, `linux`, `Debian 13`);
- aggregating two changed fields returns their full cartesian product, which means each is
  flapping independently rather than the device having changed.

**Worked example (measured 2026-10-01).** A Brother printer reported
`software_rev "Y  " → "1.70"`. `max by (software_rev)` over 7 days returned **two** series,
`1.70` and `Y  `, both continuously present — and `sprinter_boot_epoch_seconds` was constant
since July, so no reboot. Verdict: labelling disagreement, analysis stops. A PurpleAir sensor
reported `vendor` and `product_name` changing in one pass; `max by (vendor, model)` returned
**all four** combinations across the week. Same verdict. Neither was an upgrade, and in both
cases the only thing standing between the request and a fabricated answer was this step.

If it is a flap, say so plainly and stop — do not analyse it as an upgrade.

**What a REAL one looks like, for contrast (measured 2026-10-02).** A Starlink dish returned
**12** series over 6 weeks, each ending exactly where the next began —
`…09.16.cr87006.55184` ran 10-01 08:30Z to 10-02 09:30Z, then `…09.18.mr87170.1` took over.
Many series, zero overlap: that is a version history, not a disagreement. Count the series,
then check whether their spans ABUT or OVERLAP; only the overlap means flap.

## 5. Choose the windows

- **Equal lengths** on both sides of T.
- **Match hours and weekdays.** A Tuesday-morning "before" against a Saturday-night "after"
  measures the weekend, not the change.
- **Exclude a settling period derived from the data**, not a fixed number of minutes: start
  the "after" window once the locked-channel ratio and SNR are steady again and the gaps have
  stopped. Read how long that took from the series.
- **On a Technicolor XB8, the ~10-minute OFDM re-acquisition cycle repeats** on a marginal
  carrier, on both sides of T. That is a confounder, not settling — if you treat it as
  settling you will keep sliding the window and never find a clean start. Fetch
  `get_reference_doc(name: "docsis-metrics-reference")`, which also covers the mirrored
  column-0 codeword artifact and the 2-minute counter refresh that forces a 5–15 minute rate
  window.

## 6. Compare like with like

**Software change (same device).** Use **rates and ratios, never raw counters across T** —
counters reset on reboot, and a reboot is usually exactly what T is. A "99% drop in
correctable codewords" that is really a counter reset is the classic false win here.

**Replacement (P → S).** Once the metric backfill has finished, one device id covers both
sides. Before that, query P for the "before" window and S for the "after" one; the device-link
hint in the tool output names which id to use. **Name any metric that exists on only one of
the two models** — a comparison that silently drops a metric reads as an improvement.

### Worked example: a real change, end to end (measured 2026-10-02)

Starlink dish, `2026.09.16.cr87006.55184 -> 2026.09.18.mr87170.1`.

- **Bracket (step 3).** The `device_changed` row read "first seen 09:02:46Z, last seen as the
  old version at 06:33:10Z" — a 2h29m bracket.
- **Narrowed to the second.** `sprinter_boot_epoch_seconds` stepped to `1790927872` =
  **07:57:52Z**, inside the bracket. That is T. Do not skip this: the bracket alone would have
  put T up to 90 minutes late, and the windows would have straddled the reboot.
- **Settling, derived (step 5).** The post-reboot route change finished at 08:10Z, and
  `pop_ping_drop_ratio` showed one isolated 5-minute 5% blip at 08:30Z, clean from 08:35Z.
  Settling = **~62 min**, so the after window opens at 09:00Z.
- **Windows (step 5).** `09:00Z–23:45Z` on each of the two days: equal length, same clock
  hours, and — the part worth copying — **both days happened to have a reboot at ~08:00Z**, so
  the two windows sit at an identical offset from one. When the device reboots on a cycle, use
  that cycle to match the windows instead of fighting it.
- **Verdict: no measurable change.** POP latency median 22.4→23.4 ms (+4.2%) while p95
  35.3→33.5 ms (-5.1%); downlink median +8.0% while p95 -3.8%. **Median and p95 moving in
  OPPOSITE directions, both by ~1 ms or a few percent, is the signature of noise** — not of a
  small improvement. Resist reporting the flattering half.
- **The control that made it trustworthy.** `obstruction_fraction` was identical either side
  (0.0023). It is physical, so firmware MUST NOT move it; a metric that cannot legitimately
  change is the cheapest check that the two windows are comparable at all. Pick one and state
  it.

## 7. Rule out confounders

You already checked step 1's issues against the bracket back in step 3. This is the sweep for
everything else: other `device_changed` rows on the same device, WAN/ISP events, and topology
changes. If a second change lands inside either window, the comparison cannot separate them —
say so rather than attributing the whole delta to the change you were asked about.

Re-run `network_issues` here only if the windows you settled on in step 5 reach outside the
range you opened with.

## 8. Report

**Audience.** Default `operator`; append the `escalation` paragraph automatically when the
finding's owner is external (the ISP, or a vendor whose firmware caused a regression);
`end-user` on cue or offer. Fetch `get_reference_doc(name: "report-audiences")` the first
time you render.

Follow the skeleton: **Verdict → Evidence → What we could not see → Next action.**

The verdict is one of: **not a real change**, **better**, **worse**, **no measurable change**,
or **cannot tell**.

Two of those are answers, not failures, and both are commoner than the other three:

- **"Not a real change"** is the step-4 outcome — the field flapped between spellings and
  nothing happened to the device. Report the series evidence and stop; do not soften it into
  "no measurable change", which implies a real change with no effect.
- **"Cannot tell"** is correct whenever the windows were not comparable: an issue straddling T,
  a second change inside a window, or a settling period that never settles.

Evidence must carry:

- the change, and **how T was found** — including the bracket, e.g. "between 04:12 and 12:03
  on 30 Sep, narrowed to the 04:40 reboot";
- the two windows used, and the settling period excluded, with the reason it was that long;
- a per-metric before/after table with **median and p95** and the units;
- the confounders you ruled out.

"What we could not see" must name:

- metrics that exist on only one side of a replacement;
- any coverage note returned by the tools — in particular a **truncated** event read, which
  means the oldest part of the window was cut and there may be earlier changes you did not
  see;
- an unestablished reboot time, if step 3 could not pin one.
