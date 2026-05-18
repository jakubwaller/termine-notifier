# Findings (running notes)

Living doc — not the spec. Stuff we've learned while brainstorming.

## Legal

- **BGH I ZR 224/12 (2014)** — scraping freely-accessible public sites is not automatically unfair competition; AGB alone cannot prohibit it. Foundation of "the notifier is defensible."
- **Leipzig has no Terms of Service for the booking site.** No AGB, no Nutzungsbedingungen on `terminvereinbarung.leipzig.de`. Smart CJM (vendor) publishes none either.
- **`robots.txt = Disallow: /`** on the booking subdomain. Not legally binding in DE, but signals operator intent. Weakens any "we didn't know" defense.
- **Auto-booking is the legal red line.** Berlin Senat confirmed black market; Ausländerbehörde resellers earned €800–1000/mo charging €40–100/slot. We're not building that.
- **Stadt Köln tried to legally challenge `terminator.koeln` in 2022** — not on scraping grounds, on **image rights** (city photos used without permission). Threat didn't convert (site is still up 3+ years later), but it tells us cities will look for any hook. **Implication: no Stadt-Leipzig assets, logos, screenshots, or copied page text on our site.**
- **DSGVO posture for a notifier is light**: Art. 6(1)(a) consent via double opt-in, email + filters + timestamps only, EU hosting, German-resident DB. Auftragsverarbeitung (Art. 28) becomes heavy only if/when we ever auto-book — another reason we don't.

## Competitive landscape

| City | Free notifier | Paid auto-booker | Notes |
|---|---|---|---|
| **Leipzig** | **None** | Terminli (€8.95+/booking) | Gap. Our target. |
| **Berlin** | AllAboutBerlin finder; multiple Bürgerbots | berlintermin.de, Terminli | Saturated. |
| **München** | **KVR Alert München** (free, web, notify-only) | Terminli (coming) | Closest model to ours. |
| **Hamburg** | None found | Terminli | Gap. |
| **Frankfurt** | None found | Terminli | Gap. |
| **Köln** | terminator.koeln (Angular SPA, **likely parser-rotted**) | Terminli | Cautionary tale. |
| **Stuttgart** | None | None for Bürgerbüro | Gap. |

- **Terminli is McMoe's Solutions.** Paid, auto-books, 40+ cities, blog cites MDR coverage of the overload problem (not of Terminli itself — earlier claim corrected).
- **KVR Alert München** is the structural reference: free, notify-only, account-based, explicitly disclaims city affiliation. Worth copying their disclaimer language.
- **`terminator.koeln`** (by David Neukirchen) — Angular PWA, frontend loads but data layer is dead. Classic single-maintainer parser-rot when the city's backend changes. Important warning for us.
- **Köln and Berlin** are NOT on Smart CJM — different vendor backends. Our "Leipzig codebase → multi-city" hope holds only where Smart CJM is the platform (Leipzig confirmed; Hamburg/Frankfurt/Stuttgart to verify before banking on it).

## Press / context

- **MDR** is said to have run the Bürgerbüro-overload story in March and November 2024 (headlines: "Lange Wartezeiten und viel Stress" / "Bürgerbüro Leipzig vergibt Termine neu"), but the only source for this is Terminli's own blog name-dropping them — no primary MDR URLs verified. Treat as "audience problem is mainstream-press-adjacent" rather than "MDR-endorsed."
- **L-IZ.de** covered Leipzig Bürgerbüro service restrictions in April 2024 and "Ausprobiert: Termine im Bürgerbüro" in August 2023.
- **Netzpolitik.org** covered Berlin's appointment crisis & black market (2023).
- **iamexpat.de / The Berliner** covered Ausländerbehörde scalping.
- **Berlin (Nov 2025)** rolled out walk-in-without-appointment for several services. Possibly a model other cities will follow → demand could naturally decline.

## Publishing channels (verified or assumed)

- **leipglo.com** — confirmed live, English Leipzig webzine, accepts contributor pitches at `/write-for-us/`. Newcomer audience. **High-fit, pitch them.**
- **l-iz.de** — confirmed active, civic-tech coverage. Pitch after first weeks of operation.
- **r/Leipzig** — known active, mixed DE/EN. Soft-launch channel.
- **Uni Leipzig & HTWK International Offices** — direct email outreach for newcomer newsletters.
- **Welcome Center Sachsen** — state-level newcomer service.
- **MDR** — high bar but warm-on-the-story.
- **Avoid**: Telegram public channels (pattern-matched with scalpers); mass FB groups.

## Risks we've named (premortem)

