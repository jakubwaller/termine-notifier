# Termine-Notifier — Design Spec

**Date:** 2026-05-03 (last revised 2026-05-17)
**Status:** Approved (brainstorming complete; running notes in `2026-05-03-leipzig-termine-notifier-findings.md`)
**Author:** Jakub Waller

## 1. Goal

A small public web service that lets residents of German cities subscribe to
**email notifications** when an appointment slot matching their filters
becomes available at a Bürgerbüro / Bürgeramt. **Notification only — never
auto-booking.** Free, non-commercial, GDPR-compliant.

This is a multi-tenant evolution of the existing single-user
`leipzigappointmentsbot` script. The new project lives in its own repository
on Jakub's personal GitHub, initially private (flipped to public after
launch-blocker tests pass), under **AGPLv3**. The existing single-user bot is
left untouched.

**v1 ships Leipzig.** Future roadmap: **Hamburg** as the second city,
sharing all non-scraper infrastructure. The schema and module layout admit
additional cities by adding a `scraper.py` plugin per platform, not by
parametrizing the existing scraper. All cities share the same public URL
(`termine.jakubwaller.eu`); the user picks city on the signup form.

## 2. Non-goals (v1)

- **No automated booking** on behalf of users — ever. The README states this
  as a project norm; PRs that add booking functionality will be rejected.
- **No accounts / login.** Signed tokens in confirmation, manage, unsubscribe,
  and renewal URLs are sufficient.
- **No Telegram delivery.** Considered and rejected (German-press perception
  of Telegram + scalper-scene pattern-match risk). No `channel` column in
  the schema — YAGNI.
- **No use of city logos, photos, screenshots, or marketing copy** from
  leipzig.de / other municipal sites. Service and location *names* (e.g.,
  "Bürgerbüro Otto-Schill-Straße") stay verbatim because they are factual
  identifiers, not creative assets.
- **No cookies, no third-party JS, no client-side analytics.**

## 3. Architecture

### 3.1 Topology

Runs on Jakub's Raspberry Pi (SSD-backed, with a USB HDD attached for
backups). Everything inside Docker Compose; nothing on the host except
Docker itself.

```
                     +----------------------------+
 internet -----TLS---|  caddy (in docker)         |
                     |  termine.jakubwaller.eu    |
                     |  auto Let's Encrypt        |
                     +-------------+--------------+
                                   | web:8000
                  +----------------v---------------+
                  |  docker compose project         |
                  |                                  |
                  | +------+ +--------+ +---------+ |
                  | | web  | | poller | | backup  | |
                  | | Flask| | python | | sqlite3 | |
                  | +--+---+ +---+----+ +----+----+ |
                  |     \         \         |       |
                  |    +---------------+    |       |
                  |    | sqlite volume |    |       |
                  |    | /data/app.db  |<---+       |
                  |    +-------+-------+            |
                  |            |                    |
                  |    +-------v----------+         |
                  |    | usb-hdd volume   |         |
                  |    | /backup/*.db.gz  |         |
                  |    +------------------+         |
                  +---------------------------------+
                            |              |
                            | HTTPS        | HTTPS (scraper)
                            v              v
                +-----------------+  +----------------------------+
                |  Mailjet (EU)   |  | terminvereinbarung.        |
                |  primary        |  | leipzig.de (Smart CJM)     |
                +--------+--------+  +----------------------------+
                         |
                         | 429 / 5xx / quota → failover
                         v
                +-----------------+
                |  Resend (EU)    |
                |  failover       |
                +-----------------+
```

The USB HDD is mounted on the Pi host at `/mnt/backup` and bind-mounted
into the `backup` container as `/backup`. A secondary off-Pi target (rsync
to laptop/NAS) is documented in `docs/deployment.md` but not orchestrated
by compose.

### 3.2 Components

#### `caddy` container

Single image `caddy:2`, two-line `Caddyfile`:

```
termine.jakubwaller.eu {
    reverse_proxy web:8000
}
```

