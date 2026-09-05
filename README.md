<div align="center">

# Cross-Platform Streaming & Mobile Ecosystem

**Consumer streaming applications shipped and maintained across six storefronts — Apple TV, iOS, Android, Roku, Fire TV, and Samsung Smart TV — fed by a shared media API, with an adaptive-bitrate origin and real audience telemetry behind it.**

![Roku](https://img.shields.io/badge/Roku_SceneGraph-662D91?style=flat-square&logo=roku&logoColor=white)
![Apple](https://img.shields.io/badge/tvOS_/_iOS-000000?style=flat-square&logo=apple&logoColor=white)
![Android](https://img.shields.io/badge/Android_/_Fire_TV-3DDC84?style=flat-square&logo=android&logoColor=white)
![Samsung](https://img.shields.io/badge/Tizen-1428A0?style=flat-square&logo=samsung&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=flat-square&logo=javascript&logoColor=black)
![Status](https://img.shields.io/badge/apps-live%20in%206%20stores-success?style=flat-square)

</div>

> **Documentation-only repository.** App source is proprietary. Published here: the cross-platform architecture, the media pipeline, the store-review realities, and the diagnostic work behind two production incidents.

---

## 1. The problem

A Spanish-language broadcast network distributes to viewers across the United States, Mexico, and Central and South America — over the air, on the web, and through connected-TV apps. Its audience skews toward television, not browsers, which means the app is the product surface, not a companion.

Five platforms, five languages, five review boards, five sets of certification requirements — and one small team. Building five independent applications would have been unmaintainable. Building one and shimming it would have failed certification.

**The goal:** a single source of truth for content and schedule, thin platform-native clients, and a release process that survives the strictest reviewer on the strictest platform.

---

## 2. Architecture

```
                    ┌────────────────────────────────────┐
                    │  CONTENT / SCHEDULE SOURCE          │
                    │  Editor panel (auth-protected)      │
                    │  → schedule.json, channel metadata  │
                    │  Staff edit showtimes, not code     │
                    └──────────────┬─────────────────────┘
                                   │
                    ┌──────────────▼─────────────────────┐
                    │  MEDIA FEED API                     │
                    │  · channel list + artwork           │
                    │  · EPG (weekday-keyed, channel-     │
                    │    aware, timezone-normalized)      │
                    │  · stream manifests per channel     │
                    │  · versioned, cache-friendly        │
                    └──────────────┬─────────────────────┘
                                   │
        ┌──────────┬───────────┬───┴────────┬────────────┬──────────┐
        ▼          ▼           ▼            ▼            ▼          ▼
   ┌────────┐ ┌────────┐ ┌─────────┐  ┌─────────┐  ┌────────┐ ┌────────┐
   │  Roku  │ │ tvOS   │ │   iOS   │  │ Android │  │Fire TV │ │ Tizen  │
   │Scene-  │ │ Swift  │ │  Swift  │  │ Flutter │  │Android │  │  Web   │
   │Graph + │ │AVPlayer│ │AVPlayer │  │video_pl.│  │variant │  │ engine │
   │Bright- │ └────────┘ └─────────┘  └─────────┘  └────────┘ └────────┘
   │Script  │
   └────────┘
        │          │           │            │            │          │
        └──────────┴───────────┴─────┬──────┴────────────┴──────────┘
                                     ▼
                    ┌────────────────────────────────────┐
                    │  ORIGIN / PACKAGER                  │
                    │  LL-HLS, multi-rendition ABR ladder │
                    │  1080p · 720p · 480p · 360p         │
                    │  master.m3u8 per channel            │
                    └────────────────┬───────────────────┘
                                     │
                    ┌────────────────▼───────────────────┐
                    │  TELEMETRY (privacy-preserving)     │
                    │  collector → SQLite → aggregate     │
                    │  JSON → dashboard                   │
                    │  /24 network prefixes only          │
                    └────────────────────────────────────┘
```

### Why this shape

| Decision | Reasoning |
|---|---|
| **One feed API, thin native clients** | Every platform gets the same channels, artwork, and EPG from one endpoint. A schedule change propagates everywhere without shipping a build to five stores — critical, because a Roku or Apple release cycle is days, not minutes |
| **Native per platform, not a wrapper** | Roku requires SceneGraph/BrightScript; there is no webview shortcut that passes certification. tvOS demands the native focus engine and remote semantics. A cross-platform wrapper fails on the remote-control UX every reviewer tests first |
| **Staff-editable schedule** | Programming staff change showtimes weekly. Anything requiring a developer becomes stale within a month. The editor is a single authenticated page writing to the same JSON the API serves |
| **Timezone-normalized EPG** | The "on air now" row re-ticks against a fixed broadcast timezone and updates without a reload. Viewers in six countries must all see the same show marked live |

---

## 3. Publication workflows — what each platform actually demands

This is the part that is invisible until it blocks a release.

| Platform | The real constraints |
|---|---|
| **Apple (tvOS + iOS)** | Reviewer tests the app cold on real hardware. **Privacy policy and support URLs in the listing must resolve** — a redirect chain or a 404 is an immediate rejection. Streaming apps get scrutinized for content rights and for playback failure on first launch. Sign-in requirements and data-collection disclosures must match actual behavior exactly |
| **Google Play** | Data safety declaration must match what the app really transmits, including analytics SDKs. Target-API-level deadlines force a rebuild on a schedule you don't control. Content rating questionnaire is binding |
| **Roku** | The most rigid. Channel Store certification checks deep-link behavior, trick-play, memory ceilings on legacy devices, and remote-control conformance. SceneGraph performance budgets on older Roku hardware are unforgiving — an app that's fluid on a current stick can fail on a five-year-old box that a large share of the audience still uses |
| **Amazon Fire TV** | Shares the Flutter codebase with Android but has its own remote input mapping, its own store listing, and appstore-specific compatibility testing |
| **Samsung Tizen** | Web-engine based with a distinct packaging and certification path and its own TV-remote focus model |

### The lesson that cost the most

> **App store listings create permanent URL contracts.** The privacy policy, app-privacy, support, and accessibility URLs submitted to Apple, Google, and Roku must keep resolving for the life of the listing — long after the website behind them is redesigned.
>
> When the web property was migrated off its old CMS to a static build, those specific paths were identified in advance and preserved with explicit rewrites. A site redesign that quietly 404s a privacy URL doesn't break the website; it breaks the app listing, and the first notice is a store rejection or a takedown.

**Release checklist I run before any submission:**

1. Every listing URL resolves `200` — not `301` chains, not soft-404s
2. Data-collection disclosure re-verified against the current analytics configuration
3. Cold-launch test on the oldest supported hardware per platform, not the newest
4. Playback verified on a throttled connection, since reviewers frequently are on one
5. Deep links and remote back-button behavior tested to the platform's spec, not to intuition

```
[PLACEHOLDER — paste ~20 lines of sanitized client code here:
 the feed fetch + EPG parse, or the SceneGraph task node that loads
 the channel list and resolves the current on-air program.
 Redact: real endpoint hostnames, API keys, channel identifiers.]
```

---

## 4. Incident: the ABR ladder that wasn't being published

### Symptom
Viewers on weaker connections — mobile data, rural links — reported constant buffering, while the stream was flawless on good connections. Server load was normal. Encoder CPU was normal.

### Investigation
The encoder was configured to produce 720p and 480p renditions and the logs confirmed it was encoding them. But clients were reporting a single quality level.

The cause was in the playlist, not the encoder: the output profiles were **encoding** multiple renditions but **publishing** only the source rendition. The clients were pointed at the packager's auto-generated single-rendition playlist rather than the multi-rendition master. Weak connections had nothing to step down to, so the only available behavior was to stall.

### Fix
- Added every rendition to all output profiles so the ladder is actually published
- Repointed all clients at the **master playlist**, not the packager's default single-rendition manifest
- Added a **360p rung** for the weakest connections
- Raised the monitor's minimum-rendition threshold so a missing rung now alerts instead of silently degrading

**Result:** four published renditions per channel, graceful degradation on weak links, and an alarm that fires if the ladder ever collapses again.

### Open optimization
The 720p rendition currently costs nearly as much bitrate as the 1080p source for 44% of the pixels. Retuning it would raise concurrent-viewer capacity by roughly 28% on the same upstream bandwidth. Documented, quantified, queued — not shipped without a viewer-quality review, because bitrate cuts are visible.

---

## 5. Incident: a metric that measured the wrong thing

The single most transferable piece of engineering judgment in this repository.

### The trap
The packager exposes a `totalConnections` field. It is the obvious source for a per-channel viewer count, it returns a plausible number, and it is **wrong**.

The field counts the packager's own internal object instances — renditions × tracks × protocols — not human viewers. It read a steady `16` on both channels **even when one channel was transmitting zero bytes.** A dashboard built on it would have displayed confident, stable, entirely fictional audience numbers.

### How it was caught
By cross-checking the metric against an independent signal: real outbound throughput per channel. A channel sending no data cannot have viewers. The moment the two disagreed, the field's meaning was suspect — and reading the packager's semantics confirmed it.

The viewer split is now derived from measured per-channel outbound throughput, with a floor so any actively transmitting channel reports at least one.

> **The general principle:** a plausible number from a well-named field is the most dangerous kind of wrong output, because nothing about it invites scrutiny. Validate every metric against an independent signal that must move with it. This applies identically to model outputs — fluent, confident, and unverified is the failure mode that gets shipped.

### Privacy-preserving telemetry
- Only **/24 network prefixes** are retained — never full viewer IP addresses
- Geolocation resolves at the prefix level, in a separate privileged process that writes an anonymized aggregate the dashboard reads
- Collector runs on a **`systemd` timer**, aggregating into SQLite and publishing a static aggregate JSON — the dashboard never queries a live database and cannot be made to leak a row

### The count that wasn't built
The previous player displayed a "connected users" figure in the hundreds to low thousands. It was not a real measurement.

Because this network **sells airtime**, I declined to reproduce an inflated counter. Fabricated audience numbers shown to advertisers are a fraud exposure, and inflated engagement claims are a store-listing risk. The player now shows a **live badge with no number** until real telemetry justifies a figure worth publishing. Building the honest version was slower and it was the only defensible option.

---

## 6. Reliability engineering

| Concern | Implementation |
|---|---|
| **Stream monitoring** | Monitor deliberately runs on a **separate host** from the streaming origin — a monitor sharing a fate domain with the thing it watches reports "healthy" right up until both are down |
| **Alert delivery** | Alerts route through an authenticated SMTP relay after direct mail was rejected for missing reverse-DNS and a strict sender policy. Discovering your alerting is silently blackholed *during* an incident is a lesson you learn once |
| **Scheduling** | `systemd` timers rather than cron on hosts where cron isn't present; journal-integrated logging and no overlapping runs |
| **Analytics** | GA4 across web and mobile properties with a shared measurement configuration, so app and web audience are comparable rather than two disconnected numbers |

---

## 7. Code Review Notes — cross-platform and streaming defect classes

| Defect | Why it passes review | The check |
|---|---|---|
| **Hardcoded single-rendition manifest URL** | Plays perfectly on the developer's connection | Test on a throttled link. If quality never changes, ABR isn't working |
| **Device-local time in EPG logic** | Correct on the developer's machine | Set the device to a foreign timezone. "On air now" must not move |
| **Web-view shortcut for TV platforms** | Faster to build, looks fine in a screenshot | Test with a real remote. Focus, back, and trick-play conformance is what certification actually tests |
| **Metrics taken at face value** | The field name says what you wanted it to say | Cross-check against an independent signal that must correlate. If it doesn't, the field means something else |
| **Analytics added without updating disclosures** | The app works | Any SDK change requires re-verifying the Apple privacy answers and Play data-safety form before submission |

---

## 8. Live listings

| Platform | Listing |
|---|---|
| Apple App Store (iOS / tvOS) | `[PLACEHOLDER — App Store link]` |
| Google Play | `[PLACEHOLDER — Play Store link]` |
| Roku Channel Store | `[PLACEHOLDER — Roku channel link]` |
| Amazon Fire TV | `[PLACEHOLDER — Amazon appstore link]` |
| Samsung Smart TV | `[PLACEHOLDER — Samsung TV app link]` |

```
[PLACEHOLDER — screenshot grid: the app running on Roku, Apple TV, and mobile.
 Store listing screenshots are already public and safe to reuse here.]
```

```
[PLACEHOLDER — screenshot: the audience dashboard showing aggregate,
 anonymized viewer data and the ABR rendition health panel.
 Blur: any absolute figures you'd rather not publish.]
```

---

<div align="center">

**Architecture and write-up by [Ginedy McIntosh](https://github.com/ginedy-mcintosh) · Houston, TX**

</div>