1. Email deliverability cold start (SPF/DKIM/DMARC must be Day-1).
2. Parser rot when Leipzig changes HTML — alerting required ("zero matches across all plans for >2h").
3. Booking link in emails likely session-bound to our `wsid` — must test, may need `/go/<slot_id>` redirect that re-acquires wsid.
4. Fan-out amplification — cap distinct polling plans.
5. **SSD failure** (low probability, usually pre-warned via SMART). Risk is materially reduced vs SD card (which would die in 6–18 months under SQLite+Docker load), but **not eliminated** — controllers fail, power events corrupt, firmware bugs happen. **Mitigation**: WAL mode on SQLite + nightly `sqlite3 .backup` to a separate target (off-host preferred — e.g., Hetzner Storage Box ~€3/mo, ARM-compatible) + weekly `smartctl -a` SMART check with alert on reallocated-sector growth.
6. Cease-and-desist after media coverage — clearance email to Stadt Leipzig as good-faith record.
7. Mailjet free-tier daily cap (200/day) — quota guard.
8. Token secret leak — dual-secret rotation procedure.
9. "Silence = consent" framing is wrong in German admin law — drop it.
10. German-only UI cuts off the newcomer cohort — **English added to Day-1 scope.**
11. `/manage` token in every digest = forwardable power — move to a separate quarterly email.
12. Long-silent subscriber = degraded reputation; needs a 30-day reassurance ping.
13. **(New)** Don't use any Stadt Leipzig assets — Köln precedent.

## Premortem v2 (after all v1 mitigations locked in)

Pressure-tested the current plan. Sixteen new failure modes; four are launch-killers.

**Launch-killers (must address pre-MVP):**

- **#v2-1 Soft-launch isn't silent.** r/Leipzig + leipglo get scraped/indexed; Stadt Leipzig hears about us via screenshot before our notification email. Mitigation: the notification email goes out *before* any public link, not after — invert step 4 in the rollout plan.
- **#v2-2 Mailjet+Resend doesn't fix burst congestion.** Both providers throttle when 200 users need pings within 60s. Failover handles outage, not slow-under-load. Mitigation: parallel batched sends + accept that some users lose the slot race; users see "viele Termine gleichzeitig — die ersten Klicks gewinnen" in digest copy.
- **#v2-3 Single-IP polling is trivially pattern-matchable.** When Leipzig 403s the Pi, we have no clean fallback. Rotating proxies is ethically worse than the polling. Mitigation: operational runbook only — "if IP blocked, contact Stadt Leipzig, explain, negotiate." No code workaround.
- **#v2-4 `/subscribe` is a spam fan-out vector.** Scripted signups with victim emails → Mailjet bill explodes → blacklist. Mitigation: per-IP + per-email rate limit on `/subscribe` (5/h per IP, 1/24h per email-address attempted), plus a **honeypot form field** (no third-party JS, keeps the cookie-free pitch intact).

**Slow erosion:**

- **#v2-5** The 30/60-day heartbeat may worsen deliverability (spam-marked) — the v1 mitigation for #12 fights v1 #1.
- **#v2-6** 90-day TTL optimizes DSGVO, not retention. Real-world abandonment will be brutal.
- **#v2-7** Pi is a single point of failure (one apartment, one ISP, one power strip). No HA in the spec.
- **#v2-8** "Alerts to developer email" isn't paging. Jakub on vacation = silent degradation.

**Legal/positioning gaps:**

- **#v2-9** We already copy Stadt Leipzig text in the catalog files. The "no city assets" rule is already violated. **Decided: loose interpretation** — no logos, photos, screenshots, or marketing copy from leipzig.de. Service/location names stay verbatim (they're factual identifiers of public services, not creative assets — KVR Alert München uses Munich's service names verbatim too).
- **#v2-10** Open-sourcing the repo turns obscurity into documentation. Same UUIDs are already in the public single-user repo, so this is pre-existing state — but the AGPL'd new repo makes the picture cleaner for any future opposition.
- **#v2-11** The clearance email may never get sent once the service is running quietly. **Calendar reminder Day 30 post-launch** to send it — written into the rollout plan as a hard deadline.
- **#v2-12** AGPL is symbolic against scalpers. Real protection is reputational (norms in README), not legal.

**Technical landmines:**

