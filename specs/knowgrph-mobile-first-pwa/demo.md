---
title: "knowgrph Mobile-First PWA — Demonstration Walkthrough"
doc_type: "Demo"
version: "1.0.0"
date: "2026-08-20"
lang: "en-US"
frontmatter_contract: "required"
owner: "solo-founder-ai-orchestrator"
local_rung: "spec-complete"
delivered_rung: "undocumented"
lane: "authoring"
universal_scope: "false"
lifecycle_status: "proposed"
source_requirements: ".kiro/specs/knowgrph-mobile-first-pwa/requirements.md@1.1.0"
source_design: ".kiro/specs/knowgrph-mobile-first-pwa/design.md@1.0.0"
source_tasks: ".kiro/specs/knowgrph-mobile-first-pwa/tasks.md@1.0.0"
governing_guidelines:
  - "huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md"
  - "agentic-canvas-os/docs/START-WORKFLOW.md"
measurement_status: "unrecorded"
---

# Demonstration Walkthrough

## Purpose

This document is the Demo_Guide named in Requirement 10. It scripts a reproducible, timed
demonstration of the shipped feature on a clean physical device, so that first-run installability and
offline behaviour can be judged pass or fail without reading source code.

**Measurement status: `unrecorded`.** Every measured value below is a slot, not a claim. The slots are
filled by a human operator running the script on a physical device. Per Requirement 10.8, a run that
exceeds a ceiling is recorded as a failing run and retained; it is never overwritten by a later
passing run. Per the governing guidelines, no value in this document may be presented as measured
until an operator has recorded it with a date and a device identifier.

## Demonstration surface

| Field | Value |
|---|---|
| Surface under demonstration | Delivery_Surface — `https://airvio.co/knowgrph` |
| Service worker scope | `/knowgrph/` |
| Alternative surface | Authoring_Surface via `npm run dev` — permitted for rehearsal only; a rehearsal run yields `deliveryVerified: false` per Requirement 9.8 and MUST NOT be recorded in the tables below |

## Reset preconditions (Requirement 10.3)

Perform all four before every timed run. A run started from a warm device is void.

1. **Uninstall the home-screen instance.** Long-press the knowgrph icon → Remove/Uninstall. Confirm no
   knowgrph entry remains on any home screen or in the app drawer.
2. **Clear Shell_Cache for the origin.** Browser settings → Privacy → Clear browsing data → select
   Cached images and files, scoped to `airvio.co` where the browser supports per-site scoping.
   Where it does not, clear all cached files.
3. **Clear Local_Store for the origin.** Settings → Site settings → `airvio.co` → Clear & reset, which
   removes the IndexedDB database `kg:knowgrph-storage-engines` including the `graphRecords` table.
4. **Clear Replay_Queue for the origin.** Same reset as step 3 — the Replay_Queue is persisted as the
   `engineOutbox` table in the same IndexedDB database, so step 3 removes it. Confirm by reopening the
   surface and observing an empty replay queue panel.

**Verification that the reset took effect:** open the surface, open developer tools or the remote
inspector, and confirm zero Cache Storage entries and zero IndexedDB databases for the origin before
starting the timer.

## Throttling profile (Requirement 10.5)

The same profile applies to every timed run in this document. A run recorded under any other profile
is void.

| Parameter | Value |
|---|---|
| Profile name | Slow cellular |
| Downlink throughput | 400 kbps |
| Uplink throughput | 400 kbps |
| Round-trip latency | 400 ms added |
| Applied via | Chrome DevTools remote debugging Network conditions → custom profile, or an equivalent on-device network conditioner |
| CPU throttling | none — this measures network time-to-value, not device class |

## Part A — Timed first run (Requirements 10.1, 10.2, 10.4)

Start the timer at the moment the first request for `https://airvio.co/knowgrph` leaves the device.
Stop it when the reopened application has rendered in Standalone_Mode with the network disabled.

Ceilings: **at most 4 steps** and **at most 3 minutes**.