Caddy handles ACME (Let's Encrypt) automatically, persists certs in a
docker volume, serves HTTP→HTTPS redirect, and listens on the host's 80/443.

#### `web` container — Flask + gunicorn

Two gunicorn workers (ample for Pi). Routes:

| Route                    | Method | Purpose                                                                |
| ------------------------ | ------ | ---------------------------------------------------------------------- |
| `/`                      | GET    | Subscription form (Jinja2; de/en via `?lang=` or `Accept-Language`)    |
| `/subscribe`             | POST   | Validate, rate-limit, honeypot check, insert pending row, send confirm |
| `/confirm/<token>`       | GET    | Mark subscription as confirmed; send the **separate manage-link email** |
| `/unsubscribe/<token>`   | GET    | Soft-delete subscription                                               |
| `/manage/<token>`        | GET    | View / edit own filters                                                |
| `/manage/<token>`        | POST   | Apply changes                                                          |
| `/renew/<token>`         | GET    | Reset `expires_at = now + SUBSCRIPTION_TTL_DAYS`                       |
| `/go/<slot_token>`       | GET    | Booking-link redirect (re-acquires fresh `wsid` if session-bound — gated on the launch-blocker test, see §6) |
| `/admin`                 | GET    | Aggregate operational stats (token-protected); see §3.2.7              |
| `/datenschutz`           | GET    | Privacy policy                                                         |
| `/impressum`             | GET    | Imprint                                                                |
| `/healthz`               | GET    | Liveness (returns 200 if `SELECT 1` succeeds)                          |

**Tokens.** HMAC-SHA256 of `(subscription_id, purpose, version)` keyed with
`TOKEN_SECRET_PRIMARY` from `.env`, base64url-encoded. Verification accepts
both `TOKEN_SECRET_PRIMARY` and `TOKEN_SECRET_PREVIOUS` so secrets can be
rotated without invalidating existing tokens (see §3.6).

**Subscribe rate-limit and honeypot.** `/subscribe` enforces:

- ≤ 5 successful submissions per source IP per hour.
- ≤ 1 confirmation email sent to any given email address per 24h.
- A hidden honeypot form field (CSS-hidden, named `website`). Submissions
  with that field populated are silently 200-OK'd without taking any action.

#### `poller` container — single Python loop

Sleeps until next minute boundary, runs one polling cycle. Container-level
isolation already serializes; no in-process lock needed beyond Python's
synchronous flow.

A polling cycle:

1. **Housekeeping check.** If `meta.last_housekeeping_at` is more than 24h
   old, run the housekeeping pass (renewal reminders, expiries,
   hard-purges, `seen_slots` prune, 30/60-day reassurance heartbeats,
   nightly summary email) before continuing.
2. **Load active subscriptions** (`confirmed_at IS NOT NULL AND deleted_at
   IS NULL AND expires_at > now()`), grouped by `city`.
3. **For each city**, dispatch to its scraper plugin (see §3.2.5).
4. **Plan-cap enforcement.** Collapse subscriptions into at most
   `MAX_PLANS_PER_CITY=10` distinct plans by `(appointment_type, normalized
   location_set)`. Overflow subscribers are silently merged into the
   broadest available plan ("alle Standorte") and their match filtering
   happens locally on our side. New signups that can't be absorbed return
   HTTP 503 (see §3.2.6).
5. **For each plan**, the scraper returns a list of `Slot(date, time,
   location_uuid, service_uuid, booking_token)` records.
6. **Match-and-filter per subscription.** For each subscription whose
   `(appointment_type, location_set)` matches the plan, additionally
   filter slots by the subscriber's **weekday and time-of-day filter**
   (see §3.2.4), then compute `slot_hash = sha256(date|time|location_uuid|
   service_uuid)` and drop hashes already in `seen_slots` for that
   subscription within the dedup window (default 24h).
7. **Rate-limit per subscription.** If `last_notified_at` is within
   `RATE_LIMIT_MINUTES=15`, skip emailing this cycle.
8. **Send digest.** One email per subscription listing all matched slots
   grouped by day, with a per-cycle disclaimer that "viele Termine öffnen
   sich gleichzeitig — schneller Klick gewinnt." Sent via the mail layer
   (§3.2.3).
9. **Record `seen_slots` rows** only on confirmed Mailjet/Resend success.

#### Mail layer (`app/mail.py`)

- **Primary:** Mailjet Send API v3.1 (EU region).
- **Failover:** Resend (EU region) — activated when Mailjet returns 429 or
  5xx, or when the daily quota (configurable, default 6000) is exhausted.
- **Idempotency.** Each send is keyed by `(subscription_id,
  slot_hash_set_hash, cycle_id)`. A small `sent_idempotency` table records
  this key just before the API call; on retry / failover, the table is
  consulted to avoid double-send.
- **List-Unsubscribe** headers: RFC 8058 one-click + the URL form.

#### Time-of-day / weekday filter

Subscription filters extend to:

```json
{
  "appointment_types": ["...uuid..."],
  "locations": ["...uuid..."]   // or "all"
  "weekdays": [1, 2, 3, 4, 5],  // ISO 8601: 1=Mon, 7=Sun. Default: all 7.
  "time_window": {              // 24h clock. Default: 00:00–23:59.
    "start": "08:00",
    "end":   "18:00"
  }
}
```

Filtering is applied **per subscription** in polling step 6, AFTER the
shared plan-level query. The poller still requests "all slots" from the
city; the time filter narrows what to email. This avoids fragmenting
polling plans by time-of-day (which would explode the plan-cap).

#### Scraper plugin interface (`app/scrapers/`)

A scraper module exposes:

```python
def poll(plan: PollPlan, http: requests.Session) -> list[Slot]: ...
```

where `PollPlan` carries the appointment-type and location-set for a single
group. Day-1 ships exactly **one** scraper:

- `app/scrapers/smartcjm.py` — port of the existing
  `leipzigappointmentsbot.py` flow. Reacquires `wsid` at the start of every
  cycle so a stale session never lingers.

Future scrapers (Hamburg ODControls, Frankfurt TEVIS, Stuttgart Konsentas)
are added as sibling modules selected by the subscription's `city`.

#### Plan-cap overflow behavior

When a new signup's filter would create an 11th distinct plan that cannot
be absorbed into the existing 10:

1. Try merging into the broadest plan for the same city ("alle Standorte"
   for that appointment type).
2. If that plan also doesn't exist and creating it would exceed the cap,
   reject the signup with HTTP 503: *"Aktuell ist die Warteliste voll.
   Bitte in ein paar Tagen erneut versuchen."* The form preserves entered
   values for retry.

With 10 plans and "alle" as the catch-all, the 503 path is a corner case.

#### Admin endpoint & nightly summary

**`/admin`** is GET-only, returns a plain HTML/JSON page with aggregate
stats. Access guarded by `ADMIN_TOKEN` (compared in constant time; passed
via `Authorization: Bearer …` or `?token=…`).

Stats returned (segmented by city when relevant):

- `active_subscriptions`, `pending_confirmation`, `expired_last_30d`
- `signups_last_24h`, `signups_last_7d`
- `digests_sent_today`, `digests_sent_last_7d`
- `requests_to_upstream_today` (per city)
- `parser_zero_match_minutes_today` (per city)
- `mailjet_quota_used_today`, `resend_fallback_count_today`
- `last_housekeeping_at`, `last_backup_at`, `db_size_bytes`
- `current_plan_count` (per city)

Stats are aggregate only. No per-subscription detail, no email addresses,
no IPs. Stays clean of DSGVO subject-data-on-screen issues.

**Nightly summary email** is sent to `DEVELOPER_EMAIL` at the end of the
daily housekeeping pass. Same numbers as `/admin`, formatted as a short
plain-text email. So Jakub doesn't have to remember to check.

#### Housekeeping pass

- **Renewal reminders** at `expires_at - 10 days` (`RENEWAL_REMINDER_DAYS_BEFORE`).
- **30-day and 60-day reassurance heartbeats** to subscriptions with
  `confirmed_at < now() - 30d` (or 60d) and `last_notified_at IS NULL` or
  no match since `confirmed_at + 30d`, and no heartbeat sent yet at that
  milestone. Tracked in `subscriptions.heartbeat_30d_at` / `_60d_at`.
- **Soft-delete** subscriptions past `expires_at`.
- **Hard-purge** rows with `deleted_at < now() - 30d`.
- **Prune `seen_slots`** older than 7 days; prune `sent_idempotency`
  older than 14 days.
- **Trigger `backup` container** (or relies on it running on its own
  schedule — see §3.6 backups).
- **Compose and send nightly summary email** to `DEVELOPER_EMAIL`.
- **SMART check.** Once per week, run `smartctl -a /dev/<ssd>` and email
  the developer if reallocated-sectors or wear indicators move. (See §3.6
  for the host-side helper script.)

### 3.3 Data model

```sql
CREATE TABLE subscriptions (
  id                INTEGER PRIMARY KEY AUTOINCREMENT,
  email             TEXT NOT NULL,
  city              TEXT NOT NULL DEFAULT 'leipzig',  -- dispatch key for scrapers
  language          TEXT NOT NULL DEFAULT 'de',       -- 'de' or 'en'
  filters_json      TEXT NOT NULL,                    -- types, locations, weekdays, time_window
  created_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  confirmed_at      TIMESTAMP,
  last_notified_at  TIMESTAMP,
  expires_at        TIMESTAMP NOT NULL,               -- created_at + 90d; bumped only by /renew
  reminder_sent_at  TIMESTAMP,
  heartbeat_30d_at  TIMESTAMP,
  heartbeat_60d_at  TIMESTAMP,
  deleted_at        TIMESTAMP
);
CREATE INDEX idx_active_subs ON subscriptions(deleted_at, confirmed_at, expires_at, city);

CREATE TABLE seen_slots (
  subscription_id INTEGER NOT NULL,
  slot_hash       TEXT NOT NULL,
  sent_at         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (subscription_id, slot_hash),
  FOREIGN KEY (subscription_id) REFERENCES subscriptions(id) ON DELETE CASCADE
);
CREATE INDEX idx_seen_sent_at ON seen_slots(sent_at);

CREATE TABLE sent_idempotency (
  idem_key        TEXT PRIMARY KEY,                   -- sha256(sub_id|slot_hash_set|cycle_id)
  provider        TEXT NOT NULL,                      -- 'mailjet' | 'resend'
  sent_at         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_sent_idem_at ON sent_idempotency(sent_at);

CREATE TABLE meta (
  key        TEXT PRIMARY KEY,
  value      TEXT NOT NULL,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
-- last_housekeeping_at, last_backup_at, schema_version, smart_last_check_at,
-- per-city: requests_today, parser_zero_match_since, etc.
```

Stored fields: email, city, language, filters, timestamps. **No name, no IP
retention, no analytics.** Mailjet and Resend each receive the recipient
email and slot list at send time; their own retention applies. Both have
EU-region AVVs.

### 3.4 Configuration

`.env` (gitignored, mounted into all containers):

```
# Mail — primary
MAILJET_API_KEY=...
MAILJET_API_SECRET=...
MAILJET_FROM_EMAIL=termine@jakubwaller.eu
MAILJET_FROM_NAME=Leipzig-Termine
MAILJET_DAILY_QUOTA=6000

# Mail — failover
RESEND_API_KEY=...

# Tokens (dual-secret for rotation)
TOKEN_SECRET_PRIMARY=<32+ bytes>
TOKEN_SECRET_PREVIOUS=<32+ bytes or empty>

# Admin
ADMIN_TOKEN=<32+ bytes>

# URLs
PUBLIC_BASE_URL=https://termine.jakubwaller.eu

# Timing
DEDUP_WINDOW_HOURS=24
RATE_LIMIT_MINUTES=15
SUBSCRIPTION_TTL_DAYS=90
RENEWAL_REMINDER_DAYS_BEFORE=10

# Polling
MAX_PLANS_PER_CITY=10
PARSER_CANARY_THRESHOLD_HOURS=2

# Abuse control
SUBSCRIBE_RATELIMIT_PER_IP_PER_HOUR=5
SUBSCRIBE_RATELIMIT_PER_EMAIL_PER_DAY=1

# Operations
DEVELOPER_EMAIL=jakub@jakubwaller.eu
KOFI_URL=https://ko-fi.com/jakubwaller
```

The `catalog/` directory ships `leipzig/appointment_type.json` and
`leipzig/locations.json` (copied verbatim from the existing repo). Each
future city adds its own subdirectory.

### 3.5 DSGVO posture

- **Lawful basis:** Art. 6(1)(a) DSGVO consent via double opt-in.
- **Data minimization:** email + city + language + filters + timestamps.
- **Storage limitation:** 90-day TTL from creation, bumped only by `/renew`.
  Hard-purge 30 days after `deleted_at`.
- **Transparency:** Datenschutzerklärung lists every field, retention, and
  the sub-processors (Mailjet and Resend, both EU regions, both with AVVs
  named and linked).
- **User rights:** every email has a one-click unsubscribe; RFC 8058
  `List-Unsubscribe` and `List-Unsubscribe-Post` headers set. Manage link
  is sent separately, not in digests.
- **Hosting:** all subscription data on the Pi in Germany. No third-country
  transfer beyond Mailjet/Resend API calls.
- **Cookies:** none.

### 3.6 Error handling, observability & operations

- **Cycle-level exception handling.** Exceptions inside polling cycles are
  caught at the cycle boundary, logged to stdout (→ journald via Docker),
  silently retried next minute. A persistent failure counter triggers an
  alert email to `DEVELOPER_EMAIL` once per failure burst (max one per
  24h).
- **Parser canary.** A persisted counter (`meta.parser_zero_match_since`,
  per city) tracks the last cycle that produced at least one slot. If
  this exceeds `PARSER_CANARY_THRESHOLD_HOURS=2` during typical-load
  hours (08:00–20:00 Europe/Berlin), email the developer.
- **Mail provider failover.** Mailjet 429 / 5xx / quota-exceeded → retry
  via Resend with the same idempotency key. Both fail → log + retry next
  cycle (the `seen_slots` write is conditional on confirmed send).
- **`/healthz`** returns 200 if `SELECT 1` succeeds.
- **Token-secret rotation.** Documented in `docs/deployment.md`:
  1. Set `TOKEN_SECRET_PREVIOUS=$TOKEN_SECRET_PRIMARY`.
  2. Generate new 32-byte secret as `TOKEN_SECRET_PRIMARY`.
  3. `docker compose restart`. Existing tokens stay valid until next
     rotation, then invalidate.
- **Backups (`backup` container).**
  - SQLite runs in WAL mode (`PRAGMA journal_mode=WAL`).
  - The `backup` container is a small image (alpine + sqlite3 + gzip)
    that loops: every 24h it runs `sqlite3 /data/app.db ".backup
    /backup/app-$(date +%F).db"` then `gzip` the result.
  - The `/backup` directory inside the container is a bind mount from
    `/mnt/backup` on the Pi host — which is the USB HDD.
  - Retention: 30 daily snapshots in `/backup`; older snapshots deleted
    in the same loop.
  - Off-Pi secondary: `docs/deployment.md` documents `rsync` from
    `/mnt/backup` to Jakub's laptop/NAS over SSH; runs from the laptop
    side on a personal schedule, not from the Pi. Not part of compose.
  - `meta.last_backup_at` updated on each successful backup; failure
    triggers alert email.
- **SMART monitoring.** A small host-side helper script `smartcheck.sh`
  runs from a systemd timer once per week (Docker containers can't
  access `/dev/sd*` directly without privileged mode, which we avoid).
  Output is appended to a file the backup container picks up and
  emails if reallocated-sector count moved.
- **IP-block runbook.** Documented in `docs/deployment.md`: if Leipzig
  starts 403'ing the Pi's IP, the procedure is to contact Stadt Leipzig
  at `verwaltung@leipzig.de` or via 0341 115, explain the operation,
  and negotiate. **No code-level workaround** (rotating proxies is
  ethically worse than the polling itself).

### 3.7 IP / brand-assets rule

The site uses zero municipal logos, photos, screenshots, or marketing copy.
Service and location *names* are factual identifiers and remain verbatim.
The Impressum names Jakub Waller as the responsible person. The site footer
reads `Made with ❤️ in Leipzig · Impressum · Datenschutz · ☕ Kaffee
spendieren` (the last linking to `https://ko-fi.com/jakubwaller`).

The disclaimer prominent on the homepage (lifted from KVR Alert München's
pattern, adapted per city):

> Diese Website ist nicht offiziell mit der Stadt Leipzig oder den
> Bürgerbüros der Stadt Leipzig verbunden. Wir sind ein unabhängiger
> Dienst, der ausschließlich über verfügbare Termine informiert.

## 4. Repo & deployment

### 4.1 Repo layout

```
termine-notifier/
├── README.md                       # what it does, what it explicitly does NOT do (no booking),
│                                   # link to design spec and findings, AGPLv3 statement, ko-fi
├── LICENSE                         # AGPLv3 verbatim
├── .gitignore                      # .env, *.db, data/, __pycache__
├── docker-compose.yml
├── Caddyfile
├── Dockerfile.web
├── Dockerfile.poller
├── Dockerfile.backup
├── pyproject.toml
├── data/                           # gitignored, SQLite volume mountpoint
├── catalog/
│   └── leipzig/
│       ├── appointment_type.json
│       └── locations.json
├── app/
│   ├── __init__.py
│   ├── web.py
│   ├── poller.py
│   ├── housekeeping.py
│   ├── admin.py                    # /admin endpoint + nightly summary composition
│   ├── mail.py                     # Mailjet primary + Resend failover + idempotency
│   ├── tokens.py                   # dual-secret HMAC
│   ├── db.py
│   ├── config.py
│   ├── models.py                   # Subscription, Slot, PollPlan dataclasses
│   ├── scrapers/
│   │   ├── __init__.py             # dispatcher by city
│   │   └── smartcjm.py             # Leipzig (Smart CJM)
│   ├── templates/
│   │   ├── base.html
│   │   ├── form.html
│   │   ├── confirm_sent.html
│   │   ├── manage.html
│   │   ├── admin.html
│   │   ├── datenschutz.html
│   │   └── impressum.html
│   ├── i18n/
│   │   ├── de.json
│   │   └── en.json
│   └── emails/
│       ├── confirm.{de,en}.txt
│       ├── manage_link.{de,en}.txt
│       ├── digest.{de,en}.txt
│       ├── renewal.{de,en}.txt
│       ├── heartbeat.{de,en}.txt
│       └── nightly_summary.txt     # developer-only, no i18n needed
├── scripts/
│   ├── backup-loop.sh              # entrypoint for the backup container
│   └── smartcheck.sh               # host-side, called via systemd timer
├── tests/
│   ├── test_smartcjm_parser.py     # golden HTML fixture regression
│   ├── test_tokens.py              # incl. dual-secret rotation
│   ├── test_filters.py             # weekday + time-of-day filter
│   ├── test_dedup.py
│   ├── test_idempotency.py
│   ├── test_subscribe_flow.py
│   └── test_ratelimit.py
└── docs/
    ├── deployment.md               # Caddy, systemd timer for smartcheck, USB HDD mount,
    │                               # off-Pi rsync, token rotation, IP-block runbook
    └── specs/
        ├── 2026-05-03-leipzig-termine-notifier-design.md      # this file
        └── 2026-05-03-leipzig-termine-notifier-findings.md    # brainstorming notes
```

### 4.2 docker-compose.yml (sketch)

```yaml
services:
  caddy:
    image: caddy:2
    ports: ["80:80", "443:443"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    restart: unless-stopped

  web:
    build: { context: ., dockerfile: Dockerfile.web }
    env_file: .env
    volumes:
      - ./data:/data
    restart: unless-stopped

  poller:
    build: { context: ., dockerfile: Dockerfile.poller }
    env_file: .env
    volumes:
      - ./data:/data
    restart: unless-stopped

  backup:
    build: { context: ., dockerfile: Dockerfile.backup }
    env_file: .env
    volumes:
      - ./data:/data:ro
      - /mnt/backup:/backup
    restart: unless-stopped

volumes:
  caddy_data:
  caddy_config:
```

Base image for web/poller: `python:3.12-slim` (linux/arm64 multi-arch).
Backup image: `alpine:3.20` + sqlite3 + gzip + a bash loop. Combined RAM
target: < 250 MB.

### 4.3 License & repo norms

- **License:** AGPLv3, file committed Day-1.
- **Repo location:** `github.com/jakubwaller/termine-notifier`, initially
  **private**. Flipped to public after launch-blocker tests pass and
  friends/family beta is stable.
- **README explicitly states**: "This is a free, public-interest
  notification service. It does not book appointments, and it never will.
  Pull requests that add booking functionality will not be accepted."

### 4.4 Email infrastructure prerequisites (Day-1 blockers)

Before the first send goes out:

- **SPF** record on `jakubwaller.eu` includes Mailjet and Resend sending IPs.
- **DKIM** keys configured for both Mailjet and Resend on
  `termine.jakubwaller.eu`.
- **DMARC** policy `v=DMARC1; p=quarantine; rua=mailto:jakub@jakubwaller.eu`
  initially; tighten to `p=reject` after two clean weeks.

## 5. Testing

- **Unit tests** for token sign/verify (incl. dual-secret rotation), filter
  matching (incl. weekday/time-of-day), dedup, idempotency, rate-limit,
  honeypot detection.
- **Golden HTML parser fixture.** A captured real Leipzig poll response,
  fed through `app/scrapers/smartcjm.py`. The parser is the most fragile
  component; this is the regression net.
- **Integration test** spinning up Flask with an in-memory SQLite and
  walking subscribe → confirm → manage → unsubscribe.
- **No live HTTP tests** against `terminvereinbarung.leipzig.de` in CI
  (flaky, rude). Mailjet and Resend wrapped in thin interfaces so they're
  stubbable.

## 6. Rollout plan

1. **Friends & family beta (2 weeks).** Quietly operating service, ~5
   subscribers, watch journald, fix anything broken. Repo stays private.
   **Launch-blocker tests in this window:**
   - **Booking-link session-bound check.** Click an `appointment_reserve`
     URL from a real digest email in a fresh browser. If it 404s or
     session-expired, build `/go/<slot_token>` that re-acquires `wsid`
     before redirecting. Verify the redirect doesn't bind our IP's
     session to the user's reservation in any harmful way.
   - SPF/DKIM/DMARC verification via mail-tester.com or equivalent.
   - Parser canary, rate-limit, honeypot all firing as expected under
     simulated load.
   - Backup container produces gzipped snapshots on the USB HDD;
     restore-from-backup tested at least once.
2. **Flip repo public** on GitHub once friends/family beta is stable.
3. **Declarative email to Stadt Leipzig.** Before any public link. One
   page: what we do, what we don't do, DSGVO posture, polling cadence
   (≤10 req/min), open-source link, public contact. Framed as
   notification, not request.
4. **Soft launch.** Link from r/Leipzig and pitch leipglo.com via their
   "Write For Us" submission. Watch for 2–4 weeks.
5. **Day-30 calendar entry** post-launch: confirm the declarative email
   was sent and acknowledged or not; follow up if needed.
6. **Hamburg spike** (later, no public commitment). One developer-week
   to write `app/scrapers/odcontrols.py`. Only after Leipzig is stable.

## 7. References

- `docs/superpowers/specs/2026-05-03-leipzig-termine-notifier-findings.md` —
  full brainstorming notes, competitive landscape, premortems v1 and v2,
  all rationale for the decisions in this spec.
- Existing single-user bot: `github.com/jakubwaller/leipzigappointmentsbot`
  (referenced as prior art in the new repo's README).
- KVR Alert München (`kvr-alert-muenchen.de`) — reference implementation
  for free-notifier UX and disclaimer wording.