- **#v2-13** Hamburg ODControls may 5× the "2–3 days new parser" estimate. Don't promise Hamburg timelines publicly until a spike is done.
- **#v2-14** `/go/<slot_id>` redirect binds *our IP's session* to the user's booking. Unknown whether this causes problems on the city's side. Test before relying on it.
- **#v2-15** Dual-provider failover can double-send when Mailjet's 500 is a false signal. Mitigation: idempotency on the send side, keyed by `(subscription_id, slot_hash_set, cycle_id)`, not just dedup on the receive side.
- **#v2-16** Double-opt-in confirmation is a low-grade harassment vector (sign up enemy's email). Per-IP rate-limit mitigates partially; not fully solvable.

### Decided (pre-launch additions from v2)

- **Inverted rollout step order**: send the Stadt-Leipzig declarative email *before* any public link, even at soft-launch stage. (Replaces the earlier "after 4 weeks of clean operation" framing — the soft-launch posts themselves break silence.)
- **Subscribe rate-limit**: 5/h per IP, 1/24h per email-address attempted. Honeypot form field (hidden via CSS; submissions filling it are silently dropped). No third-party captcha, no cookies.
- **Burst-congestion copy in digest email**: a line acknowledging that simultaneous slot drops favor the fastest clickers, so users don't blame us for the race-loss.
- **IP-block runbook in `docs/deployment.md`**: contact Stadt Leipzig at <verwaltung@leipzig.de> or via 0341 115, explain operation, negotiate. No code-level workaround attempted.
- **Catalog files (loose interpretation)**: verbatim Leipzig service/location names are allowed (factual identifiers). Logos, photos, screenshots, and Leipzig marketing copy are not.
- **Day-30 clearance-email reminder**: hard calendar entry. Service can be operating already; this is the "declare" half of "ship-first-declare-later" with a deadline.
- **Send-side idempotency key**: `(subscription_id, slot_hash_set_hash, cycle_id)` — prevents dual-provider double-send.

## Positioning we've converged on

- **Free forever. No auto-booking.** Single most important differentiator vs Terminli.
- **"Not officially affiliated with Stadt Leipzig"** in the footer, KVR-Alert-style.
- **German + English Day-1.**
- **Data stays on a Pi in Germany.** Cookie-free. Tiny privacy page.
- **Ship first, declare later** (decided). Quietly launch and operate cleanly for ~4 weeks (friends/family → r/Leipzig + leipglo soft-launch), *then* send a one-page declarative notification email to Stadt Leipzig — framed as "I am operating this, here is what it does and how it addresses obvious concerns" rather than "may I?". Reason: asking-first manufactures a refusal that wouldn't otherwise exist; pure stealth forfeits the good-faith record. Notify-don't-ask creates the paper trail without forcing a permission decision.
- **Design data structures so adding a city is config-not-code**, but don't ship multi-city in v1.

## Still open / worth chatting about

_All previously open threads have been resolved — see Decided section below._

### Decided

- **Premortem mitigations — locked in:**
  - **#1 Email deliverability**: SPF + DKIM + DMARC configured on `jakubwaller.eu` as Day-1 deployment prerequisite. Warm-up via friends/family for 2 weeks before any wider link.
  - **#2 Parser canary**: alert to developer email if zero matches across all active polling plans for >2h during typical-load hours (08:00–20:00 Europe/Berlin).
  - **#3 Booking-link session-bound check**: **launch-blocker test** — before MVP launch, click an `appointment_reserve` URL from a real digest email in a fresh browser session. If session-bound, build a `/go/<slot_id>` redirect that re-acquires `wsid` on the fly. Don't ship without verifying.
  - **#4 Fan-out cap**: maximum **10 distinct polling plans** at any time. Subscribers whose filters can't merge into an existing plan are routed into the "alle Standorte" plan instead. If even that can't absorb them (edge case: 10 distinct appointment-type plans all using "alle"), new signups get a "wir sind ausgelastet" deferral. Total traffic to Leipzig stays ≤10 req/min regardless of subscriber count.
  - **#8 Token-secret rotation**: dual-secret pattern — `TOKEN_SECRET_PRIMARY` signs, both `_PRIMARY` and `_PREVIOUS` verify. Rotation = promote primary to previous, generate new primary. Procedure documented in `docs/deployment.md`.
  - **#11 Manage-link forwardability**: `/manage` link **never** appears in digest emails. Only `/unsubscribe`. The manage link is sent in its own dedicated email at subscribe-confirmation and renewal-reminder (~quarterly).
  - **#12 Long-silent-subscriber heartbeat**: at day 30 of no notifications, send a reassurance email ("du bist weiterhin abonniert, dein Filter passt einfach noch nicht — hier verwalten"). Same at day 60. Day 80 is already covered by the renewal-reminder.
  - **#13 No Stadt-Leipzig assets**: spec rule — no logos, photos, screenshots, or copied page text from `leipzig.de`. Original copy only. External links to leipzig.de minimized and labeled.