| # | Step | Observable outcome (screen-visible, no source inspection) |
|---|---|---|
| 1 | Open `https://airvio.co/knowgrph` in the mobile browser | The graph view renders. An install affordance appears within about a second on browsers that offer a native prompt; on browsers that do not, the manual Add-to-Home-Screen overlay appears about three seconds after the page becomes interactive. |
| 2 | Install: tap the install affordance and accept, or follow the overlay's numbered Add-to-Home-Screen steps | A knowgrph icon appears on the home screen. The install affordance disappears. |
| 3 | Disable the network: enable airplane mode, or switch the network conditioner to Offline | The browser reports no connectivity. Leave the browser; do not reopen it yet. |
| 4 | Launch knowgrph from the home-screen icon | The application opens with no browser address bar or tabs visible, the graph view renders with previously loaded content, and no install affordance and no Add-to-Home-Screen overlay appear. |

### Recorded measurement — Part A

| Field | Value |
|---|---|
| Measured step count | `unrecorded` (ceiling 4) |
| Measured elapsed time | `unrecorded` (ceiling 3 min 00 s) |
| Verdict | `unrecorded` |
| Device | `unrecorded` |
| Browser and version | `unrecorded` |
| Run date | `unrecorded` |
| Throttling profile applied | Slow cellular, as specified above |

Retain every recorded run as its own row. Do not delete a failing row (Requirement 10.8).

| Run | Date | Device / browser | Steps | Elapsed | Verdict |
|---|---|---|---|---|---|
| — | — | — | — | — | no runs recorded |

## Part B — Offline write and replay (Requirement 10.6)

Perform in order, starting from the installed, offline state left by Part A step 4. Not timed against
the Part A ceilings.

| # | Step | Observable outcome |
|---|---|---|
| B1 | While still offline, edit a graph node and commit the change | The edit appears in the graph view immediately. |
| B2 | Open the replay queue panel | Exactly one pending entry is listed for the edit, with an attempt count of zero and no conflict or parked marker. |
| B3 | Restore the network: disable airplane mode, or set the conditioner back to Slow cellular | The browser reports connectivity. |
| B4 | Watch the replay queue panel | Within a few seconds the entry disappears from the pending list. The panel shows an empty pending list, or lists only entries explicitly flagged as conflicted or needing attention. |
| B5 | Force-quit and relaunch the installed application | The edit from B1 is still present in the graph view. |

### Recorded measurement — Part B

| Field | Value |
|---|---|
| Pending entry appeared at B2 | `unrecorded` |
| Pending entry cleared at B4 | `unrecorded` |
| Edit survived relaunch at B5 | `unrecorded` |
| Flagged entries remaining | `unrecorded` |
| Verdict | `unrecorded` |

## Degraded paths (Requirement 10.9)

| Missing device capability | Affected step | Degraded path | Observable outcome of the degraded path |
|---|---|---|---|
| No deferred install prompt (iOS Safari, Firefox for Android) | A2 | Follow the manual Add-to-Home-Screen overlay instead of a native prompt | The overlay lists between two and six numbered steps for the detected browser, marked generic if the browser was not recognised. The icon still reaches the home screen. |
| No background synchronization | B4 | Bring the application to the foreground after restoring the network | Replay begins on the foreground event rather than in the background. The pending entry still clears. |
| No per-site data clearing | Reset step 2 | Clear all cached files for the browser | The reset verification still shows zero Cache Storage entries for the origin. |
| No airplane mode on the test device | A3 | Use the network conditioner's Offline profile | The browser still reports no connectivity. |
| Standalone display mode unsupported | A4 | Launch from the home-screen icon and accept a browser-chrome launch | Content still renders offline. **Requirement 2.4 is not demonstrable on this device** — record the run as failing that criterion rather than as passing. |

## Evidence mapping (Requirements 10.7, 10.10)

