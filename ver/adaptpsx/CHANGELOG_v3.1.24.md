# AdaptPSX — Changelog (by feature area) — v3.1.24

**AdaptPSX** connects to the Aerowinx **Precision Simulator X (PSX)** Boeing 747-400
simulator over TCP/IP and bridges the simulator's state to external hardware and
utilities — Aerosoft/CPFlight MCP panels, Engravity CDUs, GPS, printers and data
recording.

This document covers **the last 18 months (March 2025 – September 2026)**, up to and
including **v3.1.24**. It is grouped **by feature area** rather than by date — each bullet
notes when the change landed — so it doubles as a "what to test" guide. Almost all of the
work below is the **v3.0 / v3.1 release cycle** (May–September 2026); the sections are
ordered with the biggest, most test-worthy changes first. In-development features not yet
exposed by default are omitted.

---

## Hoppie Datalink — ACARS, CPDLC & ADS-C *(new)*

The largest addition in this release: a full datalink suite over the Hoppie network —
airline text messaging (ACARS/AOC), controller–pilot datalink (CPDLC) and automatic
position surveillance (ADS-C) — all reusing the existing PSX connection and CDU framework.
This is where most testing effort should go: message send/receive, CPDLC logon and
logon-rejection, ADS-C contract handling, and that the flight number stays in step with PSX.

- **Hoppie ACARS / CPDLC integration** *(Aug 2026, v3.1)* — added airline (AOC) text messaging
  and controller–pilot datalink over the Hoppie network. ACARS messages appear on a new
  `<HACARS` CDU page set; CPDLC bridges PSX's own datalink so clearances and logon are handled
  natively, and a rejected CPDLC logon is now reported on the EICAS advisory line and cleanly
  returned to "logged off". The flight number is kept in sync with PSX so ACARS and CPDLC always
  use the same callsign, and CDU rendering became colour-aware along the way. Turned on with
  `enableHoppie` and configured (Hoppie key, ACARS/CPDLC toggles) on a new Hoppie ACARS tab.
- **ACARS messaging pages — TELEX, OOOI and weather requests** *(Aug 2026)* — the `<HACARS` page
  set gained: a **TELEX** free-text message with an editable address directory (its own editor
  window, saved alongside the app), with sent and received telexes mirrored onto PSX's native
  ACARS message store so they raise the **ACARS MESSAGE** memo, chime and CDU MSG light; an
  **OOOI STATUS** page (out/off/on/in times) with a two-column print, and a received-messages
  index that sorts by flight phase (pre-flight / in-flight / post-flight); and a **weather
  REQUEST** page offering **METAR**, **TAF** and **ATIS** from one shared set of airport fields,
  each running `SENDING` → `SENT`, with ATIS routed to the network chosen in a new **Hoppie
  network** dropdown (VATSIM / IVAO / PilotEdge).
- **ADS-C automatic position surveillance** *(Aug 2026, v3.1.16)* — AdaptPSX now answers Hoppie
  **ADS-C** surveillance contracts, downlinking position reports (flight, time, lat/long,
  altitude, heading) built from the live PSX position feed at the requested interval. It accepts
  periodic, event and cancel requests from any ground station. Availability follows both the new
  **ADS-C** checkbox (`enableADSC`, on by default) and the aircraft's ADS arm state read from
  PSX's native ATC LOGON/STATUS page, and turning ADS off drops every active contract.
- **ADS-C full contract set, emergency mode and richer reports** *(Aug 2026, v3.1.20)* — ADS-C now
  answers the complete FANS contract set, not just periodic reporting: single on-demand reports,
  configurable periodic reporting (minimum 64 s), and the four standard **event contracts** —
  waypoint change, level-range deviation, vertical-rate and lateral deviation — evaluated once a
  second. A CPDLC emergency downlink (**MAYDAY** / **PAN**) automatically arms ADS-C emergency
  reporting and a cancel clears it; while armed, reports are tagged `EMERGENCY` and sped up. An
  optional **rich reports** mode adds ground/air data (true track, ground speed, vertical speed)
  and the next two waypoints, while keeping the basic report prefix so simpler parsers still work.
