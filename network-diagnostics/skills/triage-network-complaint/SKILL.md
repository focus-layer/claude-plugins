---
name: triage-network-complaint
description: >
  Localize a network problem to a layer — ISP/WAN, gateway, DNS, Wi-Fi, or a
  single device — using cheap broad checks, then dispatch to the right
  specialist skill. Covers both vague complaints ("internet is slow", "video
  calls keep freezing", "it works then stops", "pages won't load", "this device
  keeps going offline") AND a specific already-observed symptom the user wants
  explained ("there was 100% packet loss around 10:00 today, why?", "the
  internet dropped for a few minutes this morning", "what caused the outage at
  <time>?", "why did latency spike last night?"). Use this FIRST whenever the
  user names a symptom but not its cause — naming a time, a metric, or a
  network does NOT make a request too specific for triage; localization is
  still the first move. Read-only: it observes and routes, it does not change
  anything.
argument-hint: "[network-id-or-name] [optional: affected device]"
allowed-tools: >
  Bash, Read, Skill,
  mcp__sprinter__ask_user,
  mcp__sprinter__get_reference_doc,
  mcp__sprinter__show_network,
  mcp__sprinter__list_networks,
  mcp__sprinter__find_network,
  mcp__sprinter__find_device,
  mcp__sprinter__show_device,
  mcp__sprinter__device_presence_history,
  mcp__sprinter__network_issues,
  mcp__sprinter__event_evidence,
  mcp__sprinter__issue_chart,
  mcp__sprinter__network_tech_stack,
  mcp__sprinter__topology_path,
  mcp__sprinter__topology_neighbors,
  mcp__sprinter__isp_info,
  mcp__sprinter__ioda,
  mcp__sprinter__network_info,
  mcp__sprinter__network_ping,
  mcp__sprinter__network_dns_lookup,
  mcp__sprinter__network_speed_test,
  mcp__sprinter__network_jitter_test,
  mcp__sprinter__network_traceroute,
  mcp__sprinter__show_probes,
  mcp__sprinter__timeseries_instant,
  mcp__sprinter__timeseries_range,
  mcp__sprinter__timeseries_analyze,
  mcp__sprinter__traceroute_history
---

# Triage a Network Complaint

> **All data comes from `mcp__sprinter__*` tools — nothing else.** Do not shell out
> to `promql`, `query_graph`/Cypher, `psql`, `logcli`, or `kubectl`. Those reach the
> cluster's internal datastores, which exist only for an operator inside the network:
> in a customer's Claude Code, Claude Desktop, or ChatGPT session the command is
> missing or the host does not resolve, and the session dies holding half a
> diagnosis. It also passes testing on a developer laptop, so there is no local
> signal that it broke. `timeseries_range` and `timeseries_instant` accept arbitrary
> PromQL, and `timeseries_analyze` computes change points, outlier clusters, and
> baseline stats for one series — between them they cover what a raw VM query would
> have done. If an MCP tool cannot express what you need, report the gap; do not
> route around it.

> **Output discipline.** Investigate quietly. Do NOT narrate your process to the
> user — no "let me…", no "now I'll…", no announcing which tools you are loading
> or calling, no step-by-step play-by-play, and no explaining your reasoning or
> the platform/coverage landscape (e.g. "since this is a UniFi device…", "Focus Layer
> supports several platforms…"). Call tools without describing the act of calling
> them. Surface only what matters to the user: the findings, the supporting
> evidence, and the verdict/next step. Keep any interim text minimal.

Most real complaints are **underspecified**: "is the internet slow today?" can
mean a download-speed drop, DNS failures, intermittent dead spells, one bad
device, or an ISP outage. This skill's job is **localization, not deep
diagnosis** — narrow the report to a layer, then hand off. It is **read-only**.

> **A precise symptom is still a triage job.** "There was 100% packet loss to
> the internet at 10:00 UTC — why?" names a time, a metric, and a network, and it
> is tempting to read that as already-localized and jump straight to inspecting
> the hardware you suspect. It is not localized: it says *that* connectivity
> broke, never *where*. Every one of ISP outage, modem fault, a flapping router
> WAN port, a LAN storm, and a dead agent produces exactly that sentence. Run
> Step 2 in order — issues, then the anchor ladder — before you open any single
> device. Starting with a device-by-device walk is how a session spends twenty
> calls arriving at a place two calls would have reached, and it biases the
> conclusion toward whichever box you happened to look at first.

> **Do not guess to fill a vacuum. This is the single most important rule here.**
> When the instruments cannot see the failing path, or the broad checks all come
> back clean, the correct next move is **not** to manufacture a plausible root
> cause — it is to **stop and either report the blind spot or ask the user one
> concrete question.** A confident wrong diagnosis is worse than an honest "I
> can't see this from here, and here is what would settle it," because the user
> acts on it. The two failure modes to refuse:
>
> - **Hypothesis inflation.** Producing a ranked shortlist of causes *none of
>   which you can distinguish from your vantage point*, then defending it. If you
>   cannot measure which item on the list is true, the list is a guess dressed as
>   analysis — say you cannot tell them apart, do not present them as findings.
> - **Vantage blindness.** Reading a number without asking *where it was measured
>   from*. A probe from a wired agent says nothing about a Wi-Fi client's path
>   (see the **coverage gate**, Step 1.5). A single stuck/canned value (identical
>   to the millisecond across runs) is a measurement artifact, not a signal.
>
> When you are about to write "it's probably X" and you have no measurement that
> singles out X, that sentence is the trigger to invoke the **Ask the user**
> discipline (Step 1.5) or to report the blind spot honestly instead.

No working-directory check needed (no repo writes). It dispatches to other
skills, which run their own checks.

## Step 1 — Minimal intake (don't over-ask)

Pin down only what changes the search:
- **Scope:** one device, or everything? (One device → likely that device's
  link, not the network.)
- **When / since when, constant or intermittent?** Intermittent + time-of-day
  pattern points at congestion/Wi-Fi; constant points at config/ISP.
- **What "slow/broken" means to them:** can't load pages (DNS/WAN), buffering
  video (throughput/jitter), calls drop (loss/jitter/roaming), specific app.

Resolve the network and any named device. The user may belong to more than one
organization (org); MCP tools span all of them, so **infer first**: if the prompt
names a device, call `find_device` (omit `network_id` to search every org) — a
single match resolves the `network_id` and its org. If it names a network by name,
call `find_network` (name prefix, across all orgs). Otherwise fall back to
`list_networks` (spans all orgs; rows carry `tenant_id` + `tenant_name`). If a name
is ambiguous across **two orgs**, disambiguate by org via `ask_user`. Get
`network_id` (the org/`tenant_id` follows from it).

**An IP does not identify a device — `(network_id, address)` does.** Networks have
overlapping IP ranges, so an address search can return several *different* devices on
different networks. This bites hardest here, because triage gets pointed at exactly the
addresses that collide: `.1` and `.254` gateways. In one fleet `192.168.1.254` is an AT&T
gateway on **two** networks — same vendor, same name, different box. When an address
matches more than one device, `ask_user` showing **network — vendor — product**; never
pick one, and never treat a single match as proof of uniqueness. See "An IP address is NOT
a device identity" in the plugin CLAUDE.md.

**Correlating across networks (a multi-site tenant).** A tenant who owns several sites may
reasonably ask a cross-network question — *"is the ISP flaky at all four offices?"*, *"did
the same firmware break every location?"*. That is answerable, but **never correlate by IP**:
two gateways both at `192.168.1.254` are unrelated boxes at unrelated sites. Correlate on
`device_id`, `mac`, `serial_num`, or `vendor`+`product_name`, and label every finding with
its **network** — an answer that says "the gateway is dropping packets" without saying
*which site* is worse than no answer.

**Hold onto the `network_id` and pass it into every network-scoped call for the
rest of the session** — `network_tech_stack`, `show_network`, `network_ping`,
`timeseries_*`, `network_issues`, and the rest all require it, and a call made
without it is rejected (you then have to redo it). It does not persist
automatically between tool calls; treat it as a variable you thread explicitly.
`show_device` and `find_device` echo the `network_id` in their results, so if you
ever lose it, read it back from the last such result rather than dropping the
argument.

## Step 1.5 — Coverage gate + ask, don't guess

Before you localize, answer one question about **yourself**: *can Focus Layer
actually observe the path the user is complaining about?* The scratch-real
failure this skill exists to prevent was diagnosing a **Wi-Fi client's**
intermittent failure from a **wired agent** on a **Google Nest** network — a path
that is **structurally invisible** to Focus Layer here — and then inventing an
"IPv6 cold-start" root cause from a stuck ping value measured on the wrong side of
the link. Do the coverage check *first*, so a blind spot is named as a blind spot
and not filled with a guess.

**A. Run the coverage check (cheap, from data you already fetch in Step 2).**
Establish the observability of the failing path:

- **Agent vantage.** `show_network`'s agent roster: how many agents, and are any
  on the **same segment as the complaint**? An intermittent **Wi-Fi** complaint
  when the only agent is **wired to the router** means *no probe you run touches
  the failing path* — a wired-agent ping/traceroute/HTTP measures the router's
  uplink, not the client's Wi-Fi. Say that out loud in the verdict; never present
  a wired-vantage measurement as evidence about a wireless client.
- **Platform telemetry.** From `network_tech_stack`'s per-platform notes: does the
  WiFi platform expose **per-client** data at all? **Google Wifi / Nest exposes no
  client roster or per-client link telemetry** — so for a single-client Wi-Fi
  complaint on Nest there is **nothing to read**, by design. An empty Wi-Fi column
  here is a **structural limit**, not a fault and not something to keep probing.

**A2. Read the coverage block before calling any layer clean.**

The gate above is about *vantage* — whether a probe can reach the failing path.
This one is about *time*: every answer you get comes from a store with a
retention horizon, and the horizons differ by a factor of about 2000. Metrics go
back roughly 84 days; the raw probe archive 7; traceroute 7; a tenant's device
and evidence history can be as short as **one hour**. A window that reaches past
one of them comes back **shorter, not empty** — fewer issues, an empty topology,
a device that "was never there" — and that reads exactly like a quieter network.

So: `network_issues` may return a `coverage_gaps` array, and `topology_neighbors`
/ `topology_path` / `device_presence_history` / `traceroute_history` may carry a
coverage note. **When one is present, read it before you conclude anything about
that layer.** The rule in one line:

> **A source that could not be consulted is not a source that came back healthy.**

The `state` tells you what to do:

| state | What it means | What you must NOT do |
|-------------------|--------------------------------------------------------|--------------------------------|
| `covered` | The source answered for your whole window. | — |
| `current_only` | It can tell you what is true NOW, not what was true then. | Quote today's value as the incident's. |
| `horizon_limited` | Its data begins at `data_from`, after your window's start. | Read the short result as "nothing happened". |
| `source_absent` | This network has NO such data, ever. | Report it as expired, or as a change. |
| `source_error` | We tried to read it and could not. | Treat the empty answer as a finding. |

`current_only` is the one that misleads quietly. Asking what a device was
attached to last night, against a tenant with a 24-hour window, returns **today's
attachment** — a substitution, not a memory. If a coverage note says
`current_only`, say so in the verdict rather than presenting the value as
historical.

`show_device` has the same limit and does not report it: it **answers about now**
and cannot answer about your incident window. Every evidence section carries its
own `observed_at` — check it against the complaint window before treating any
section as evidence about the incident.

**Warnings now ride on SUCCESSFUL results.** An empty `timeseries_range` or
`timeseries_instant` result is a success carrying a `warning` field, not an
error. Read the warning on a successful response; if you only check for errors
you will read "no data, no error" as "the metric was flat".

**B. If the failing path IS observable, investigate normally** (Step 2 onward) and
report from the data. The gate only diverts you when you are blind.

**C. If the failing path is NOT observable — OR your Step 2 broad checks all come
back clean — do NOT start hypothesizing. Ask the user ONE concrete question**
(`ask_user`) to narrow the complaint into something you *can* act on, or to
confirm the symptom shape. Pick the one question whose answer most changes the
next step. Rules for the question:

- **Offer coarse estimate buckets, and always an explicit "I don't know / not
  sure" choice.** End users often genuinely don't know the specifics, and a
  question with no escape hatch pushes them to invent an answer — which is just
  *their* guess replacing yours. A good `ask_user` for frequency:
  - `A few times an hour`
  - `A few times a day`
  - `Once every few days`
  - `Only after the Mac wakes from sleep / reconnects`
  - `Not sure / I don't know`
- **Ask about things the user can actually observe**, not internal state:
  *how often*, *all sites or only some*, *one device or several*, *always after
  sleep/reconnect*, *does a wired device on the same network have the problem
  too*. The last one is gold on a blind Wi-Fi network — a wired device that is
  clean while the Mac fails localizes to Wi-Fi without any Focus Layer Wi-Fi
  telemetry at all.
- **One question, not an interrogation.** This is triage, not a form. If the first
  answer resolves the ambiguity, proceed; only ask a second if it still changes
  the layer.

**D. If the path is unobservable AND the user cannot add detail, stop guessing and
close honestly.** Do not emit a ranked shortlist of causes you cannot tell apart.
Instead:

1. **Name the blind spot and why it exists** — "Focus Layer can't see the Wi-Fi path
   on this network: the only agent is wired to the router, and Google Nest reports
   no per-client Wi-Fi telemetry, so no probe I can run touches the failing link."
2. **State the positive findings you CAN stand behind** — the layers you *did*
   clear (wired path clean on v4/v6, WAN link healthy, DHCP DORA_OK, no rogue
   server, no presence flapping). These are real and they eliminate whole layers;
   that is a genuine result, not a shrug.
3. **Hand the user concrete client-side capture steps to run during a failure,
   before the workaround** (e.g. before toggling Wi-Fi), each labelled with how to
   read it. For a MacBook:
   ```
   ping -c5 192.168.86.1                 # gateway reachable over Wi-Fi?
   ping -c5 8.8.8.8                      # IPv4 to internet
   ping6 -c5 2001:4860:4860::8888        # IPv6 to internet
   dig +short example.com                # DNS resolving?
   ```
   Tell them what each split means (only `ping6` fails → IPv6-over-Wi-Fi path;
   gateway ping shows loss → Wi-Fi link/roaming; DNS fails but pings fine → DNS).
   These are **bisection tests the user runs**, framed as "here's how to catch it,"
   not a root cause you claim to have found.

The discipline in one line: **measure what you can, ask about what you can't, and
never let a gap in coverage become a guess in the report.**

## Step 2 — Broad instruments first (cheap, in parallel)

Run the wide checks before any narrow probe. These tell you which layer to
suspect:

- **Orient FIRST — run both `network_tech_stack` and `show_network`** once at the
  start (they are complementary, not interchangeable, and together give you the
  full lay of the land before any narrow probe):
  - `network_tech_stack` — the **hardware**: ISP + connection technology, the
    gateway, infrastructure (switches/routers), and the **WiFi platform
    inventory**, each device with its `device_id` and IP so you can hand those
    straight to `show_device` / `network_ping` / `snmp_get` without a lookup. It
    also carries **authored per-platform notes on what each platform can and
    cannot tell you** (e.g. Google Wifi exposes no client roster, so wireless-vs-
    wired and AP association are unknowable). Read those notes before concluding
    anything is broken: an empty WiFi column is often a **structural platform
    limit**, not a fault or a Focus Layer bug — say so instead of chasing it.
  - `show_network` — the **operational state** the tech stack does not carry:
    DHCP servers + config, the **agent roster with live online status** (is the
    network even being observed right now?), LAN addressing, and subnets. A
    complaint that turns out to be "the agent is offline" or "DHCP is
    misconfigured" is caught here, cheaply, up front.
- **What has Focus Layer already flagged?** `network_issues` over the complaint
  window (loss/RTT shifts, outlier clusters, DNS/DHCP/HTTP probe issues). This
  is the highest-signal first look — for any metric a predefined probe collects
  (`pt_ping`/`pt_multi_ping`/`pt_dns`/`pt_http`/`pt_irtt`/`pt_dhcp`), prefer
  these computed issues over eyeballing `timeseries_range`: each carries
  catalog-tuned baselines (`baseline_mean`, `baseline_p95`,
  `segment_before_mean`/`segment_after_mean`). Check `truncated` before
  reporting a count. Use `issue_chart` to see a flagged metric over time, and
  `event_evidence` (with the response's `analysis_run_id` + an issue's event
  IDs) to pull the evidence behind a flagged issue before dispatching on it.
- **The standing anchor ladder — *where on the path* did it break?** For any
  connectivity, loss, or "internet was down" complaint this is the cheapest
  decisive read available, and it comes **before** any interface-by-interface
  walk. Focus Layer already pings three fixed points from the agent on every
  network, continuously. Their `target_class` is what names them (get the
  `probe_id` for each from `show_probes`):

  | Probe                     | `target_class`       | What loss on it proves                     |
  |---------------------------|----------------------|--------------------------------------------|
  | `ping_default_gw`         | `tc_default_gateway` | the agent's LAN path to the router          |
  | `ping_isp_hop_<ip>`       | `tc_first_isp_hop`   | the router's WAN path to the ISP edge       |
  | `ping_8_8_8_8`            | `tc_google_dns`      | the full path to the public internet        |

  One `timeseries_range` on `sprinter_loss` over the window returns all three at
  once (they differ only by `probe_id`). Read them **as a pattern** — no single
  series localizes anything, the *combination* does:

  | gateway | ISP hop | 8.8.8.8 | Verdict                                                     |
  |---------|---------|---------|-------------------------------------------------------------|
  | loss    | loss    | loss    | LAN-side: the agent's own link, a switch, or the router itself |
  | clean   | loss    | loss    | **at or beyond the router's WAN port** — open the WAN bracket below |
  | clean   | clean   | loss    | beyond the ISP edge — upstream routing/peering; corroborate with `ioda` |
  | clean   | clean   | clean   | not a connectivity fault — look at DNS, throughput, or the client |

  **Loss on the ISP hop and on 8.8.8.8 starting and ending together is one event,
  not two.** Traceroute going to `* * *` in the same window is the *same* event
  seen from the routing plane — do not report it as a separate "route change"
  finding, and do not let `RouteChangedHops` talk you into one.

  **The ladder brackets the fault; it does not finish the job.** The
  `clean / loss / loss` row spans **two** hops — router→modem and modem→ISP —
  because the modem/ONT is invisible at layer 3 and never appears in a
  traceroute, and on a double-NAT site (consumer router behind an ISP gateway)
  that gap is where the fault usually is. Resolve inside the bracket, in this
  order:

  1. **The modem/ONT itself — the only reads inside the gap.** Find the WAN
     device first: on a bridge-mode site it is a *separate box* from the one
     `network_tech_stack` calls the gateway, so sweep
     `find_device(device_class=...)` over `cable_modem`, `cable_gateway`,
     `fiber_ont`, `fiber_gateway`, `cellular_gateway`, `modem` (usually at
     `192.168.100.1`). Then two signals, **in this order**:
     - **Its scripted-probe samples.** A WAN telemetry probe reaches the modem
       over HTTP across the router↔modem link, so a **fresh sample during the
       window proves that hop was up** — free of the ICMP confound below, and on
       some modems it is the only thing that answers at all. Key on the sample
       **timestamp**, not the value. One-way inference only: samples present ⇒
       path up; samples absent ⇒ *unknown*, since a failing rule, expired
       credentials and an offline agent look identical. These rules emit
       **no** gauge at all when the collect fails, only a classified error, so a
       sample arriving really does mean the device was read — and a
       `docsis_connectivity_operational_up = 0` is the modem's own answer about
       its CMTS link, not our failure to reach it.
     - **`timeseries_range` on `sprinter_ping_loss_ratio{device_id="<wan
       device>"}`.** Clean here with loss on the ISP hop puts the fault
       **beyond** the modem; loss here too puts it on the **router↔modem link or
       the modem itself**. **Calibrate this series before reading it** — it comes
       from the fleet multi-ping probe, so it quantizes to 0.2, many modems
       rate-limit ICMP into a permanent non-zero baseline, and a flat `1.0`
       usually means "never answers" rather than "down". `get_reference_doc`,
       `name: wan-metrics-reference`, *Using the WAN device's ping as a path
       anchor* has the tri-state procedure. Run it against a window that starts
       **before** the complaint, or do not use the series at all.
  2. **The router's WAN port.** `timeseries_range` on
     `sprinter_interface_operational_up` for that interface. `oper_up = 0` while
     `admin_up = 1` is **carrier loss on the router↔modem cable** — the break is
     inside the building, on a physical link, and nothing upstream can cause it.
  3. **The routing plane —** `traceroute_history`. Read the STORED traces across
     the window (`probe_id`/`target` + start/end), or resolve one exactly by
     passing an `event_evidence` payload's `TracerouteSnapshotId` as
     `snapshot_id`. **Read the raw `path_hops` lines, not the parsed counts.** An
     **`!N`** (ICMP network-unreachable) coming from the *local router's own IP*
     means the router had no default route — the same conclusion as (2), reached
     independently. A long run of `* * *` before it is ICMP error rate limiting,
     not extra hops failing. Do NOT use `network_traceroute` here: that runs a
     fresh live trace and says nothing about a window that has passed.
  4. **The modem's own WAN telemetry values** (`get_reference_doc`,
     `name: wan-metrics-reference`). If link state held steady with no alarms
     across the whole window — a fiber ONT still showing the PON link up with
     unchanged optical RX power, a DOCSIS modem with unchanged lock — then the
     **modem→ISP segment did not drop**, and the fault was on the router side of
     the modem.

  **Never report "the ISP link failed" from the ladder alone.** `clean / loss /
  loss` is equally consistent with a one-minute Ethernet flap on the cable
  between the router and the modem, and on a site with a customer-owned router
  that is the more common cause. Name which of the two hops the evidence in
  1–4 actually indicts, or say that you could not separate them.
- **The egress path (when a device is named) — the layer map itself.** Call
  `topology_path(network_id=<net>, from_device=<device_id>, to="internet")`. It
  returns the device's actual route out: device → serving AP / switch → gateway
  → ISP, with the `public_ip`, `first_isp_hop`, and `asn` on the egress edge.
  **This is the localization skeleton** — every layer in the table below is a hop
  on it, and each hop is named with a `device_id` you can hand straight to
  `network_ping` / `device_presence_history` / `show_device`. Localizing "the
  internet is slow" means deciding *which hop on this path* is at fault; getting
  the path first makes the rest of Step 2 a targeted check rather than a sweep.

  Read the hops for structure, too: a Wi-Fi first hop (`ATTACHED_TO ... ssid=`)
  says the complaint is behind a wireless link; a `MESH_BACKHAUL` hop says the
  traffic crosses a wireless backhaul (a common hidden bottleneck — the client's
  own link can be strong while the extender's backhaul starves it). If it
  reports `found: false`, read the `status`/`reason` — an `UNPLACED` device is
  itself a finding, not an error.

  `as_of` (RFC3339) makes this a **diff**: the path now vs the path when things
  worked. An extender whose backhaul flipped wired→wireless, or a device that
  moved to a different switch port, shows up here and nowhere else.

  **When the complaint is that a device *went offline*, `as_of` is not a
  refinement — it is required.** The graph closes a device's edges once it stops
  being observed, so an offline device returns `found: false` at "now" no matter
  how it was wired. Pin the onset `T` first (`device_presence_history`), then ask
  as of `T − 15 min`. Treating the "now" answer as the finding reports the
  outage's *consequence* as if it were its cause.
- **"This device keeps going offline / drops intermittently" (a named device):**
  call `device_presence_history` with the device's `device_id` over the complaint
  window. It returns the device's **state-transition timeline** (online ⇄ sleep ⇄
  offline, plus Wi-Fi roam/disassoc events), which localizes an intermittency
  complaint in one cheap read — a single `network_ping` only sees up-or-down
  *now*. Read it as: `online <-> offline` flapping = a real drop pattern;
  `online -> sleep -> online` = a healthy power-saving client (`sleep` =
  associated-but-dozing, not a fault); clustered `roamed`/`disassociated` events
  = a Wi-Fi roaming problem (→ `diagnose-wifi-roaming`). Omit `start`/`stop` for
  the last 24h, or pass an explicit window for "what happened last night".
- **Upstream / ISP:** `isp_info` for the ASN, then `ioda` over the window for a
  known ISP/regional outage. Rules out "it's not us."
- **A packet-loss episode? Check for a `lan_wide_connectivity_loss` issue first —
  it IS the LAN-vs-WAN verdict, already computed.** Loss on the internet anchors
  (`ping_8_8_8_8` / the first hop) proves the agent lost the internet, not *where*
  the fault was. The agent reaches the internet through the LAN, so an internal
  fault — a broadcast/multicast storm, a switching loop, a gateway melting down —
  shows up as internet loss from the agent's vantage while the cause is inside the
  building. Focus Layer pings **every device on the network** from the agent (the
  multi-ping fleet probe, `pt_multi_ping`) and the insights pipeline correlates it:
  when a large fraction of the fleet is lossy in the same window it fires a
  **`lan_wide_connectivity_loss`** issue (`probeType = pt_multi_ping`). If your
  `network_issues` fetch already returned one for the loss window, the answer is
  **internal fault, not upstream** — use it and skip the manual query. Only when no
  such issue fired, fall back to the manual correlation: over the loss window,
  `timeseries_range` on `sprinter_ping_loss_ratio` for the LAN infra (gateway,
  switches `device_class="network_switch"`, the Wi-Fi controller, a couple of
  always-on internal hosts) and compare to the anchors. **Loss on WAN anchors only,
  LAN clean → a real upstream event. Loss on LAN infra too (many internal targets
  at once, elevated RTT/jitter) → the fault is inside the network** and the internet
  loss is a symptom, not the cause — do not report it as "a one-off WAN event."
- **The WAN link itself — the modem's own telemetry.** `ioda` tells you whether
  the *ISP* is broken for everyone; this tells you whether **this** connection is
  degraded, which is a far more common cause of "the internet is slow" and is
  invisible to every other check here. It is a cheap, decisive read — do it on any
  complaint that could be upstream (slow, buffering, drops, "works then stops").

  You already have the **Connection technology** (`cable` / `fiber` / `cellular`)
  and the gateway's `device_id` from `network_tech_stack`. Fetch
  `get_reference_doc(name: "wan-metrics-reference")`, use **only** that
  technology's section, and `timeseries_instant` its metrics against the WAN
  device's `device_id`. Grade each against the band the doc gives.

  **If the technology reads `unknown`, do not give up — look for the modem or
  dish.** On the classic bridge-mode setup (a standalone cable modem behind a
  consumer router) the "gateway" is the *router*, so the technology resolves to
  `unknown` even though the modem is streaming DOCSIS from its own IP (usually
  `192.168.100.1`). **Starlink lands here too**: its dish is a generic `modem`, so
  the tech reads `unknown` while the dish streams satellite telemetry from
  `192.168.100.1` and the Starlink *router* (`192.168.1.1`) is what got named.
  Sweep `find_device(device_class=...)` over `cable_modem`, `cable_gateway`,
  `fiber_ont`, `fiber_gateway`, `cellular_gateway`, and `modem` (a `modem`-class
  hit with `vendor == "Starlink"` is the dish → satellite section); a hit gives
  you the technology **and** the `device_id` that actually emits. Only if that
  sweep comes back empty is the WAN genuinely unreadable.

  Report the **cause**, not the number, and mind the **two-sided** metrics: a
  DOCSIS downstream at `+10 dBmV` is an *overdriven input* (remove an amplifier),
  not "strong signal" — the opposite fix from a weak one. The doc names what each
  tail means; use its words.

  How to read the result:
  - Any WAN metric `poor` → **stop localizing inward.** The fault is on the cable
    plant / fiber / cellular link. Report it with the cause and, for cable, that
    it is an ISP/line issue the user should raise with their provider — then
    **hand off to `Skill(troubleshoot-cable-modem)`** for the per-carrier reading
    (which carrier, artifact vs plant fault, what the customer felt, the quotable
    ISP statement). The band check here is yes/no; that skill is the diagnosis,
    and it knows the collection artifacts that make a `poor` read false.
  - All WAN metrics `good` → the WAN link is healthy. This is a **positive
    finding**: it eliminates the entire upstream layer, so the problem is inside
    the building (Wi-Fi, LAN, DNS, or the device). Say so — it is what lets the
    rest of the triage be conclusive rather than a shrug.
  - Metrics missing → follow the **When metrics are missing** table in the
    reference doc. Do not ask a fiber, cellular, or Starlink customer for
    credentials; those rules read unauthenticated endpoints (the Starlink dish
    speaks a credential-free local gRPC API), so missing metrics never mean "no
    credentials" there.
- **DHCP server health (when the complaint is "can't connect" / "no IP" /
  "connects then drops"):** a client that cannot get or renew a lease looks
  exactly like a dead network, but the WAN and Wi-Fi links can be perfectly
  healthy. Focus Layer runs an active DORA probe, so this is a direct read, not an
  inference. Get the DHCP probe's `probe_id` from `show_probes(network_id=<net>)`
  (the `pt_dhcp` probe), fetch `get_reference_doc(name: "dhcp-metrics-reference")`,
  and grade its metrics with `timeseries_instant` / `timeseries_range`. The three
  reads that decide it: `rate(sprinter_dhcp_timeouts_total[15m])` (server not
  answering), `rate(sprinter_dhcp_acks_seen_total[15m])` vs
  `rate(sprinter_dhcp_offers_seen_total[15m])` (offers-without-acks = pool
  exhaustion, a distinct fault from "down"), and `sprinter_dhcp_num_responses`
  (`>= 2` → a rogue/second server, which causes exactly the intermittent
  wrong-gateway breakage users describe as "sometimes it works").

  How to read the result:
  - Timeouts climbing, acks flat → **DHCP server down / unreachable.** Report it;
    the fault is the server, not the link.
  - Offers flowing but acks flat (often with NAKs) → **pool exhaustion / lease
    refusal.** Different remediation (widen the scope / free addresses), so do not
    collapse it into "down".
  - `num_responses >= 2` → **a second DHCP server is answering.** Cross-check with
    the `dhcp_new_server_seen` / `dhcp_multiple_servers_seen` events in
    `network_issues`; this is a prime cause of intermittent connectivity.
  - All healthy (acks flowing, DORA fast, one responder) → **DHCP is fine**, a
    positive finding that eliminates the layer. If metrics are missing, follow the
    reference doc's **When metrics are missing** table — the usual cause is the
    wrong label (`device_id` instead of `probe_id`) or no DHCP probe on the
    network, not a dead server.
- **Gateway / network basics:** `network_info` (gateway, addressing),
  `network_ping` the gateway and a public anchor (e.g. 1.1.1.1) for loss/RTT.
- **DNS:** `network_dns_lookup` a couple of names — resolution failure or slow
  resolver is a classic "internet is broken" that is neither WAN nor Wi-Fi.
- **Throughput / jitter** (only if "slow"/"buffering"): `network_speed_test`,
  `network_jitter_test`.
- **Wi-Fi link (if a device is named or the complaint smells wireless):**
  `show_device` on the client and read `evidence.wifiSnapshots[].association`
  — `signalDbm` (RSSI; some sources only give `signalBars` 0–5), `band`,
  `txRetriesRatio` (0..1; -1 = not reported), and the serving AP
  (`anchorDeviceId`). Also read `wifiSnapshots[].source` — it is
  `wifi_<platform>` and names the platform: **`wifi_ubiquiti`** (UniFi) is
  **rich** per client (signal, SNR, retries), while **`wifi_att-bgw`**
  (BGW320 gateway) is **sparse** (no per-client SNR/retries — only signal +
  deauth/disassoc). Missing SNR/retries on a BGW320 is **expected, not a
  fault** — do not read a sparse snapshot as a problem. `wired: true` means
  Ethernet behind an AP — rule Wi-Fi out. This is Focus Layer's own evidence,
  polled from the controller about every minute (`observedAt` 1–2 min stale is
  fine); this snapshot read is a **routing signal only** — the authoritative
  live-VM read and catalog-band grading happen in `diagnose-wifi-basic`, so no
  controller API call or band-grading is needed here. (Metric names and health
  bands: fetch via `get_reference_doc`, `name: wifi-metrics-reference`.) If the
  overview lacks `wifiSnapshots` (older server), retry with
  `sections="discovery"`.

## Step 3 — Localize to a layer

From the above, decide where the problem lives. When a device was named, each
layer here maps to a **hop on its `topology_path`** — walk the path outward and
ask which hop the evidence indicts:

| Signal                                                    | Layer                |
|-----------------------------------------------------------|----------------------|
| **Anchors: gw loss + ISP-hop loss + 8.8.8.8 loss**        | **LAN / the router** |
| **Anchors: gw clean, ISP-hop loss, 8.8.8.8 loss**         | **Router WAN port → modem → ISP** (bracket; resolve per Step 2) |
| **Anchors: gw clean, ISP-hop clean, 8.8.8.8 loss**        | **Beyond the ISP edge** |
| `ioda`/`isp_info` shows upstream outage                   | ISP (everyone)       |
| **A WAN metric grades `poor`** (see below)                | **The WAN link**     |
| **All WAN metrics `good`** — rules the whole layer OUT    | **Look inward**      |
| Gateway ping bad, WAN side bad, LAN side fine             | Gateway / WAN        |
| DNS lookups fail/slow, ping-by-IP fine                    | DNS                  |
| Speed test low but loss/latency fine                      | Throughput / WAN     |
| Only one device bad; others on same AP / switch port fine | That device          |
| Wi-Fi client(s): weak signal / high retries / jitter      | Wi-Fi                |
| A `MESH_BACKHAUL` hop on the path (wireless backhaul)     | Wi-Fi (mesh)         |
| An infra hop on the path went `offline` at onset          | That infra hop       |

The WAN row is the one that most often ends a triage early, and it cuts **both**
ways. A `poor` DOCSIS upstream or a dark fiber explains "the internet is slow"
completely — stop looking inward and report the line fault. Equally, an
all-`good` WAN link *eliminates the entire upstream layer*, which is what turns
the rest of the triage from guesswork into a narrowing search. Distinguish it
from the `ioda` row: `ioda` says the **ISP** is down for everybody; the WAN
metrics say **this specific line** is degraded, which is far more common and
which no other check here can see.

## Step 4 — Dispatch

- **Wi-Fi, single client or "slow/choppy on wireless":** invoke
  `Skill(diagnose-wifi-basic)` with the device — the general Wi-Fi link
  health check.
- **Wi-Fi, "stuck on far AP" / won't roam / mesh involved:** invoke
  `Skill(diagnose-wifi-roaming)`. The association evidence helps pick this
  route: weak `signalDbm` while `anchorDeviceId` names a distant AP is the
  roaming-shaped signature.
- **Wi-Fi, "which/how many devices have weak signal / high retries" (the whole
  network, not one client):** invoke `Skill(diagnose-wifi-basic)` — it answers
  fleet questions with a **single** network-scoped VM query that even joins in
  device names (`... * on(device_id) group_left(device_name) sprinter_deviceInfo`),
  not a per-device `show_device` loop. Do not scrape `wifiSnapshots` signal
  device-by-device here; the snapshot lags and that is the slow, wrong path.
- **Unknown single device acting up:** `Skill(troubleshoot-device)` first to
  learn what it is and check its identity/connectivity (it asks "what is this
  device?" and hands off to the Wi-Fi specialists if the device turns out to be
  wireless).
- **ISP/WAN, gateway, DNS, throughput:** no specialist skill exists yet —
  report the localized finding and the evidence directly. These are good
  candidates for future triage→specialist funnels; note that to the user.
- **Nothing localized — the failing path was unobservable (coverage gate, Step
  1.5) or every broad check came back clean:** do NOT invent a layer to blame.
  Close per Step 1.5-D — name the blind spot, list the layers you cleared, and
  hand the user the client-side capture steps. "I couldn't reproduce or localize
  it from here, and here's exactly how to catch it next time" is a complete,
  honest triage outcome.

## Step 5 — "When did this start?" before you hand off

Localization answers *where* the problem lives; the specialist will answer *why*.
But one cheap correlation pays off before (or alongside) the hand-off: **pin when
the trouble started and sweep what else happened at that moment.** A complaint
that began at 21:14 is far more diagnosable once you see the gateway's DHCP
config changed at 21:13, or the serving AP went offline at 21:12 — and that
co-occurrence often *is* the localization.

Read-only, on tools already listed above. Full recipe: fetch via
`get_reference_doc`, `name: when-did-this-start`. Short form:

1. **Pin the onset `T`.** If a device is named, `device_presence_history` over
   the complaint window — the first `online -> offline` flap or clustered
   `roamed`/`disassociated` is the onset (an `online -> sleep -> online` cycle is
   healthy power-save, not an onset). For a network-wide complaint, the earliest
   significant `network_issues` `startTime` is the onset — a `mean_shift`
   issue's start *is* a computed inflection point, so prefer it over eyeballing
   `timeseries_range` for any predefined-probe metric (the reference doc has the
   raw-metric fallback recipe for everything else).
2. **Sweep a tight window around `T`** (`T ± 15 min`) with `network_issues`,
   reading events by `probeType`: **`pt_dhcp`** (a config change / rogue server
   on the gateway at ≈ `T` explains "lost connection" / "weird IP" for the whole
   LAN), **`pt_traceroute`** (path change upstream), `pt_ping`/`pt_dns` on the
   gateway (infra-wide). DHCP and traceroute events have **no** dedicated history
   tool — they come through `network_issues`. Check `isp_info`/`ioda` for an ISP
   outage that began at `T`. Run `device_presence_history` on **each infra hop
   the `topology_path` named** (serving AP, switch, gateway) — one of them going
   `offline` at ≈ `T` localizes the complaint outright.
   **And diff the path across `T`**: `topology_path(..., as_of=<before T>)` vs
   now. A route that *changed* at onset — a device that moved switch ports, an
   extender whose backhaul flipped wired→wireless — is a cause no probe sweep
   would surface.
3. **Let the correlation sharpen the dispatch.** A DHCP/path/ISP event at ≈ `T`
   re-points triage away from "one device" toward the infra layer — say so and
   route accordingly. A clean sweep around a single device's onset reinforces
   "this device" as the layer.

This is localization-grade (cheap, broad), not the specialist's deep timeline —
note onset + any co-occurring infra event in the hand-off so the specialist
starts from it.

### Did something REBOOT at `T`?

The highest-signal single fact in this whole step, and the cheapest. A switch
that rebooted at 03:12 explains every neighbour that lost its uplink at 03:12 —
the reboot *is* the answer, and nothing else in the sweep above will name it.

Two derived series carry it. Neither is collected: both are computed in
VictoriaMetrics from every uptime metric at once (sprinter#484), which is why
one query covers the Starlink dish, the fibre link, the cellular gateway, the
Wi-Fi mesh nodes and every SNMP device.

| Series | What it is | Use it for |
|---|---|---|
| `sprinter_uptime_resets` | count of uptime drops in the last 30 min: `0` healthy, `1` for 30 min after a restart | **detection** — this is what raises the issue |
| `sprinter_boot_epoch_seconds` | UNIX epoch the thing came up | **the "when"** — read it directly, accurate to about a second |

```promql
# What restarted on this network in the complaint window?
max_over_time(sprinter_uptime_resets{network_id="<id>"}[24h]) > 0

# When did that thing last come up? (epoch seconds -> a timestamp)
sprinter_boot_epoch_seconds{device_id="<id>"}
```

The `uptime_of` label names WHAT restarted — `system`, `optical_link`,
`cellular_gateway`, `starlink`, `wifi_node`, `interface`. A device can report
more than one, and they mean genuinely different things: `optical_link` on a
fibre gateway is a link flap, not a reboot of the box.

**How confident to sound.** This is corroboration for a problem the user has
already reported, not an alert. The sentence it exists to enable is *"during the
window you reported, this router's uptime reset — it most likely rebooted."*
Say that plainly. Do not stack hedges onto it: the caveats below change the
wording in specific, named situations, and outside those a reset during the
reported window is good evidence and should be presented as such.

**Four caveats, each of which changes what you should say — and only when it
applies:**

- **A steady flapper is INVISIBLE here.** If a device restarts at a constant
  rate for the whole window, the reset count becomes its own baseline and no
  finding is raised. This method is good at "it rebooted once", weak at "it
  reboots constantly". Absence of a finding is not evidence the device is
  stable — check `sprinter_boot_epoch_seconds` directly, which shows a recent
  boot regardless.
- **The finding reads about 30 minutes LONGER than the event.** The plateau
  lasts exactly the recording rule's window. Quote the boot epoch as the time,
  never the finding's duration.
- **Severity is 1.00 for all of these**, by construction. It does not rank them
  — do not read a high severity as a worse reboot.
- **Read `uptime_source` before calling it a reboot.** Since sprinter#580 the
  metric carries the OID that produced it, so you do not have to guess:

  | `uptime_source`  | What a reset means                                      |
  |------------------|---------------------------------------------------------|
  | `hrSystemUptime` | The HOST rebooted. Say so directly.                     |
  | `snmpEngineTime` | The SNMP engine restarted — usually the box, but on a Linux host it can be `snmpd` alone. |
  | `sysUpTime`      | Same caveat, plus this counter wraps (below).            |

  A measured example of why this matters: a host up 344 days reported 7.65 days
  on both agent-scoped OIDs. On the two agent-scoped sources, prefer "the SNMP
  agent restarted, likely the device" over asserting a reboot outright.

**The 497-day wrap, in proportion.** `sysUpTime` and `hrSystemUptime` are
32-bit TimeTicks and roll over every 497.1 days, and at that instant the drop
looks like a reboot. This does **not** mean the device is invisible — it is
collected and charted normally; only the interpretation of that one drop is
ambiguous. It is one instant per 497 days per device, so for it to mislead you
it must land inside the very window being investigated: roughly 1 in 12,000 for
a one-hour window. Treat it as a footnote, not a reason to hedge. If you do want
to rule it out, a wrap advances `sprinter_boot_epoch_seconds` by **exactly**
497.1027 days where a real reboot advances it by an arbitrary amount.

If `sprinter_uptime_resets` returns no series at all for a network whose devices
DO report uptime, that is a broken recording rule rather than a quiet network —
the `UptimeResetsRuleDead` alert covers it. Say "restart detection is
unavailable", not "nothing restarted".

And do not read a plausible series COUNT as proof the rule is healthy. When these
rules broke once (sprinter#484 §12.1), `count(sprinter_uptime_resets)` kept
returning the correct 9 for as long as the last good samples stayed inside the
lookback window, while every evaluation was failing. If you need to know the rule
is alive rather than merely remembered, check freshness:
`max(time() - timestamp(sprinter_uptime_resets))` should be under a couple of
minutes.

## Honesty

- Report what you checked and what you did NOT (e.g. "did not run a speed test
  because the complaint was DNS-shaped"). Don't imply full coverage.
- "It's the ISP" requires positive evidence (`ioda`/`isp_info`), not just
  "everything looks fine locally."
- If the broad checks come back clean, say so — "no current network-level issue
  detected in the window; if it was intermittent, give me a tighter time range"
  — rather than inventing a cause.
- **Vantage before value.** For every measurement you cite, be sure it was taken
  from a point that can *see* the failing path. A wired-agent probe is silent on a
  Wi-Fi client's link; a stuck value identical across runs is an artifact. If your
  only numbers come from the wrong vantage, you have no evidence — say so.
- **A blind spot is a finding, not a failure.** "Focus Layer cannot observe this path
  from here (why), but these layers are clean, and here's how to catch it" is a
  legitimate and *complete* answer. Prefer it over any root cause you cannot
  measure. Never publish a ranked list of indistinguishable causes as if it were a
  diagnosis.