One named invocable check per demonstrated requirement. Every check below runs on the
Authoring_Surface and therefore yields `deliveryVerified: false` per Requirement 9.8, except where
the surface column says otherwise. The demonstration itself is a manual observation on the
Delivery_Surface and does not substitute for these checks.

| Demonstrated by | Requirement | Named invocable check | Surface |
|---|---|---|---|
| A1 shell render, A4 offline render | 1.1, 1.2, 1.4 | `npm -C canvas run test:smoke:pwa-offline-shell:browser` | Authoring |
| A2 install, A4 standalone | 2.1, 2.4 | `npm -C canvas run test:smoke:pwa-installability:browser` | Authoring |
| A2 install affordance behaviour | 2.2, 2.3, 2.5, 2.6, 2.8 | `npm -C canvas run test:ci:unit -- pwa.install.properties` | Authoring |
| A4 offline graph content | 3.2, 3.4 | `npm -C canvas run test:ci:unit -- pwa.localStore.properties` | Authoring |
| B1, B2, B4 | 4.1, 4.4, 4.5 | `npm -C canvas run test:smoke:pwa-replay:browser` | Authoring |
| B4 flagged entries remaining | 4.6, 4.8, 4.9, 4.14 | `npm -C canvas run test:ci:unit -- pwa.replay.outcomeProperties` | Authoring |
| B5 edit survives relaunch | 4.11, 4.13 | `npm -C canvas run test:ci:unit -- pwa.replay.roundTrip` | Authoring |
| A1, A2 manual overlay path | 5.1, 5.2, 5.7 | `npm -C canvas run test:ci:unit -- pwa.overlay.properties` | Authoring |
| Tap and scroll feel throughout | 6.1–6.7 | `npm -C canvas run test:smoke:pwa-touch-audit:browser` | Authoring |
| A1 first-load weight under throttling | 7.1–7.6 | `node ./scripts/check-pwa-payload-budget.mjs` | Authoring |
| No third-party requests during the run | 8.7 | `npm -C canvas run test:smoke:pwa-network-allowlist:browser` | Authoring |
| B1 local-only until replay | 8.6 | `npm -C canvas run test:ci:unit -- pwa.privacy.properties` | Authoring |
| Superseded cache purge across a deploy | 1.3, 9.6 | `npm run production:sw-upgrade:verify` | **Delivery** |
| Lane order and recorded crossings | 9.1 | `npm run release:lifecycle:receipts` | **Mirror → Delivery** |
| This document's own shape | 10.1–10.10 | `node --test scripts/__tests__/pwa-demo-guide-contract.test.mjs` | Authoring |
| Whole-feature gate | all | `npm run pwa:mobile-first:check` | Authoring |

### Claims marked unverified (Requirement 10.10)

These have no invocable check and rest on manual observation only. They MUST NOT be reported as
verified.

| Claim | Manual observation used instead |
|---|---|
| Requirement 8.8 — the feature adds 0.00 USD monthly recurring cost | Operator reviews the Cloudflare and any third-party billing statements for the 30 days following the Delivery crossing and confirms no new line item. Derived from the machine-checked Requirements 8.1 and 8.2, but the billing statement itself is read by a human. |
| Perceived native feel of the installed application | Operator's subjective judgement during Part A. Not a pass/fail criterion and not counted toward any requirement. |
| Time-to-value as experienced by a first-time user unfamiliar with the product | The Part A timing is performed by an operator who already knows the steps, so it is a floor, not a representative user measurement. |

## How to record a run

1. Perform every reset precondition and verify it.
2. Apply the throttling profile exactly as stated.
3. Run Part A with a stopwatch; run Part B immediately afterwards.
4. Append a row to the Part A run table and fill the Part B measurement block. Set
   `measurement_status` in this document's frontmatter to `recorded` once at least one complete run
   exists.
5. Record any exceeded ceiling as a failing verdict and leave the row in place (Requirement 10.8).
6. Note any degraded path taken, naming the missing capability (Requirement 10.9).