- **ADS-C route awareness** *(Aug 2026, v3.1.21)* — new active-route reading derives the **next and
  next-plus-one waypoints** and the **cross-track distance** from PSX's live FMC route, feeding the
  waypoint-change and lateral-deviation event reports. It reads the *active* route so "next
  waypoint" is the true upcoming fix rather than one already passed.
- **ACARS REFUELING and FPL DATA reports** *(Aug 2026, v3.1.19)* — the REFUELING page now follows the
  aircraft's selected **weight unit** (KGS / LBS × 1000, from PSX pin programming), shows the uplift
  discrepancy (**DIFF = ON BOARD − (QTY BEFORE + SUPPLIED)**), snapshots the on-board fuel when
  refuelling starts, and defaults the **FUEL TYPE** from PSX (JETA / JETA1 / TS1 / JETB) while
  staying crew-editable. A new **FPL DATA** report downlinks block/taxi fuel and take-off weight to
  the DISPATCH address, warning **NO DISP ADDR** (with the CDU MSG light) if no dispatch address is
  set. The REQUEST page was rearranged to free a fifth airport slot.
- **Configurable ACARS name on the message index** *(Sep 2026, v3.1.22)* — the received-messages
  index title now uses the configurable ACARS name that already drives the CDU MENU prompt, instead
  of a hard-coded `HACARS`.

## CDU display & behaviour

Shared improvements to how the add-on pages render on the physical CDUs.

- **Per-CDU menu placement** *(Aug 2026, v3.1.16)* — the ACARS entry on the CDU MENU can now be shown
  on the left, centre and/or right CDU independently, from the config tab. Default is **Centre only**.
- **Cleaner page changes and entry editing** *(Aug 2026, v3.1.16)* — switching pages now writes the new
  content while the screen is briefly blanked, so the previous page no longer flashes through;
  **DELETE** followed by a line-select key clears just that entry field, and **CLR** now cancels a
  pending DELETE.

## Stability & Reliability

A broad thread-safety, performance and resource-leak sweep, prompted by the same class of defects
found in the sister SwitchFS app plus an independent scan of AdaptPSX. These fixes target freezes,
runaway CPU and console spam under heavy PSX traffic and on disconnect — worth stress-testing with
busy sessions and repeated connect/disconnect cycles.

- **Fixed GUI freezes, high CPU and resource leaks under heavy PSX traffic** *(Aug 2026)* — the
  debug console no longer freezes the interface when a busy PSX stream floods it (its output is now
  batched instead of redrawn character-by-character); many on-screen updates (status labels, printer
  view, chat, connect state, updater progress) were moved onto the UI thread to stop intermittent
  glitches and crashes; and the app no longer pegs a CPU core or leaks the network connection when
  PSX drops without a clean exit. Also fixed in the same pass: a chat crash when users timed out,
  truncated magnetic variation in the GPS output, an ETA field that never matched, and a
  settings-file read that could silently revert later settings to their defaults if one value was bad.
- **Fixed a console/CDU flood on PSX disconnect** *(Aug 2026)* — closing the PSX connection could
  trigger an endless loop that spammed the debug console with "PSX CONNECTION CLOSED" and CDU-reset
  lines; the disconnect cleanup now runs exactly once.

## Central Logging (Aggregator)

New **opt-in** central error/event logging to the Simulator Solutions Aggregator, to help diagnose
issues in the field. It is **disabled by default** (staged rollout). Testers should verify the
consent prompt, the enable/disable toggle, and that nothing is sent while it is off.

- **Central error/event logging (opt-in)** *(Aug 2026)* — AdaptPSX can now forward errors and key
  events (app start/stop, crashes) to the Simulator Solutions central logging service. It is
  best-effort and never blocks the app, is disabled by default, and is gated behind a GDPR
  privacy-consent dialog the first time it is turned on. Crashes on the UI thread are captured and
  reported automatically.
- **Logging on/off and server controls in the app** *(Aug 2026)* — the logging tab gained an
  **Enable** checkbox and a server-URL field. Ticking Enable shows the privacy notice, records
  consent and starts logging immediately (with a visible confirmation event); unticking stops it.
  Both settings are saved in `Adapt.ini`.