- **Open source under AGPLv3.** Yes to OSS — transparency is itself a legal defense, aligns with anti-Terminli framing, enables community per-platform scrapers, and our existing single-user repo is already public so the precedent is set. License is AGPLv3 specifically because it's the only OSI-approved option that closes the "fork and run privately as a paid service" loophole (GPLv3 doesn't reach network services; MIT/BSD allows closed-source commercial forks; non-commercial licenses aren't OSI-approved). README adds a norms-layer: explicit statement that auto-booking PRs will not be accepted. Repo lives on Jakub's personal GitHub. `.env`, `data/`, secrets stay gitignored. Design spec + findings doc get committed under `docs/` as public design artifact.

- **Telegram is out.** Bad press in Germany; would associate the service with the scalper scene; pattern-match risk is real. Drop the `channel` column from the schema entirely — YAGNI until there's a clear ask for a non-email channel (and even then, web push or Signal would be more likely than Telegram).

- **Copy/UX patterns from KVR Alert München to lift** (verbatim where translated):
  - Headline pattern: "Nie wieder freie Bürgerbüro-Termine in Leipzig verpassen."
  - Disclaimer (most important line on the site): "Diese Website ist nicht offiziell mit der Stadt Leipzig oder den Bürgerbüros der Stadt Leipzig verbunden. Wir sind ein unabhängiger Dienst, der ausschließlich über verfügbare Termine informiert."
  - Three-step explainer (Anmelden / Konfigurieren / Benachrichtigt werden).
  - "100% kostenlos" framing prominently (direct anti-Terminli signal).
  - Footer "Made with ❤️ in Leipzig" or equivalent.
- **Where we're cleaner than KVR Alert** (becomes a marketing line): No account, no SSO, no IP/UA logging, no US-headquartered processors (Vercel/Supabase), data on a Pi in Germany. Lift this as: "Kein Konto. Kein Tracking. Keine US-Dienstleister. Daten leben auf einem Raspberry Pi in Deutschland."
- **Operational lift: dual email provider for redundancy.** KVR Alert uses Mailjet AND Resend simultaneously — almost certainly redundancy against daily-cap/outage. Configure Resend as failover that activates if Mailjet returns 429 or 5xx. Adds ~one day of impl work, materially reduces premortem risk #7.

- **Multi-city: yes, but as a roadmap, not v1.** Decided: Leipzig v1 → Hamburg v2 (Jakub now lives there; no free competition). Schema includes a `city` dimension Day-1. **But** every major city uses a different platform vendor — multi-city is "one parser per platform," not parametrized config. Scraper becomes a pluggable interface (`scraper.py` per platform), same subscription/email/web logic. Cheapest rollout order: Leipzig (Smart CJM, done) → Köln (likely Smart CJM, ~½ day if confirmed) → Hamburg (ODControls, ~2–3 days) → Frankfurt (TEVIS) / Stuttgart (Konsentas) only on demand.

  **Platform map (verified except where noted):**
  | City | Platform | Vendor |
  |---|---|---|
  | Leipzig | Smart CJM | Smart CJM (footer-confirmed) |
  | Hamburg | ODControls | "Mandant HH" (system version string) |
  | Stuttgart | Konsentas | Girona Softwareentwicklung GmbH |
  | Frankfurt | TEVIS | ekom21 (`tevis.ekom21.de`) |
  | Köln | Smart CJM (**unverified**, URL pattern matches Leipzig's exactly) | — |
  | München | (out of scope, KVR Alert exists) | — |
  | Berlin | service.berlin.de + Konsentas in places | mixed; saturated |

- **Ko-fi tip jar — yes, with strict placement.** A ko-fi link does NOT disrupt the "100% kostenlos" claim *if* it's positioned as a voluntary post-value tip jar, not fundraising. Precedent: Jakub's earlier COVID-appointment bot received voluntary coffees; allaboutberlin.com runs the same model. Tax/legal trivial at this scale (Freigrenze €256/yr).
  - **Allowed placements**: small footer link ("☕ Kaffee spendieren"), bottom of digest emails *after* the user has received a real match, bottom of Impressum/Datenschutz.
  - **Forbidden placements**: hero, signup page, confirmation-before-first-match, anywhere that suggests payment is needed.
  - **Forbidden framing**: "support our work," "fund the project," supporter perks, faster notifications for tippers. Tip-jar language only ("Kaffee," "Dankeschön").
  - Link target: `https://ko-fi.com/jakubwaller`.