## Remote control

- **Remote printer toggle over the PSX link** *(Aug 2026, v3.1.15)* — any PSX client can now toggle
  AdaptPSX's master **Enable Printer Output** by sending an `addon=adaptpsx;printer;;;enable=1`
  (or `enable=0`) command. The change is immediate — no stop/start — and the GUI checkbox and status
  label stay in sync. This is built on a small command dispatcher that further remote commands can
  extend.

## Monitor Tab & Diagnostics

Small but handy additions to the Monitor tab so an operator can see at a glance what each subsystem
is doing and which firmware is connected.

- **Live activity timestamps** *(Aug 2026)* — new "last activity" times for ACARS and CPDLC, stamped
  whenever a message is sent or received (matching the existing serial-connect time). They clear when
  PSX disconnects.
- **Aerosoft MCP firmware version shown** *(Aug 2026)* — the Monitor tab now shows the connected
  Aerosoft MCP panel's firmware version on its own row.

## Auto-update & Security

The self-update path was completed and hardened: checking for new builds, verifying them before
trusting them, comparing version numbers correctly, and reading the right feed.

- **Auto-update with certificate verification** *(May 2026)* — the app can check for and download new
  builds automatically, and a downloaded build is verified against a certificate before it is
  trusted, so an update cannot be tampered with in transit.
- **Correct version comparison for the update check** *(Aug 2026, v3.1)* — the update check now
  compares versions part-by-part (proper `major.minor.patch` semantics) instead of treating the
  version as a single decimal, so three-part version numbers no longer confuse the "is there a newer
  build?" test.
- **Reads the standard download feed key** *(Aug 2026, v3.1.14)* — the update check now reads `dlurl=`
  from the version feed, the same key the shared Updater tool and the sibling apps write, so a
  tool-generated feed self-installs instead of falling back to the website dialog (older `updateurl=`
  feeds still work).
- **Corrected version feed source** *(Sep 2026, v3.1.24)* — the update check now reads its version feed
  from the GitHub Pages feed the Updater publishes. The retired `simulatorsolutions.com.au` feed
  served a bogus version with no download link, which caused a false "new version available" popup on
  every launch; existing settings are auto-migrated off the old address, while any custom feed you set
  is still honoured.
- **Manual "Check For Updates" button** *(Sep 2026, v3.1.23)* — the **About** tab gained a working
  on-demand update check that confirms "You are running the latest version" when nothing is newer, and
  shows a clear dialog on a bad feed or network error, rather than failing silently.

## Build, Packaging & Standards

Housekeeping to bring AdaptPSX in line with the shared Simulator Solutions application standard and
its automated deployment. None of this changes how the app behaves for end users.

- **Version renumbered to 3.x, with one source of truth** *(Aug 2026)* — the version was consolidated
  to a single place (the app's own `version` string) and bumped from 1.5 to **3.0**, ending years of
  mixed "Beta 7" / `1.x` / `B2.3` numbering. A commit hook now bumps the patch number automatically.
- **Self-contained deployment bundle** *(Aug 2026)* — a new packaging step bundles the app together
  with its `Adapt.ini` / `Chat.ini` settings into a ready-to-deploy folder for the WorkHub deployment
  pipeline.
- **Rebuilt on Java 11** *(Aug 2026)* — the build moved from Java 10 to Java 11 (needed for the
  standard HTTP client used by central logging). End users now need a Java 11 or newer runtime.
- **Git hygiene** *(Aug 2026)* — stopped tracking runtime files that should never have been in source
  control — the live `Adapt.ini` (which held a secret), chat settings, PSX text output, crash dumps
  and an old build zip — with matching ignore rules.

---

*Period covered: March 2025 – September 2026, up to v3.1.24. AdaptPSX passed through several version
schemes over its life ("Beta 7" in 2016, a `1.x` line through 2023, internal `B2.3` references); in
August 2026 it was consolidated and renumbered to **v3.0**, now the single source of truth the
auto-update feed checks against, with the patch number bumped automatically on each commit. WIP
snapshots, doc/tooling-only commits, merges and file-tracking cleanup are omitted.*
