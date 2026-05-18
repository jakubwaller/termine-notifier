# Termine-Notifier Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a free, multi-tenant email notifier for free Bürgerbüro appointments in Leipzig (Hamburg to follow), per the design spec at `docs/superpowers/specs/2026-05-03-leipzig-termine-notifier-design.md`.

**Architecture:** Three long-lived containers (Caddy / Flask web / Python poller) plus a backup container, sharing a SQLite volume. Mailjet primary with Resend failover. Per-city scraper plugins; v1 ships only Leipzig's Smart-CJM scraper. Token-based, account-free, cookie-free.

**Tech Stack:** Python 3.12, Flask + gunicorn, SQLite, BeautifulSoup, requests, Mailjet & Resend SDKs (or thin HTTP wrappers), Caddy 2, Docker Compose, pytest, AGPLv3.

---

## File Structure

The new repository `termine-notifier/` lives separately from the current `leipzigappointmentsbotpremium/` workspace. All paths in this plan are relative to that new repo's root unless explicitly noted.

| File | Responsibility |
|---|---|
| `pyproject.toml` | Project metadata, deps, pytest config |
| `LICENSE` | AGPLv3 verbatim |
| `README.md` | What the project does, anti-booking norm, ko-fi link |
| `.gitignore` | `.env`, `data/`, `*.db`, `__pycache__`, `.venv` |
| `Caddyfile` | Single vhost, auto-TLS via Let's Encrypt |
| `docker-compose.yml` | Caddy + web + poller + backup services |
| `Dockerfile.web` | Python 3.12-slim + Flask + gunicorn |
| `Dockerfile.poller` | Python 3.12-slim |
| `Dockerfile.backup` | Alpine + sqlite3 + gzip + bash loop |
| `app/__init__.py` | Empty package marker |
| `app/config.py` | `.env` loader → typed `Config` dataclass |
| `app/db.py` | sqlite3 connection helper + schema bootstrap + migrations |
| `app/models.py` | `Subscription`, `Slot`, `PollPlan`, `Filter` dataclasses |
| `app/tokens.py` | Dual-secret HMAC sign/verify |
| `app/catalog.py` | Per-city JSON catalog loader |
| `app/filters.py` | Subscription-filter matching (incl. weekday + time window) |
| `app/scrapers/__init__.py` | Dispatcher (`city → scraper`) |
| `app/scrapers/smartcjm.py` | Leipzig scraper (port of existing bot) |
| `app/mail.py` | Mailjet primary + Resend failover + idempotency |
| `app/poller.py` | Polling loop entry point (used by `poller` container) |
| `app/housekeeping.py` | Daily pass: renewals, heartbeats, summary email, etc. |
| `app/admin.py` | `/admin` route + nightly summary composition |
| `app/web.py` | Flask app + routes + form |
| `app/ratelimit.py` | In-memory IP + email rate-limit for `/subscribe` |
| `app/templates/*.html` | Jinja2 web templates (de + en via i18n dict) |
| `app/i18n/de.json`, `app/i18n/en.json` | UI strings |
| `app/emails/*.{de,en}.txt` | Plain-text email templates |
| `catalog/leipzig/{appointment_type,locations}.json` | Service & location catalogs |
| `scripts/backup-loop.sh` | Backup container entrypoint |
| `scripts/smartcheck.sh` | Host-side SMART monitor (called via systemd timer) |
| `tests/*` | Unit + integration tests |
| `docs/deployment.md` | Caddy, USB HDD mount, SMART systemd timer, token rotation, IP-block runbook |
| `docs/specs/` | Spec + findings docs copied from this repo |

---

## Phase 0: Repository Scaffolding

### Task 0.1: Initialize the new repo

**Files:**
- Create: `termine-notifier/` (new directory, sibling to current workspace)
- Create: `termine-notifier/.gitignore`
- Create: `termine-notifier/LICENSE`
- Create: `termine-notifier/README.md`
- Create: `termine-notifier/pyproject.toml`

- [ ] **Step 1: Create the directory, init, set git identity**

```bash
mkdir -p ../termine-notifier
cd ../termine-notifier
git init -b main
git config user.name "Jakub Waller"
git config user.email termine@jakubwaller.eu
```

- [ ] **Step 2: Write `.gitignore`**

```
.env
.env.*
!.env.example
data/
*.db
*.db-shm
*.db-wal
__pycache__/
.venv/
.pytest_cache/
.coverage
htmlcov/
*.egg-info/
dist/
build/
node_modules/
```

- [ ] **Step 3: Write the AGPLv3 `LICENSE` file**

Download the canonical text:

```bash
curl -fsSL https://www.gnu.org/licenses/agpl-3.0.txt -o LICENSE
```

If offline, copy from `https://www.gnu.org/licenses/agpl-3.0.txt` manually.

- [ ] **Step 4: Write `README.md`**

```markdown
# Termine-Notifier

Free email notifications for free Bürgerbüro appointments in German cities.

**v1 covers Leipzig. Hamburg is next.**

## What this does

You enter your email, pick the appointment type (e.g. Wohnsitzanmeldung) and
which Bürgerbüros work for you, and receive an email the moment a matching
slot is available on `terminvereinbarung.leipzig.de`. You then book it
yourself on the official city website. We never book on your behalf.

## What this explicitly does NOT do

- **No automated booking.** The project will not accept pull requests that
  add booking functionality. Forks that add booking will remain forks — not
  merged upstream, not endorsed.
- **No account, no login, no tracking, no cookies, no third-party JS.**
- **No data resale, no advertising, no paid features.**

## How it works

A Raspberry Pi in Germany runs a small Docker Compose project: a Caddy
reverse proxy, a Flask web app for subscriptions, a Python poller that
checks the city's booking site once a minute, and a backup container that
snapshots the SQLite database to an external HDD.

## Not affiliated with Stadt Leipzig

This is an independent service. We only inform about available appointments.
The city of Leipzig has no involvement.

## License

AGPLv3 — see `LICENSE`.

## Support

If this helped you, you can buy me a coffee: <https://ko-fi.com/jakubwaller>
```

- [ ] **Step 5: Write `pyproject.toml`**

```toml
[project]
name = "termine-notifier"
version = "0.1.0"
description = "Free email notifier for German municipal appointment systems"
requires-python = ">=3.12"
license = { text = "AGPL-3.0-only" }
authors = [{ name = "Jakub Waller" }]
dependencies = [
  "flask>=3.0",
  "gunicorn>=22.0",
  "requests>=2.32",
  "beautifulsoup4>=4.12",
  "python-dotenv>=1.0",
]

[project.optional-dependencies]
dev = [
  "pytest>=8.0",
  "pytest-cov>=5.0",
  "freezegun>=1.5",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v"

[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
include = ["app*"]
```

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "chore: initialize project (license, readme, pyproject)"
```

---

### Task 0.2: Set up a Python virtualenv and verify installation

**Files:** None.

- [ ] **Step 1: Create venv and install**

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```

- [ ] **Step 2: Verify pytest works**

```bash
pytest
```

Expected: `no tests ran in 0.00s` (no tests yet).

- [ ] **Step 3: No commit** — `.venv/` is gitignored.

---

## Phase 1: Configuration & Database

### Task 1.1: Config loader (test first)

**Files:**
- Create: `tests/test_config.py`
- Create: `app/__init__.py`
- Create: `app/config.py`
- Create: `.env.example`

- [ ] **Step 1: Write the failing test**

`tests/test_config.py`:

```python
import os
import pytest
from app.config import Config, load_config

def test_load_config_from_env(monkeypatch):
    monkeypatch.setenv("MAILJET_API_KEY", "mj_key")
    monkeypatch.setenv("MAILJET_API_SECRET", "mj_secret")
    monkeypatch.setenv("MAILJET_FROM_EMAIL", "termine@example.eu")
    monkeypatch.setenv("MAILJET_FROM_NAME", "Termine")
    monkeypatch.setenv("MAILJET_DAILY_QUOTA", "6000")
    monkeypatch.setenv("RESEND_API_KEY", "re_key")
    monkeypatch.setenv("TOKEN_SECRET_PRIMARY", "a" * 32)
    monkeypatch.setenv("TOKEN_SECRET_PREVIOUS", "")
    monkeypatch.setenv("ADMIN_TOKEN", "b" * 32)
    monkeypatch.setenv("PUBLIC_BASE_URL", "https://termine.example.eu")
    monkeypatch.setenv("DEDUP_WINDOW_HOURS", "24")
    monkeypatch.setenv("RATE_LIMIT_MINUTES", "15")
    monkeypatch.setenv("SUBSCRIPTION_TTL_DAYS", "90")
    monkeypatch.setenv("RENEWAL_REMINDER_DAYS_BEFORE", "10")
    monkeypatch.setenv("MAX_PLANS_PER_CITY", "10")
    monkeypatch.setenv("PARSER_CANARY_THRESHOLD_HOURS", "2")
    monkeypatch.setenv("SUBSCRIBE_RATELIMIT_PER_IP_PER_HOUR", "5")
    monkeypatch.setenv("SUBSCRIBE_RATELIMIT_PER_EMAIL_PER_DAY", "1")
    monkeypatch.setenv("DEVELOPER_EMAIL", "dev@example.eu")
    monkeypatch.setenv("KOFI_URL", "https://ko-fi.com/jakubwaller")
    monkeypatch.setenv("DB_PATH", "/tmp/test.db")

    cfg = load_config()
    assert cfg.mailjet_api_key == "mj_key"
    assert cfg.token_secret_primary == "a" * 32
    assert cfg.token_secret_previous == ""
    assert cfg.subscription_ttl_days == 90
    assert cfg.max_plans_per_city == 10
    assert cfg.kofi_url == "https://ko-fi.com/jakubwaller"

def test_load_config_missing_required(monkeypatch):
    monkeypatch.delenv("MAILJET_API_KEY", raising=False)
    with pytest.raises(KeyError):
        load_config()
```

- [ ] **Step 2: Run test, verify it fails**

```bash
pytest tests/test_config.py -v
```

Expected: ImportError / ModuleNotFoundError for `app.config`.

- [ ] **Step 3: Create `app/__init__.py`** (empty file)

```bash
touch app/__init__.py
```

- [ ] **Step 4: Implement `app/config.py`**

```python
from __future__ import annotations
import os
from dataclasses import dataclass

@dataclass(frozen=True)
class Config:
    mailjet_api_key: str
    mailjet_api_secret: str
    mailjet_from_email: str
    mailjet_from_name: str
    mailjet_daily_quota: int
    resend_api_key: str
    token_secret_primary: str
    token_secret_previous: str
    admin_token: str
    public_base_url: str
    dedup_window_hours: int
    rate_limit_minutes: int
    subscription_ttl_days: int
    renewal_reminder_days_before: int
    max_plans_per_city: int
    parser_canary_threshold_hours: int
    subscribe_ratelimit_per_ip_per_hour: int
    subscribe_ratelimit_per_email_per_day: int
    developer_email: str
    kofi_url: str
    db_path: str

def _req(key: str) -> str:
    val = os.environ.get(key)
    if val is None:
        raise KeyError(f"Missing required env var: {key}")
    return val

def _req_int(key: str) -> int:
    return int(_req(key))

def load_config() -> Config:
    return Config(
        mailjet_api_key=_req("MAILJET_API_KEY"),
        mailjet_api_secret=_req("MAILJET_API_SECRET"),
        mailjet_from_email=_req("MAILJET_FROM_EMAIL"),
        mailjet_from_name=_req("MAILJET_FROM_NAME"),
        mailjet_daily_quota=_req_int("MAILJET_DAILY_QUOTA"),
        resend_api_key=_req("RESEND_API_KEY"),
        token_secret_primary=_req("TOKEN_SECRET_PRIMARY"),
        token_secret_previous=os.environ.get("TOKEN_SECRET_PREVIOUS", ""),
        admin_token=_req("ADMIN_TOKEN"),
        public_base_url=_req("PUBLIC_BASE_URL"),
        dedup_window_hours=_req_int("DEDUP_WINDOW_HOURS"),
        rate_limit_minutes=_req_int("RATE_LIMIT_MINUTES"),
        subscription_ttl_days=_req_int("SUBSCRIPTION_TTL_DAYS"),
        renewal_reminder_days_before=_req_int("RENEWAL_REMINDER_DAYS_BEFORE"),
        max_plans_per_city=_req_int("MAX_PLANS_PER_CITY"),
        parser_canary_threshold_hours=_req_int("PARSER_CANARY_THRESHOLD_HOURS"),
        subscribe_ratelimit_per_ip_per_hour=_req_int("SUBSCRIBE_RATELIMIT_PER_IP_PER_HOUR"),
        subscribe_ratelimit_per_email_per_day=_req_int("SUBSCRIBE_RATELIMIT_PER_EMAIL_PER_DAY"),
        developer_email=_req("DEVELOPER_EMAIL"),
        kofi_url=_req("KOFI_URL"),
        db_path=os.environ.get("DB_PATH", "/data/app.db"),
    )
```

- [ ] **Step 5: Create `.env.example`** with the same keys (no real values)

```
MAILJET_API_KEY=
MAILJET_API_SECRET=
MAILJET_FROM_EMAIL=termine@jakubwaller.eu
MAILJET_FROM_NAME=Leipzig-Termine
MAILJET_DAILY_QUOTA=6000
RESEND_API_KEY=
TOKEN_SECRET_PRIMARY=
TOKEN_SECRET_PREVIOUS=
ADMIN_TOKEN=
PUBLIC_BASE_URL=https://termine.jakubwaller.eu
DEDUP_WINDOW_HOURS=24
RATE_LIMIT_MINUTES=15
SUBSCRIPTION_TTL_DAYS=90
RENEWAL_REMINDER_DAYS_BEFORE=10
MAX_PLANS_PER_CITY=10
PARSER_CANARY_THRESHOLD_HOURS=2
SUBSCRIBE_RATELIMIT_PER_IP_PER_HOUR=5
SUBSCRIBE_RATELIMIT_PER_EMAIL_PER_DAY=1
DEVELOPER_EMAIL=
KOFI_URL=https://ko-fi.com/jakubwaller
DB_PATH=/data/app.db
```

- [ ] **Step 6: Run tests, verify they pass**

```bash
pytest tests/test_config.py -v
```

Expected: both tests PASS.

- [ ] **Step 7: Commit**

```bash
git add app/ tests/test_config.py .env.example
git commit -m "feat(config): typed env loader with required/optional fields"
```

---

### Task 1.2: Database schema bootstrap

**Files:**
- Create: `tests/test_db.py`
- Create: `app/db.py`

- [ ] **Step 1: Write the failing test**

`tests/test_db.py`:

```python
import sqlite3
from pathlib import Path
from app.db import connect, init_schema, SCHEMA_VERSION

def test_init_schema_creates_all_tables(tmp_path):
    db_path = tmp_path / "test.db"
    conn = connect(str(db_path))
    init_schema(conn)
    cur = conn.execute("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")
    tables = {row[0] for row in cur}
    for expected in ("subscriptions", "seen_slots", "sent_idempotency",
                     "meta", "city_state", "slots_cache"):
        assert expected in tables, f"missing table: {expected}"

def test_transaction_commits_on_success(tmp_path):
    from app.db import transaction
    db_path = tmp_path / "t.db"
    conn = connect(str(db_path))
    init_schema(conn)
    with transaction(conn):
        conn.execute("INSERT INTO meta (key, value) VALUES ('k', 'v')")
    row = conn.execute("SELECT value FROM meta WHERE key='k'").fetchone()
    assert row[0] == "v"

def test_transaction_rolls_back_on_exception(tmp_path):
    from app.db import transaction
    db_path = tmp_path / "t.db"
    conn = connect(str(db_path))
    init_schema(conn)
    try:
        with transaction(conn):
            conn.execute("INSERT INTO meta (key, value) VALUES ('k', 'v')")
            raise RuntimeError("boom")
    except RuntimeError:
        pass
    row = conn.execute("SELECT value FROM meta WHERE key='k'").fetchone()
    assert row is None

def test_standalone_dml_visible_to_second_connection(tmp_path):
    """Autocommit means a standalone INSERT is visible across connections
    without an explicit commit. Regression guard for the isolation_level
    footgun: if connect() is changed to default isolation_level, this test
    fails because the second connection sees an empty meta table."""
    db_path = str(tmp_path / "t.db")
    c1 = connect(db_path)
    init_schema(c1)
    c1.execute("INSERT INTO meta (key, value) VALUES ('k', 'v')")
    c2 = connect(db_path)
    row = c2.execute("SELECT value FROM meta WHERE key='k'").fetchone()
    assert row is not None and row[0] == "v"

def test_transaction_does_not_collide_with_dml(tmp_path):
    """The transaction() context manager must work right after standalone
    DML on the same connection. Regression guard for 'cannot start a
    transaction within a transaction'."""
    from app.db import transaction
    db_path = str(tmp_path / "t.db")
    conn = connect(db_path)
    init_schema(conn)
    conn.execute("INSERT INTO meta (key, value) VALUES ('a', '1')")
    # If isolation_level is not None, this BEGIN will raise OperationalError.
    with transaction(conn):
        conn.execute("INSERT INTO meta (key, value) VALUES ('b', '2')")
    assert conn.execute("SELECT COUNT(*) FROM meta").fetchone()[0] >= 2

def test_init_schema_is_idempotent(tmp_path):
    db_path = tmp_path / "test.db"
    conn = connect(str(db_path))
    init_schema(conn)
    init_schema(conn)  # should not raise
    cur = conn.execute("SELECT value FROM meta WHERE key='schema_version'")
    assert cur.fetchone()[0] == str(SCHEMA_VERSION)

def test_wal_mode_enabled(tmp_path):
    db_path = tmp_path / "test.db"
    conn = connect(str(db_path))
    init_schema(conn)
    cur = conn.execute("PRAGMA journal_mode")
    assert cur.fetchone()[0].lower() == "wal"
```

- [ ] **Step 2: Run, verify it fails**

```bash
pytest tests/test_db.py -v
```

Expected: ImportError for `app.db`.

- [ ] **Step 3: Implement `app/db.py`**

```python
from __future__ import annotations
import sqlite3
from contextlib import contextmanager
from pathlib import Path

SCHEMA_VERSION = 1

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS subscriptions (
  id                INTEGER PRIMARY KEY AUTOINCREMENT,
  email             TEXT NOT NULL,
  city              TEXT NOT NULL DEFAULT 'leipzig',
  language          TEXT NOT NULL DEFAULT 'de',
  filters_json      TEXT NOT NULL,
  created_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  confirmed_at      TIMESTAMP,
  last_notified_at  TIMESTAMP,
  expires_at        TIMESTAMP NOT NULL,
  reminder_sent_at  TIMESTAMP,
  heartbeat_30d_at  TIMESTAMP,
  heartbeat_60d_at  TIMESTAMP,
  deleted_at        TIMESTAMP
);
CREATE INDEX IF NOT EXISTS idx_active_subs
  ON subscriptions(deleted_at, confirmed_at, expires_at, city);

CREATE TABLE IF NOT EXISTS seen_slots (
  subscription_id INTEGER NOT NULL,
  slot_hash       TEXT NOT NULL,
  sent_at         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (subscription_id, slot_hash),
  FOREIGN KEY (subscription_id) REFERENCES subscriptions(id) ON DELETE CASCADE
);
CREATE INDEX IF NOT EXISTS idx_seen_sent_at ON seen_slots(sent_at);

CREATE TABLE IF NOT EXISTS sent_idempotency (
  idem_key  TEXT PRIMARY KEY,
  provider  TEXT NOT NULL,
  sent_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS idx_sent_idem_at ON sent_idempotency(sent_at);

CREATE TABLE IF NOT EXISTS meta (
  key        TEXT PRIMARY KEY,
  value      TEXT NOT NULL,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Per-city operational state. Typed columns instead of LIKE'd meta keys
-- so that city typos cannot create ghost rows that never alert.
CREATE TABLE IF NOT EXISTS city_state (
  city                  TEXT PRIMARY KEY,
  zero_match_since      TIMESTAMP,      -- NULL when last cycle had a slot
  last_canary_alert_at  TIMESTAMP,
  requests_today        INTEGER NOT NULL DEFAULT 0,
  last_polled_at        TIMESTAMP
);

-- Short-lived cache of slots we've handed out booking links for.
-- /go/<slot_token> reads this to know which city's upstream to redirect to.
-- Each new city's scraper writes here when it discovers a slot.
CREATE TABLE IF NOT EXISTS slots_cache (
  slot_token   TEXT PRIMARY KEY,
  city         TEXT NOT NULL,
  upstream_url TEXT NOT NULL,            -- pre-rendered redirect target
  cached_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS idx_slots_cache_at ON slots_cache(cached_at);
"""

def connect(db_path: str) -> sqlite3.Connection:
    Path(db_path).parent.mkdir(parents=True, exist_ok=True)
    # `isolation_level=None` = autocommit mode. Without this, Python's sqlite3
    # module opens implicit BEGINs before DML statements and never closes
    # them — which then collides with the explicit BEGIN issued by the
    # `transaction()` context manager (raises "cannot start a transaction
    # within a transaction"). Autocommit + explicit transactions where needed
    # is the correct pairing.
    conn = sqlite3.connect(db_path, isolation_level=None)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA foreign_keys=ON")
    return conn

@contextmanager
def transaction(conn: sqlite3.Connection):
    """Atomic BEGIN…COMMIT (or ROLLBACK on exception).

    Requires the connection to be in autocommit mode (`isolation_level=None`),
    which `connect()` above sets. Outside this context manager, every
    statement is its own transaction.
    """
    conn.execute("BEGIN")
    try:
        yield conn
    except BaseException:
        conn.execute("ROLLBACK")
        raise
    else:
        conn.execute("COMMIT")

def init_schema(conn: sqlite3.Connection) -> None:
    conn.executescript(SCHEMA_SQL)
    conn.execute(
        "INSERT INTO meta (key, value) VALUES ('schema_version', ?) "
        "ON CONFLICT (key) DO UPDATE SET value=excluded.value, "
        "updated_at=CURRENT_TIMESTAMP",
        (str(SCHEMA_VERSION),),
    )
```

- [ ] **Step 4: Run tests, verify pass**

```bash
pytest tests/test_db.py -v
```

Expected: 7 PASS (schema-tables, idempotent-init, WAL-mode, transaction-commit, transaction-rollback, cross-connection-visibility, transaction-no-collide).

- [ ] **Step 5: Commit**

```bash
git add app/db.py tests/test_db.py
git commit -m "feat(db): schema bootstrap with WAL mode and idempotent init"
```

---

### Task 1.3: Models (dataclasses)

**Files:**
- Create: `tests/test_models.py`
- Create: `app/models.py`

- [ ] **Step 1: Write the failing test**

`tests/test_models.py`:

```python
from datetime import datetime, time
from app.models import Subscription, Slot, PollPlan, Filter

def test_filter_from_json():
    f = Filter.from_json('{"appointment_types": ["uuid-a"], "locations": "all", '
                         '"weekdays": [1,2,3,4,5], "time_window": {"start":"08:00","end":"18:00"}}')
    assert f.appointment_types == ["uuid-a"]
    assert f.locations == "all"
    assert f.weekdays == [1, 2, 3, 4, 5]
    assert f.time_window_start == time(8, 0)
    assert f.time_window_end == time(18, 0)

def test_filter_to_json_roundtrip():
    f = Filter(
        appointment_types=["a", "b"],
        locations=["loc-1"],
        weekdays=[1, 7],
        time_window_start=time(9, 0),
        time_window_end=time(17, 30),
    )
    s = f.to_json()
    f2 = Filter.from_json(s)
    assert f2.appointment_types == f.appointment_types
    assert f2.locations == f.locations
    assert f2.weekdays == f.weekdays
    assert f2.time_window_start == f.time_window_start
    assert f2.time_window_end == f.time_window_end

def test_slot_hash_is_deterministic():
    s1 = Slot(
        date="2026-06-10", time_str="10:30",
        location_uuid="loc-1", service_uuid="svc-1",
        booking_token="abc",
    )
    s2 = Slot(
        date="2026-06-10", time_str="10:30",
        location_uuid="loc-1", service_uuid="svc-1",
        booking_token="def",  # different token, same logical slot
    )
    assert s1.hash() == s2.hash()
    s3 = Slot(date="2026-06-11", time_str="10:30",
              location_uuid="loc-1", service_uuid="svc-1",
              booking_token="abc")
    assert s1.hash() != s3.hash()
```

- [ ] **Step 2: Run, verify it fails**

```bash
pytest tests/test_models.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement `app/models.py`**

```python
from __future__ import annotations
import hashlib
import json
from dataclasses import dataclass, field
from datetime import datetime, time
from typing import Union

LocationsSpec = Union[list[str], str]  # list of UUIDs, or "all"

@dataclass(frozen=True)
class Filter:
    appointment_types: list[str]
    locations: LocationsSpec
    weekdays: list[int]                # ISO 8601: 1=Mon … 7=Sun
    time_window_start: time
    time_window_end: time

    def to_json(self) -> str:
        return json.dumps({
            "appointment_types": list(self.appointment_types),
            "locations": self.locations if self.locations == "all"
                         else list(self.locations),
            "weekdays": list(self.weekdays),
            "time_window": {
                "start": self.time_window_start.strftime("%H:%M"),
                "end":   self.time_window_end.strftime("%H:%M"),
            },
        })

    @classmethod
    def from_json(cls, s: str) -> "Filter":
        d = json.loads(s)
        tw = d.get("time_window", {"start": "00:00", "end": "23:59"})
        return cls(
            appointment_types=list(d["appointment_types"]),
            locations="all" if d["locations"] == "all" else list(d["locations"]),
            weekdays=list(d.get("weekdays", [1, 2, 3, 4, 5, 6, 7])),
            time_window_start=_parse_hhmm(tw["start"]),
            time_window_end=_parse_hhmm(tw["end"]),
        )

def _parse_hhmm(s: str) -> time:
    hh, mm = s.split(":")
    return time(int(hh), int(mm))

@dataclass(frozen=True)
class Slot:
    date: str          # YYYY-MM-DD
    time_str: str      # HH:MM
    location_uuid: str
    service_uuid: str
    booking_token: str # opaque, session-bound — excluded from hash

    def hash(self) -> str:
        payload = f"{self.date}|{self.time_str}|{self.location_uuid}|{self.service_uuid}"
        return hashlib.sha256(payload.encode("utf-8")).hexdigest()

@dataclass(frozen=True)
class Subscription:
    id: int
    email: str
    city: str
    language: str       # 'de' or 'en'
    sub_filter: Filter  # named 'sub_filter' to avoid shadowing builtin filter()
    created_at: datetime
    confirmed_at: datetime | None
    last_notified_at: datetime | None
    expires_at: datetime
    reminder_sent_at: datetime | None
    heartbeat_30d_at: datetime | None
    heartbeat_60d_at: datetime | None
    deleted_at: datetime | None

@dataclass(frozen=True)
class PollPlan:
    """A polling unit shared across subscriptions with the same scrape needs."""
    city: str
    appointment_type: str
    locations: LocationsSpec  # list of UUIDs OR "all"

    def key(self) -> str:
        if self.locations == "all":
            locs = "all"
        else:
            locs = ",".join(sorted(self.locations))
        return f"{self.city}|{self.appointment_type}|{locs}"
```

- [ ] **Step 4: Run, verify pass**

```bash
pytest tests/test_models.py -v
```

Expected: 3 PASS.

- [ ] **Step 5: Commit**

```bash
git add app/models.py tests/test_models.py
git commit -m "feat(models): Filter / Slot / Subscription / PollPlan dataclasses"
```

---

## Phase 2: Tokens & Catalog

### Task 2.1: Dual-secret HMAC tokens

**Files:**
- Create: `tests/test_tokens.py`
- Create: `app/tokens.py`

- [ ] **Step 1: Write the failing test**

`tests/test_tokens.py`:

```python
import pytest
from app.tokens import sign, verify, InvalidToken

SECRET_A = "a" * 32
SECRET_B = "b" * 32

def test_sign_and_verify_with_primary():
    tok = sign(123, "confirm", primary=SECRET_A, previous="")
    assert verify(tok, "confirm", primary=SECRET_A, previous="") == 123

def test_verify_fails_on_wrong_purpose():
    tok = sign(123, "confirm", primary=SECRET_A, previous="")
    with pytest.raises(InvalidToken):
        verify(tok, "unsubscribe", primary=SECRET_A, previous="")

def test_verify_with_rotated_secret():
    """A token signed with the old secret still verifies after rotation."""
    tok = sign(7, "manage", primary=SECRET_A, previous="")
    # Rotation: old primary becomes previous, new primary set
    assert verify(tok, "manage", primary=SECRET_B, previous=SECRET_A) == 7

def test_verify_fails_after_second_rotation():
    """A token signed with the original secret invalidates after two rotations."""
    tok = sign(7, "manage", primary=SECRET_A, previous="")
    # After two rotations: previous now holds the FIRST rotation's primary,
    # which is not SECRET_A.
    with pytest.raises(InvalidToken):
        verify(tok, "manage", primary="c" * 32, previous=SECRET_B)

def test_verify_fails_on_tampered_token():
    tok = sign(123, "confirm", primary=SECRET_A, previous="")
    tampered = tok[:-1] + ("A" if tok[-1] != "A" else "B")
    with pytest.raises(InvalidToken):
        verify(tampered, "confirm", primary=SECRET_A, previous="")
```

- [ ] **Step 2: Run, verify it fails**

```bash
pytest tests/test_tokens.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement `app/tokens.py`**

```python
from __future__ import annotations
import base64
import hashlib
import hmac

TOKEN_VERSION = 1  # bump on any change to the signed payload format

class InvalidToken(Exception):
    pass

def _b64u(b: bytes) -> str:
    return base64.urlsafe_b64encode(b).rstrip(b"=").decode("ascii")

def _b64u_decode(s: str) -> bytes:
    pad = "=" * (-len(s) % 4)
    return base64.urlsafe_b64decode(s + pad)

def _hmac(secret: str, payload: bytes) -> bytes:
    return hmac.new(secret.encode("utf-8"), payload, hashlib.sha256).digest()

def _payload(subscription_id: int, purpose: str) -> bytes:
    return f"{TOKEN_VERSION}:{subscription_id}:{purpose}".encode("utf-8")

def sign(subscription_id: int, purpose: str, *, primary: str, previous: str) -> str:
    sig = _hmac(primary, _payload(subscription_id, purpose))
    return f"{TOKEN_VERSION}.{subscription_id}.{purpose}.{_b64u(sig)}"

def verify(token: str, purpose: str, *, primary: str, previous: str) -> int:
    try:
        ver_str, sub_id_str, tok_purpose, sig_b64 = token.split(".", 3)
    except ValueError:
        raise InvalidToken("malformed token")
    try:
        version = int(ver_str)
    except ValueError:
        raise InvalidToken("non-integer version")
    if version != TOKEN_VERSION:
        raise InvalidToken(f"unsupported token version {version}")
    if tok_purpose != purpose:
        raise InvalidToken("purpose mismatch")
    try:
        sub_id = int(sub_id_str)
    except ValueError:
        raise InvalidToken("non-integer subscription id")
    payload = _payload(sub_id, purpose)
    try:
        sig = _b64u_decode(sig_b64)
    except Exception:
        raise InvalidToken("bad signature encoding")
    for secret in (primary, previous):
        if not secret:
            continue
        expected = _hmac(secret, payload)
        if hmac.compare_digest(sig, expected):
            return sub_id
    raise InvalidToken("signature does not match")
```

- [ ] **Step 4: Run, verify pass**

```bash
pytest tests/test_tokens.py -v
```

Expected: 5 PASS.

- [ ] **Step 5: Commit**

```bash
git add app/tokens.py tests/test_tokens.py
git commit -m "feat(tokens): dual-secret HMAC sign/verify with rotation support"
```

---

### Task 2.2: Catalog loader

**Files:**
- Create: `tests/test_catalog.py`
- Create: `app/catalog.py`
- Create: `catalog/leipzig/appointment_type.json` (copy from existing repo)
- Create: `catalog/leipzig/locations.json` (copy from existing repo)

- [ ] **Step 1: Copy the catalog JSON files from the existing repo and create scraper_config.json**

```bash
mkdir -p catalog/leipzig
cp ../leipzigappointmentsbotpremium/appointment_type.json catalog/leipzig/
cp ../leipzigappointmentsbotpremium/locations.json catalog/leipzig/
```

Then create `catalog/leipzig/scraper_config.json`:

```json
{
  "vendor": "smartcjm",
  "base_url": "https://terminvereinbarung.leipzig.de/m/leipzig-ba/extern/calendar",
  "uid": "b76cab25-49bd-44e3-950d-aab715881ea7",
  "steps": "serviceslocationssearch_resultsbookingfinish"
}
```

Per-city values let the same `smartcjm.py` scraper handle multiple Smart-CJM
tenants (e.g., Köln, which uses the same vendor with a different `uid` and
`base_url`). Adding a Smart-CJM city is then catalog-only — no Python edit.

- [ ] **Step 2: Write the failing test**

`tests/test_catalog.py`:

```python
import pytest
from app.catalog import load_catalog, CatalogError, Catalog

def test_load_leipzig_catalog():
    cat = load_catalog("leipzig")
    assert isinstance(cat, Catalog)
    assert len(cat.appointment_types) > 0
    assert len(cat.locations) > 0
    # appointment_types and locations are name → uuid maps
    sample_name, sample_uuid = next(iter(cat.appointment_types.items()))
    assert isinstance(sample_name, str)
    assert len(sample_uuid) == 36  # UUID

def test_load_unknown_city_raises():
    with pytest.raises(CatalogError):
        load_catalog("atlantis")

def test_catalog_lookup_helpers():
    cat = load_catalog("leipzig")
    name = next(iter(cat.appointment_types.keys()))
    uuid = cat.appointment_types[name]
    assert cat.appointment_type_name_for(uuid) == name
    assert cat.appointment_type_uuid_for(name) == uuid
```

- [ ] **Step 3: Run, verify fail**

```bash
pytest tests/test_catalog.py -v
```

Expected: ImportError.

- [ ] **Step 4: Implement `app/catalog.py`**

```python
from __future__ import annotations
import json
from dataclasses import dataclass
from functools import lru_cache
from pathlib import Path

CATALOG_ROOT = Path(__file__).parent.parent / "catalog"

class CatalogError(Exception):
    pass

@dataclass(frozen=True)
class Catalog:
    city: str
    appointment_types: dict[str, str]  # name → uuid
    locations: dict[str, str]          # name → uuid
    scraper_config: dict               # vendor-specific, opaque to web layer

    def appointment_type_name_for(self, uuid: str) -> str | None:
        return next((n for n, u in self.appointment_types.items() if u == uuid), None)

    def location_name_for(self, uuid: str) -> str | None:
        return next((n for n, u in self.locations.items() if u == uuid), None)

    def appointment_type_uuid_for(self, name: str) -> str | None:
        return self.appointment_types.get(name)

    def location_uuid_for(self, name: str) -> str | None:
        return self.locations.get(name)

@lru_cache(maxsize=8)
def load_catalog(city: str) -> Catalog:
    city_dir = CATALOG_ROOT / city
    if not city_dir.is_dir():
        raise CatalogError(f"Unknown city: {city}")
    try:
        ats = json.loads((city_dir / "appointment_type.json").read_text(encoding="utf-8"))
        locs = json.loads((city_dir / "locations.json").read_text(encoding="utf-8"))
        scfg = json.loads((city_dir / "scraper_config.json").read_text(encoding="utf-8"))
    except FileNotFoundError as exc:
        raise CatalogError(f"Missing catalog file for {city}: {exc.filename}") from exc
    return Catalog(city=city, appointment_types=ats, locations=locs,
                   scraper_config=scfg)
```

- [ ] **Step 5: Run, verify pass**

```bash
pytest tests/test_catalog.py -v
```

Expected: 3 PASS.

- [ ] **Step 6: Commit**

```bash
git add app/catalog.py catalog/ tests/test_catalog.py
git commit -m "feat(catalog): per-city JSON catalog loader with name/uuid lookups"
```

---

## Phase 3: Filter Matching

### Task 3.1: Filter-to-slot matching

**Files:**
- Create: `tests/test_filters.py`
- Create: `app/filters.py`

- [ ] **Step 1: Write the failing test**

`tests/test_filters.py`:

```python
from datetime import time, date
from app.models import Filter, Slot
from app.filters import matches

def make_slot(date_str="2026-06-10", time_str="10:30", loc="loc-1", svc="svc-A"):
    return Slot(date=date_str, time_str=time_str, location_uuid=loc,
                service_uuid=svc, booking_token="t")

def make_filter(types=("svc-A",), locations=("loc-1",), weekdays=(1,2,3,4,5),
                start=time(0,0), end=time(23,59)):
    return Filter(
        appointment_types=list(types),
        locations=list(locations),
        weekdays=list(weekdays),
        time_window_start=start,
        time_window_end=end,
    )

def test_match_basic():
    assert matches(make_filter(), make_slot()) is True

def test_no_match_wrong_service():
    assert matches(make_filter(types=("svc-A",)), make_slot(svc="svc-B")) is False

def test_no_match_wrong_location():
    assert matches(make_filter(locations=("loc-1",)), make_slot(loc="loc-2")) is False

def test_match_locations_all():
    f = make_filter(locations=())
    f = Filter(
        appointment_types=["svc-A"],
        locations="all",
        weekdays=[1,2,3,4,5,6,7],
        time_window_start=time(0,0),
        time_window_end=time(23,59),
    )
    assert matches(f, make_slot(loc="loc-anywhere")) is True

def test_no_match_wrong_weekday():
    # 2026-06-13 is a Saturday (ISO weekday 6)
    f = make_filter(weekdays=(1,2,3,4,5))
    assert matches(f, make_slot(date_str="2026-06-13")) is False

def test_match_time_window():
    f = make_filter(start=time(9,0), end=time(17,0))
    assert matches(f, make_slot(time_str="09:00")) is True
    assert matches(f, make_slot(time_str="17:00")) is True
    assert matches(f, make_slot(time_str="08:59")) is False
    assert matches(f, make_slot(time_str="17:01")) is False

def test_invalid_date_string_does_not_match():
    f = make_filter()
    assert matches(f, make_slot(date_str="not-a-date")) is False
```

- [ ] **Step 2: Run, verify fail**

```bash
pytest tests/test_filters.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement `app/filters.py`**

```python
from __future__ import annotations
from datetime import date, time
from app.models import Filter, Slot

def matches(f: Filter, slot: Slot) -> bool:
    if slot.service_uuid not in f.appointment_types:
        return False
    if f.locations != "all" and slot.location_uuid not in f.locations:
        return False
    try:
        d = date.fromisoformat(slot.date)
    except ValueError:
        return False
    if d.isoweekday() not in f.weekdays:
        return False
    try:
        hh, mm = slot.time_str.split(":")
        t = time(int(hh), int(mm))
    except (ValueError, IndexError):
        return False
    if t < f.time_window_start or t > f.time_window_end:
        return False
    return True
```

- [ ] **Step 4: Run, verify pass**

```bash
pytest tests/test_filters.py -v
```

Expected: 7 PASS.

- [ ] **Step 5: Commit**

```bash
git add app/filters.py tests/test_filters.py
git commit -m "feat(filters): per-subscription filter matching with weekday & time-of-day"
```

---

## Phase 4: Smart-CJM Scraper

### Task 4.1: Capture a golden HTML fixture

**Files:**
- Create: `tests/fixtures/leipzig_with_slots.html`
- Create: `tests/fixtures/leipzig_no_slots.html`
- Create: `tests/fixtures/leipzig_session_expired.html`

- [ ] **Step 1: Manually capture the fixtures**

Visit `https://terminvereinbarung.leipzig.de/m/leipzig-ba/extern/calendar/search_result?search_mode=earliest&uid=b76cab25-49bd-44e3-950d-aab715881ea7` in a real browser, walk through the booking flow to a search-result page that DOES show slots, save the HTML as `tests/fixtures/leipzig_with_slots.html`.

Then do the same on a page with no slots (e.g. extremely-narrow filter) → `leipzig_no_slots.html`.

Force a session expiry by waiting/manipulating cookies → `leipzig_session_expired.html` (the page should contain the literal string "Session abgelaufen").

These fixtures are committed to the repo as test data.

- [ ] **Step 2: Commit fixtures**

```bash
git add tests/fixtures/
git commit -m "test: capture Leipzig HTML fixtures for parser regression tests"
```

---

### Task 4.2: Smart-CJM HTML parser

**Files:**
- Create: `tests/test_smartcjm_parser.py`
- Create: `app/scrapers/__init__.py`
- Create: `app/scrapers/smartcjm.py`

- [ ] **Step 1: Write the failing test**

`tests/test_smartcjm_parser.py`:

```python
from pathlib import Path
from app.scrapers.smartcjm import parse_slots

FIXTURES = Path(__file__).parent / "fixtures"

def test_parse_with_slots():
    html = (FIXTURES / "leipzig_with_slots.html").read_text(encoding="utf-8")
    slots = parse_slots(html)
    assert len(slots) > 0
    s = slots[0]
    assert s.date  # ISO date
    assert ":" in s.time_str
    assert s.booking_token  # opaque

def test_parse_no_slots():
    html = (FIXTURES / "leipzig_no_slots.html").read_text(encoding="utf-8")
    assert parse_slots(html) == []

def test_session_expired_returns_empty():
    html = (FIXTURES / "leipzig_session_expired.html").read_text(encoding="utf-8")
    assert parse_slots(html) == []
```

- [ ] **Step 2: Run, verify fail**

```bash
pytest tests/test_smartcjm_parser.py -v
```

Expected: ImportError.

- [ ] **Step 3: Create `app/scrapers/__init__.py`** (empty for now)

```bash
touch app/scrapers/__init__.py
```

- [ ] **Step 4: Implement `app/scrapers/smartcjm.py`** (based on the ACTUAL Leipzig HTML structure as of 2026-05-17, captured into the fixtures in Task 4.1)

> **NOTE**: The existing single-user `leipzigappointmentsbot.py` parser is
> based on an older HTML layout ("Termin Uhrzeiten" headings + free-form
> `<li>` elements) that Stadt Leipzig has since replaced. The current page
> uses stable `data-testid` attributes designed for automation. The parser
> below targets the current layout. The legacy regex-against-prose approach
> is NOT used.
>
> Each slot in the current Smart-CJM HTML looks like:
> ```html
> <li data-testid="slot_button_li-N">
>   <button ... onclick="return appointment_reserve(
>       '2026-05-20T09%3a15%3a00%2b02%3a00',  // URL-encoded ISO datetime
>       '10',                                  // duration in minutes
>       '<location_uuid>',
>       '<service_uuid>');" ...>
>     <strong data-slot-from="2026-05-20T09:15:00.0000000+02:00">…</strong>
>   </button>
> </li>
> ```

```python
from __future__ import annotations
import re
import urllib.parse
from bs4 import BeautifulSoup
from app.models import Slot

# Captures the four arguments of appointment_reserve(...).
# Arg 0: URL-encoded ISO datetime, e.g. "2026-05-20T09%3a15%3a00%2b02%3a00"
# Arg 1: duration in minutes (ignored)
# Arg 2: location UUID
# Arg 3: service UUID
APPOINTMENT_RESERVE_RE = re.compile(
    r"appointment_reserve\(\s*"
    r"'([^']+)'\s*,\s*"   # encoded datetime
    r"'(\d+)'\s*,\s*"     # duration minutes
    r"'([^']+)'\s*,\s*"   # location uuid
    r"'([^']+)'\s*\)"     # service uuid
)
SLOT_LI_TESTID_RE = re.compile(r"^slot_button_li-\d+$")

def parse_slots(html: str) -> list[Slot]:
    """Parse Smart-CJM search-result HTML into Slot records.

    Returns an empty list when the response is a session-expired page or
    when no slot_button_li-N elements are present.
    """
    if "Session abgelaufen" in html:
        return []
    soup = BeautifulSoup(html, "html.parser")
    slots: list[Slot] = []
    for li in soup.find_all("li", attrs={"data-testid": SLOT_LI_TESTID_RE}):
        btn = li.find("button")
        if not btn:
            continue
        onclick = btn.get("onclick", "")
        m = APPOINTMENT_RESERVE_RE.search(onclick)
        if not m:
            continue
        encoded_dt, _duration, location_uuid, service_uuid = m.groups()
        dt = urllib.parse.unquote(encoded_dt)
        # Expected format: "2026-05-20T09:15:00+02:00"
        if "T" not in dt:
            continue
        date_part, time_part = dt.split("T", 1)
        time_str = time_part[:5]  # "HH:MM"
        slots.append(Slot(
            date=date_part,
            time_str=time_str,
            location_uuid=location_uuid,
            service_uuid=service_uuid,
            # The token is the encoded datetime — preserved for upstream
            # redirect composition. NOT used in Slot.hash() (hash uses
            # date|time|location|service for stable dedup).
            booking_token=encoded_dt,
        ))
    return slots
```

- [ ] **Step 5: Run, verify pass**

```bash
pytest tests/test_smartcjm_parser.py -v
```

Expected: 3 PASS. If `test_parse_with_slots` fails because the regex/structure assumptions don't match the actual fixture, examine the fixture HTML and adjust the parser. (This is the riskiest single task in the plan — the existing bot's parser is fragile.)

- [ ] **Step 6: Commit**

```bash
git add app/scrapers/ tests/test_smartcjm_parser.py
git commit -m "feat(scrapers): Smart-CJM HTML parser ported from single-user bot"
```

---

### Task 4.3: Smart-CJM polling client (live request flow)

**Files:**
- Modify: `app/scrapers/smartcjm.py`
- Create: `tests/test_smartcjm_client.py`

- [ ] **Step 1: Write the failing test using mocked HTTP**

`tests/test_smartcjm_client.py`:

```python
from unittest.mock import patch, MagicMock
from app.models import PollPlan
from app.scrapers.smartcjm import poll

LEIPZIG_BASE = "https://terminvereinbarung.leipzig.de/m/leipzig-ba/extern/calendar"

def _mock_session(redirect_url: str, services_html: str, locations_html: str):
    sess = MagicMock()
    # Each .get / .post returns an object with .url and .text
    sess.get.return_value = MagicMock(url=redirect_url, text="", status_code=200)
    sess.post.side_effect = [
        MagicMock(text=services_html, status_code=200, url=""),
        MagicMock(text=locations_html, status_code=200, url=""),
    ]
    return sess

def test_poll_returns_slots_from_locations_response():
    plan = PollPlan(city="leipzig",
                    appointment_type="29cd0a26-fe7a-4d65-88cd-1e05fd749c71",
                    locations="all")
    redirect = f"{LEIPZIG_BASE}/?wsid=fake-wsid&uid=b76cab25"
    locations_html = (
        "<h3>Termin Uhrzeiten am Montag, 2026-06-10</h3>"
        "<ul><li>10:30 <a onclick=\"appointment_reserve('loc-1|svc-1')\">x</a></li></ul>"
    )
    sess = _mock_session(redirect_url=redirect,
                         services_html="",
                         locations_html=locations_html)
    slots = poll(plan, http=sess)
    assert len(slots) == 1
    assert slots[0].date == "2026-06-10"
    assert slots[0].time_str == "10:30"

def test_poll_session_expired_returns_empty():
    plan = PollPlan(city="leipzig", appointment_type="x", locations="all")
    redirect = f"{LEIPZIG_BASE}/?wsid=fake&uid=b76cab25"
    sess = _mock_session(redirect_url=redirect,
                         services_html="",
                         locations_html="Session abgelaufen")
    assert poll(plan, http=sess) == []
```

- [ ] **Step 2: Run, verify fail**

```bash
pytest tests/test_smartcjm_client.py -v
```

Expected: ImportError for `poll`.

- [ ] **Step 3: Extend `app/scrapers/smartcjm.py`** by appending:

```python
import requests
from app.models import PollPlan
from app.catalog import load_catalog

def _acquire_wsid(http: requests.Session, base_url: str, uid: str) -> str:
    r = http.get(
        f"{base_url}/search_result?search_mode=earliest&uid={uid}",
        timeout=30, allow_redirects=True,
    )
    if "wsid=" not in r.url:
        raise RuntimeError("wsid not found in redirect URL")
    return r.url.split("wsid=", 1)[1].split("&", 1)[0]

def _post_services(http: requests.Session, wsid: str, plan: PollPlan,
                   catalog, scfg: dict) -> None:
    parts = []
    for code in catalog.appointment_types.values():
        amount = "1" if code == plan.appointment_type else ""
        parts.append(f"services={code}")
        parts.append(f"service_{code}_amount={amount}")
    body = (
        f"action_type=&steps={scfg['steps']}&"
        "step_current=services&step_current_index=0&step_goto=%2B1&services=&"
        + "&".join(parts)
    )
    http.post(
        f"{scfg['base_url']}/?uid={scfg['uid']}&wsid={wsid}&lang=de&rev=HL0Ur#top",
        headers={"Content-Type": "application/x-www-form-urlencoded"},
        data=body, timeout=30,
    )

def _post_locations(http: requests.Session, wsid: str, plan: PollPlan,
                    catalog, scfg: dict) -> str:
    if plan.locations == "all":
        locations_all = "1"
        loc_uuids = list(catalog.locations.values())
    else:
        locations_all = ""
        loc_uuids = list(plan.locations)
    loc_parts = "&".join(f"locations={u}" for u in loc_uuids)
    body = (
        f"action_type=search&steps={scfg['steps']}&"
        "step_current=locations&step_current_index=1&step_goto=%2B1&"
        f"locations_selected_all={locations_all}&{loc_parts}"
    )
    r = http.post(
        f"{scfg['base_url']}/?uid={scfg['uid']}&wsid={wsid}&lang=de&",
        headers={"Content-Type": "application/x-www-form-urlencoded"},
        data=body, timeout=30,
    )
    return r.text

def poll(plan: PollPlan, http: requests.Session) -> list[Slot]:
    """Run the 3-step Smart-CJM flow against the city's tenant. Returns parsed slots."""
    catalog = load_catalog(plan.city)
    scfg = catalog.scraper_config
    if scfg.get("vendor") != "smartcjm":
        raise RuntimeError(
            f"city {plan.city} not configured for smartcjm scraper "
            f"(vendor={scfg.get('vendor')})"
        )
    wsid = _acquire_wsid(http, scfg["base_url"], scfg["uid"])
    _post_services(http, wsid, plan, catalog, scfg)
    html = _post_locations(http, wsid, plan, catalog, scfg)
    return parse_slots(html)
```

- [ ] **Step 4: Run, verify pass**

```bash
pytest tests/test_smartcjm_client.py -v
```

Expected: 2 PASS. (Note: `_post_services` is invoked but its return value isn't tested directly — the second-element of `side_effect` is `_post_locations`'s response.)

- [ ] **Step 5: Commit**

```bash
git add app/scrapers/smartcjm.py tests/test_smartcjm_client.py
git commit -m "feat(scrapers): Smart-CJM client (3-step flow with fresh wsid per cycle)"
```

---

### Task 4.4: Scraper dispatcher

**Files:**
- Modify: `app/scrapers/__init__.py`
- Create: `tests/test_scraper_dispatch.py`

- [ ] **Step 1: Write the failing test**

```python
import pytest
from app.scrapers import get_scraper, UnsupportedCity

def test_get_scraper_leipzig():
    scraper = get_scraper("leipzig")
    assert hasattr(scraper, "poll")

def test_get_scraper_unknown():
    with pytest.raises(UnsupportedCity):
        get_scraper("atlantis")
```

- [ ] **Step 2: Run, verify fail**

- [ ] **Step 3: Implement** `app/scrapers/__init__.py`:

```python
"""City scraper registry.

EACH SCRAPER MODULE MUST EXPOSE A MODULE-LEVEL FUNCTION:

    def poll(plan: PollPlan, http: requests.Session) -> list[Slot]: ...

This is the only contract. Modules MUST NOT define classes or expect to
be instantiated — `get_scraper(city)` returns the module itself, and the
caller invokes `module.poll(plan, http=http)`. When adding a new city
(e.g., Hamburg ODControls), create `app/scrapers/<vendor>.py` with this
free function signature, then add an entry to `_REGISTRY` below.
"""
from __future__ import annotations
from types import ModuleType
from typing import Protocol
import requests
from app.models import PollPlan, Slot
from app.scrapers import smartcjm

class ScraperProtocol(Protocol):
    """Structural type used for documentation / mypy. Not enforced at runtime."""
    def poll(self, plan: PollPlan, http: requests.Session) -> list[Slot]: ...

class UnsupportedCity(Exception):
    pass

_REGISTRY: dict[str, ModuleType] = {
    "leipzig": smartcjm,
    # When adding Hamburg, etc.:
    #   from app.scrapers import odcontrols
    #   "hamburg": odcontrols,
}

def get_scraper(city: str) -> ModuleType:
    """Return the scraper module for `city`. The module's `poll(plan, http)`
    is the only attribute the caller may rely on."""
    if city not in _REGISTRY:
        raise UnsupportedCity(city)
    return _REGISTRY[city]
```

- [ ] **Step 4: Run, verify pass**

- [ ] **Step 5: Commit**

```bash
git add app/scrapers/__init__.py tests/test_scraper_dispatch.py
git commit -m "feat(scrapers): city-to-scraper dispatcher with registry"
```

---

## Phase 5: Mail Layer

### Task 5.1: Mail sender interface with provider failover and idempotency

**Files:**
- Create: `tests/test_mail.py`
- Create: `app/mail.py`

- [ ] **Step 1: Write the failing test**

`tests/test_mail.py`:

```python
import sqlite3
from unittest.mock import patch, MagicMock
import pytest
from app.db import connect, init_schema
from app.mail import send, MailFailed, _idem_key

@pytest.fixture
def db(tmp_path):
    conn = connect(str(tmp_path / "t.db"))
    init_schema(conn)
    return conn

def _ok():
    r = MagicMock()
    r.status_code = 200
    return r

def _resp(code):
    r = MagicMock()
    r.status_code = code
    return r

def test_send_uses_mailjet_when_ok(db):
    with patch("app.mail._call_mailjet", return_value=_ok()) as mj, \
         patch("app.mail._call_resend") as re_:
        send(db, "alice@example.com", "subj", "body", idem_key="k1")
    mj.assert_called_once()
    re_.assert_not_called()
    row = db.execute("SELECT provider FROM sent_idempotency WHERE idem_key='k1'").fetchone()
    assert row["provider"] == "mailjet"

def test_failover_to_resend_on_mailjet_5xx(db):
    with patch("app.mail._call_mailjet", return_value=_resp(503)), \
         patch("app.mail._call_resend", return_value=_ok()) as re_:
        send(db, "alice@example.com", "subj", "body", idem_key="k2")
    re_.assert_called_once()
    row = db.execute("SELECT provider FROM sent_idempotency WHERE idem_key='k2'").fetchone()
    assert row["provider"] == "resend"

def test_idempotency_skips_second_send(db):
    with patch("app.mail._call_mailjet", return_value=_ok()) as mj:
        send(db, "alice@example.com", "subj", "body", idem_key="k3")
        send(db, "alice@example.com", "subj", "body", idem_key="k3")
    assert mj.call_count == 1  # second call short-circuited by idempotency

def test_raises_when_both_providers_fail(db):
    with patch("app.mail._call_mailjet", return_value=_resp(503)), \
         patch("app.mail._call_resend", return_value=_resp(503)):
        with pytest.raises(MailFailed):
            send(db, "alice@example.com", "subj", "body", idem_key="k4")
    row = db.execute("SELECT * FROM sent_idempotency WHERE idem_key='k4'").fetchone()
    assert row is None  # claim rolled back on full failure → retry possible

def test_pending_row_blocks_second_call_after_crash(db):
    """If the process died mid-send leaving provider='pending', the next call must skip."""
    db.execute(
        "INSERT INTO sent_idempotency (idem_key, provider) VALUES (?, 'pending')",
        ("k5",),
    )
    with patch("app.mail._call_mailjet") as mj, \
         patch("app.mail._call_resend") as re_:
        send(db, "alice@example.com", "subj", "body", idem_key="k5")
    mj.assert_not_called()
    re_.assert_not_called()
```

- [ ] **Step 2: Run, verify fail**

- [ ] **Step 3: Implement `app/mail.py`**

```python
from __future__ import annotations
import hashlib
import os
import sqlite3
from typing import Any
import requests

class MailFailed(Exception):
    pass

def _idem_key(subscription_id: int, slot_hashes: list[str], cycle_id: str) -> str:
    payload = f"{subscription_id}|{','.join(sorted(slot_hashes))}|{cycle_id}"
    return hashlib.sha256(payload.encode("utf-8")).hexdigest()

def _call_mailjet(to: str, subject: str, body: str) -> Any:
    return requests.post(
        "https://api.mailjet.com/v3.1/send",
        auth=(os.environ["MAILJET_API_KEY"], os.environ["MAILJET_API_SECRET"]),
        json={"Messages": [{
            "From": {"Email": os.environ["MAILJET_FROM_EMAIL"],
                     "Name":  os.environ["MAILJET_FROM_NAME"]},
            "To":   [{"Email": to}],
            "Subject":  subject,
            "TextPart": body,
            "Headers":  {
                "List-Unsubscribe": _list_unsub_header(to),
                "List-Unsubscribe-Post": "List-Unsubscribe=One-Click",
            },
        }]},
        timeout=30,
    )

def _call_resend(to: str, subject: str, body: str) -> Any:
    return requests.post(
        "https://api.resend.com/emails",
        headers={"Authorization": f"Bearer {os.environ['RESEND_API_KEY']}"},
        json={
            "from": f"{os.environ['MAILJET_FROM_NAME']} <{os.environ['MAILJET_FROM_EMAIL']}>",
            "to": [to],
            "subject": subject,
            "text": body,
            "headers": {
                "List-Unsubscribe": _list_unsub_header(to),
                "List-Unsubscribe-Post": "List-Unsubscribe=One-Click",
            },
        },
        timeout=30,
    )

def _list_unsub_header(to_email: str) -> str:
    # Caller is expected to inject the actual unsubscribe URL via the
    # mail-template flow; this is a placeholder header until tied in.
    return f"<{os.environ.get('PUBLIC_BASE_URL', '')}/unsubscribe>"

def send(conn: sqlite3.Connection, to: str, subject: str, body: str,
         *, idem_key: str) -> None:
    """Send `body` to `to`. Idempotent on `idem_key`.

    Order: claim the idempotency row FIRST (atomic INSERT OR IGNORE), then
    attempt sends. If both providers fail the claim is rolled back so a
    retry can proceed. If the process dies between claim and successful
    send, the row remains with provider='pending' and the next call
    short-circuits — preventing a double-send on crash recovery.
    """
    cur = conn.execute(
        "INSERT OR IGNORE INTO sent_idempotency (idem_key, provider) "
        "VALUES (?, 'pending')",
        (idem_key,),
    )
    if cur.rowcount == 0:
        return  # already claimed by an earlier call
    try:
        resp = _call_mailjet(to, subject, body)
        provider = "mailjet"
        if resp.status_code >= 500 or resp.status_code == 429:
            resp = _call_resend(to, subject, body)
            provider = "resend"
        if resp.status_code >= 400:
            raise MailFailed(f"both providers failed; last status {resp.status_code}")
    except Exception:
        conn.execute("DELETE FROM sent_idempotency WHERE idem_key=?", (idem_key,))
        raise
    conn.execute(
        "UPDATE sent_idempotency SET provider=? WHERE idem_key=?",
        (provider, idem_key),
    )
```

- [ ] **Step 4: Run, verify pass**

- [ ] **Step 5: Commit**

```bash
git add app/mail.py tests/test_mail.py
git commit -m "feat(mail): Mailjet+Resend failover with send-side idempotency"
```

---

## Phase 6: Polling Cycle

### Task 6.1: Plan grouping from subscriptions

**Files:**
- Create: `tests/test_planning.py`
- Create: `app/planning.py`

- [ ] **Step 1: Write the failing test**

`tests/test_planning.py`:

```python
from datetime import time
from app.models import Filter
from app.planning import build_plans, plan_for_subscription

def make_filter(types, locations):
    return Filter(
        appointment_types=list(types),
        locations="all" if locations == "all" else list(locations),
        weekdays=[1,2,3,4,5,6,7],
        time_window_start=time(0,0),
        time_window_end=time(23,59),
    )

def test_build_plans_merges_same_filters():
    f = make_filter(["svc-A"], "all")
    subs = [
        ("leipzig", f), ("leipzig", f), ("leipzig", f),
    ]
    plans = build_plans(subs, max_plans_per_city=10)
    assert len(plans) == 1
    assert plans[0].city == "leipzig"
    assert plans[0].appointment_type == "svc-A"

def test_build_plans_splits_by_type():
    plans = build_plans(
        [("leipzig", make_filter(["A"], "all")),
         ("leipzig", make_filter(["B"], "all"))],
        max_plans_per_city=10,
    )
    types = sorted(p.appointment_type for p in plans)
    assert types == ["A", "B"]

def test_build_plans_collapses_to_all_when_cap_exceeded():
    """11 unique (type, location) combinations collapse to per-type "all" plans."""
    subs = []
    for i in range(11):
        subs.append(("leipzig",
                     make_filter(["svc-A"], [f"loc-{i}"])))
    plans = build_plans(subs, max_plans_per_city=10)
    # Should collapse the 11 single-location plans into one "all" plan for svc-A.
    assert len(plans) == 1
    assert plans[0].locations == "all"
    assert plans[0].appointment_type == "svc-A"

def test_would_exceed_cap_signals_overflow():
    from app.planning import would_exceed_cap
    # Cap of 3; existing has 3 distinct types each with one "all" plan
    existing = [
        ("leipzig", make_filter(["A"], "all")),
        ("leipzig", make_filter(["B"], "all")),
        ("leipzig", make_filter(["C"], "all")),
    ]
    new = make_filter(["D"], "all")
    assert would_exceed_cap(existing, "leipzig", new, max_plans_per_city=3) is True
    # Same as existing → no overflow
    same = make_filter(["A"], "all")
    assert would_exceed_cap(existing, "leipzig", same, max_plans_per_city=3) is False
```

- [ ] **Step 2: Run, verify fail**

- [ ] **Step 3: Implement `app/planning.py`**

```python
from __future__ import annotations
from collections import OrderedDict
from app.models import Filter, PollPlan

def plan_for_subscription(city: str, f: Filter) -> list[PollPlan]:
    """A subscription that wants multiple appointment types fans into multiple plans."""
    out = []
    for atype in f.appointment_types:
        out.append(PollPlan(city=city, appointment_type=atype, locations=f.locations))
    return out

def build_plans(subscriptions: list[tuple[str, Filter]],
                *, max_plans_per_city: int) -> list[PollPlan]:
    """Return a deduplicated list of polling plans, collapsing into "all" if cap exceeded."""
    # Step 1: gather all needed plans
    plans: OrderedDict[str, PollPlan] = OrderedDict()
    for city, f in subscriptions:
        for p in plan_for_subscription(city, f):
            plans.setdefault(p.key(), p)
    # Step 2: count per city
    per_city: dict[str, list[PollPlan]] = {}
    for p in plans.values():
        per_city.setdefault(p.city, []).append(p)
    # Step 3: collapse overflow to per-type "all"
    out: list[PollPlan] = []
    for city, city_plans in per_city.items():
        if len(city_plans) <= max_plans_per_city:
            out.extend(city_plans)
            continue
        # Group by appointment_type, replace each group with one "all" plan
        by_type: dict[str, list[PollPlan]] = {}
        for p in city_plans:
            by_type.setdefault(p.appointment_type, []).append(p)
        for atype, _ in by_type.items():
            out.append(PollPlan(city=city, appointment_type=atype, locations="all"))
    return out

def would_exceed_cap(existing: list[tuple[str, Filter]],
                     new_city: str, new_filter: Filter,
                     *, max_plans_per_city: int) -> bool:
    """Predict whether adding (new_city, new_filter) would exceed the cap
    EVEN AFTER the per-type "all" collapse. Used by /subscribe to return 503."""
    augmented = existing + [(new_city, new_filter)]
    plans = build_plans(augmented, max_plans_per_city=max_plans_per_city)
    per_city: dict[str, int] = {}
    for p in plans:
        per_city[p.city] = per_city.get(p.city, 0) + 1
    return any(count > max_plans_per_city for count in per_city.values())
```

- [ ] **Step 4: Run, verify pass**

- [ ] **Step 5: Commit**

```bash
git add app/planning.py tests/test_planning.py
git commit -m "feat(planning): subscription→plan grouping with overflow collapse"
```

---

### Task 6.2: Subscription repository

**Files:**
- Create: `tests/test_repo.py`
- Create: `app/repo.py`

- [ ] **Step 1: Write the failing test**

`tests/test_repo.py`:

```python
from datetime import datetime, timedelta, time
from app.db import connect, init_schema
from app.models import Filter
from app.repo import insert_pending, confirm, soft_delete, active_subscriptions, \
    set_last_notified, record_seen_slot, has_seen_slot
import pytest

@pytest.fixture
def db(tmp_path):
    conn = connect(str(tmp_path / "t.db"))
    init_schema(conn)
    return conn

def _f():
    return Filter(
        appointment_types=["svc-A"], locations="all",
        weekdays=[1,2,3,4,5,6,7],
        time_window_start=time(0,0), time_window_end=time(23,59),
    )

def test_insert_pending_then_confirm(db):
    sub_id = insert_pending(db, email="a@x.com", city="leipzig",
                            language="de", filter_=_f(),
                            ttl_days=90)
    assert sub_id > 0
    assert active_subscriptions(db) == []  # not confirmed yet
    confirm(db, sub_id)
    active = active_subscriptions(db)
    assert len(active) == 1
    assert active[0].email == "a@x.com"
    assert active[0].confirmed_at is not None

def test_soft_delete_removes_from_active(db):
    sid = insert_pending(db, email="a@x.com", city="leipzig",
                         language="de", filter_=_f(), ttl_days=90)
    confirm(db, sid)
    soft_delete(db, sid)
    assert active_subscriptions(db) == []

def test_seen_slot_dedup(db):
    sid = insert_pending(db, email="a@x.com", city="leipzig",
                         language="de", filter_=_f(), ttl_days=90)
    confirm(db, sid)
    assert has_seen_slot(db, sid, "hash1") is False
    record_seen_slot(db, sid, "hash1")
    assert has_seen_slot(db, sid, "hash1") is True
```

- [ ] **Step 2: Run, verify fail**

- [ ] **Step 3: Implement `app/repo.py`**

```python
from __future__ import annotations
import sqlite3
from datetime import datetime, timedelta
from app.models import Filter, Subscription

def insert_pending(conn: sqlite3.Connection, *, email: str, city: str,
                   language: str, filter_: Filter, ttl_days: int) -> int:
    expires_at = (datetime.utcnow() + timedelta(days=ttl_days)).isoformat()
    cur = conn.execute(
        "INSERT INTO subscriptions (email, city, language, filters_json, expires_at) "
        "VALUES (?,?,?,?,?)",
        (email, city, language, filter_.to_json(), expires_at),
    )
    return cur.lastrowid

def confirm(conn: sqlite3.Connection, sub_id: int) -> None:
    conn.execute(
        "UPDATE subscriptions SET confirmed_at=CURRENT_TIMESTAMP "
        "WHERE id=? AND confirmed_at IS NULL",
        (sub_id,),
    )

def soft_delete(conn: sqlite3.Connection, sub_id: int) -> None:
    conn.execute(
        "UPDATE subscriptions SET deleted_at=CURRENT_TIMESTAMP WHERE id=?",
        (sub_id,),
    )

def _row_to_subscription(row: sqlite3.Row) -> Subscription:
    from datetime import datetime
    def _p(s): return datetime.fromisoformat(s) if s else None
    return Subscription(
        id=row["id"],
        email=row["email"],
        city=row["city"],
        language=row["language"],
        sub_filter=Filter.from_json(row["filters_json"]),
        created_at=_p(row["created_at"]),
        confirmed_at=_p(row["confirmed_at"]),
        last_notified_at=_p(row["last_notified_at"]),
        expires_at=_p(row["expires_at"]),
        reminder_sent_at=_p(row["reminder_sent_at"]),
        heartbeat_30d_at=_p(row["heartbeat_30d_at"]),
        heartbeat_60d_at=_p(row["heartbeat_60d_at"]),
        deleted_at=_p(row["deleted_at"]),
    )

def active_subscriptions(conn: sqlite3.Connection) -> list[Subscription]:
    rows = conn.execute(
        "SELECT * FROM subscriptions "
        "WHERE confirmed_at IS NOT NULL "
        "AND deleted_at IS NULL "
        "AND expires_at > CURRENT_TIMESTAMP "
        "ORDER BY id"
    ).fetchall()
    return [_row_to_subscription(r) for r in rows]

def set_last_notified(conn: sqlite3.Connection, sub_id: int) -> None:
    conn.execute("UPDATE subscriptions SET last_notified_at=CURRENT_TIMESTAMP "
                 "WHERE id=?", (sub_id,))

def record_seen_slot(conn: sqlite3.Connection, sub_id: int, slot_hash: str) -> None:
    conn.execute(
        "INSERT OR IGNORE INTO seen_slots (subscription_id, slot_hash) VALUES (?,?)",
        (sub_id, slot_hash),
    )

def has_seen_slot(conn: sqlite3.Connection, sub_id: int, slot_hash: str) -> bool:
    return conn.execute(
        "SELECT 1 FROM seen_slots WHERE subscription_id=? AND slot_hash=?",
        (sub_id, slot_hash),
    ).fetchone() is not None
```

- [ ] **Step 4: Run, verify pass**

- [ ] **Step 5: Commit**

```bash
git add app/repo.py tests/test_repo.py
git commit -m "feat(repo): subscription CRUD + seen-slot dedup helpers"
```

---

### Task 6.3: Polling cycle orchestration

**Files:**
- Create: `tests/test_polling_cycle.py`
- Create: `app/cycle.py`

- [ ] **Step 1: Write the failing test**

`tests/test_polling_cycle.py`:

```python
from datetime import datetime, time
from unittest.mock import patch, MagicMock
import pytest
from app.db import connect, init_schema
from app.models import Filter, Slot
from app.repo import insert_pending, confirm
from app.cycle import run_cycle

@pytest.fixture
def db(tmp_path, monkeypatch):
    # Set the env vars that `cfg=None → load_config()` requires inside run_cycle.
    for k, v in {
        "MAILJET_API_KEY":"m","MAILJET_API_SECRET":"m","MAILJET_FROM_EMAIL":"x@x",
        "MAILJET_FROM_NAME":"x","MAILJET_DAILY_QUOTA":"6000","RESEND_API_KEY":"r",
        "TOKEN_SECRET_PRIMARY":"x"*32,"TOKEN_SECRET_PREVIOUS":"",
        "ADMIN_TOKEN":"a"*32,"PUBLIC_BASE_URL":"https://x",
        "DEDUP_WINDOW_HOURS":"24","RATE_LIMIT_MINUTES":"15",
        "SUBSCRIPTION_TTL_DAYS":"90","RENEWAL_REMINDER_DAYS_BEFORE":"10",
        "MAX_PLANS_PER_CITY":"10","PARSER_CANARY_THRESHOLD_HOURS":"2",
        "SUBSCRIBE_RATELIMIT_PER_IP_PER_HOUR":"99",
        "SUBSCRIBE_RATELIMIT_PER_EMAIL_PER_DAY":"99",
        "DEVELOPER_EMAIL":"dev@x","KOFI_URL":"https://k",
    }.items():
        monkeypatch.setenv(k, v)
    conn = connect(str(tmp_path / "t.db"))
    init_schema(conn)
    return conn

def _f(types, locs="all"):
    return Filter(
        appointment_types=list(types),
        locations="all" if locs == "all" else list(locs),
        weekdays=[1,2,3,4,5,6,7],
        time_window_start=time(0,0), time_window_end=time(23,59),
    )

def test_cycle_sends_one_digest_per_subscriber_on_match(db):
    sid = insert_pending(db, email="a@x.com", city="leipzig",
                        language="de", filter_=_f(["svc-A"]), ttl_days=90)
    confirm(db, sid)
    fake_slots = [Slot("2026-06-10", "10:30", "loc-1", "svc-A", "tok")]
    with patch("app.cycle.get_scraper") as gs, \
         patch("app.cycle.send_digest") as send_d:
        scraper = MagicMock()
        scraper.poll.return_value = fake_slots
        gs.return_value = scraper
        run_cycle(db, max_plans_per_city=10, rate_limit_minutes=15,
                  cycle_id="c1")
        send_d.assert_called_once()
        args = send_d.call_args
        assert args.kwargs["subscription"].id == sid
        assert args.kwargs["matched_slots"] == fake_slots

def test_cycle_skips_already_seen_slot(db):
    sid = insert_pending(db, email="a@x.com", city="leipzig",
                        language="de", filter_=_f(["svc-A"]), ttl_days=90)
    confirm(db, sid)
    fake_slots = [Slot("2026-06-10", "10:30", "loc-1", "svc-A", "tok")]
    from app.repo import record_seen_slot
    record_seen_slot(db, sid, fake_slots[0].hash())
    with patch("app.cycle.get_scraper") as gs, \
         patch("app.cycle.send_digest") as send_d:
        gs.return_value.poll.return_value = fake_slots
        run_cycle(db, max_plans_per_city=10, rate_limit_minutes=15,
                  cycle_id="c1")
    send_d.assert_not_called()

def test_cycle_respects_rate_limit(db):
    sid = insert_pending(db, email="a@x.com", city="leipzig",
                        language="de", filter_=_f(["svc-A"]), ttl_days=90)
    confirm(db, sid)
    db.execute("UPDATE subscriptions SET last_notified_at=CURRENT_TIMESTAMP "
               "WHERE id=?", (sid,))
    with patch("app.cycle.get_scraper") as gs, \
         patch("app.cycle.send_digest") as send_d:
        gs.return_value.poll.return_value = [
            Slot("2026-06-10", "10:30", "loc-1", "svc-A", "tok"),
        ]
        run_cycle(db, max_plans_per_city=10, rate_limit_minutes=15,
                  cycle_id="c1")
    send_d.assert_not_called()
```

- [ ] **Step 2: Run, verify fail**

- [ ] **Step 3: Implement `app/cycle.py`**

```python
from __future__ import annotations
import sqlite3
from datetime import datetime, timedelta
import requests
from app.filters import matches
from app.planning import build_plans
from app.repo import (active_subscriptions, has_seen_slot, record_seen_slot,
                       set_last_notified)
from app.scrapers import get_scraper
from app.models import Subscription, Slot, PollPlan

# Imported here so tests can monkey-patch it.
from app.digest import send_digest  # noqa: E402

def run_cycle(conn: sqlite3.Connection, *, max_plans_per_city: int,
              rate_limit_minutes: int, cycle_id: str,
              cfg=None,
              http: requests.Session | None = None) -> None:
    if cfg is None:
        from app.config import load_config
        cfg = load_config()
    subs = active_subscriptions(conn)
    if not subs:
        return
    http = http or requests.Session()
    plans = build_plans([(s.city, s.sub_filter) for s in subs],
                        max_plans_per_city=max_plans_per_city)
    # Collect slots per plan + per-city canary tracking
    slots_by_plan: dict[str, list[Slot]] = {}
    cities_with_any_slot: set[str] = set()
    cities_polled: set[str] = set()
    for p in plans:
        cities_polled.add(p.city)
        try:
            slots_by_plan[p.key()] = get_scraper(p.city).poll(p, http=http)
            if slots_by_plan[p.key()]:
                cities_with_any_slot.add(p.city)
        except Exception:
            slots_by_plan[p.key()] = []
    # Update per-city canary state in the typed city_state table.
    # Clear `zero_match_since` when at least one plan returned slots;
    # set it on the first all-zero cycle.
    now_iso = datetime.utcnow().isoformat()
    for city in cities_polled:
        # Ensure the row exists.
        conn.execute(
            "INSERT INTO city_state (city) VALUES (?) "
            "ON CONFLICT (city) DO NOTHING",
            (city,),
        )
        if city in cities_with_any_slot:
            conn.execute(
                "UPDATE city_state SET zero_match_since=NULL, "
                "last_polled_at=? WHERE city=?",
                (now_iso, city),
            )
        else:
            conn.execute(
                "UPDATE city_state "
                "SET zero_match_since=COALESCE(zero_match_since, ?), "
                "    last_polled_at=? "
                "WHERE city=?",
                (now_iso, now_iso, city),
            )
    now = datetime.utcnow()
    rate_cutoff = now - timedelta(minutes=rate_limit_minutes)
    for sub in subs:
        if sub.last_notified_at and sub.last_notified_at > rate_cutoff:
            continue
        # Gather candidate slots from any plan that covers this subscription's filter
        candidates: list[Slot] = []
        for plan in plans:
            if plan.city != sub.city:
                continue
            if plan.appointment_type not in sub.sub_filter.appointment_types:
                continue
            for slot in slots_by_plan.get(plan.key(), []):
                if not matches(sub.sub_filter, slot):
                    continue
                if has_seen_slot(conn, sub.id, slot.hash()):
                    continue
                candidates.append(slot)
        if not candidates:
            continue
        # Send and record atomically. Mailjet idempotency prevents double
        # sends across retries; this transaction ensures that IF the email
        # was sent, the seen_slots + last_notified_at writes are visible
        # together — preventing a crash from re-presenting the same slots.
        from app.db import transaction
        # Cache each slot's city + upstream URL so /go/<token> works for
        # any city without hardcoding Leipzig. The scrapers know their
        # own upstream URL format; ask them via the catalog.
        from app.catalog import load_catalog
        scfg = load_catalog(sub.city).scraper_config
        for slot in candidates:
            upstream = _build_upstream_url(scfg, slot)
            conn.execute(
                "INSERT INTO slots_cache (slot_token, city, upstream_url) "
                "VALUES (?, ?, ?) ON CONFLICT (slot_token) DO NOTHING",
                (slot.booking_token, sub.city, upstream),
            )
        send_digest(conn=conn, subscription=sub, matched_slots=candidates,
                    cycle_id=cycle_id, cfg=cfg)
        with transaction(conn):
            for slot in candidates:
                record_seen_slot(conn, sub.id, slot.hash())
            set_last_notified(conn, sub.id)

def _build_upstream_url(scfg: dict, slot) -> str:
    """Vendor-specific upstream booking URL composition.

    For Smart-CJM, the URL is `{base_url}/?uid={uid}&appointment_reserve={token}`.
    Add new branches when adding non-Smart-CJM vendors.
    """
    vendor = scfg.get("vendor")
    if vendor == "smartcjm":
        return (f"{scfg['base_url']}/?uid={scfg['uid']}"
                f"&appointment_reserve={slot.booking_token}")
    raise RuntimeError(f"no upstream-URL builder for vendor: {vendor}")
```

- [ ] **Step 4: Create a stub `app/digest.py`** so the import works:

```python
"""Digest-email composer. Filled in in Task 7.1."""
from __future__ import annotations
import sqlite3
from app.models import Subscription, Slot

def send_digest(*, conn: sqlite3.Connection, subscription: Subscription,
                matched_slots: list[Slot], cycle_id: str, cfg) -> None:
    raise NotImplementedError("digest emails wired up in Task 7.1")
```

- [ ] **Step 5: Run, verify pass**

```bash
pytest tests/test_polling_cycle.py -v
```

Expected: 3 PASS. (The stub digest is only invoked when patched out in tests, so its NotImplementedError never fires.)

- [ ] **Step 6: Commit**

```bash
git add app/cycle.py app/digest.py tests/test_polling_cycle.py
git commit -m "feat(cycle): orchestrate scrape→match→dedup→rate-limit→send pipeline"
```

---

## Phase 7: Digest Email + i18n

### Task 7.1: i18n string bundles + digest template + send_digest

**Files:**
- Create: `app/i18n/de.json`
- Create: `app/i18n/en.json`
- Create: `app/i18n/__init__.py`
- Create: `app/emails/digest.de.txt`
- Create: `app/emails/digest.en.txt`
- Modify: `app/digest.py`
- Create: `tests/test_digest.py`

- [ ] **Step 1: Write the failing test**

`tests/test_digest.py`:

```python
from datetime import datetime, time
from unittest.mock import patch
import pytest
from app.db import connect, init_schema
from app.models import Filter, Slot, Subscription
from app.digest import render_digest_text

def _sub(language="de"):
    return Subscription(
        id=1, email="a@x.com", city="leipzig", language=language,
        sub_filter=Filter(
            appointment_types=["svc-A"], locations="all",
            weekdays=[1,2,3,4,5,6,7],
            time_window_start=time(0,0), time_window_end=time(23,59),
        ),
        created_at=datetime(2026,5,1), confirmed_at=datetime(2026,5,1),
        last_notified_at=None,
        expires_at=datetime(2026,8,1),
        reminder_sent_at=None, heartbeat_30d_at=None, heartbeat_60d_at=None,
        deleted_at=None,
    )

def test_render_digest_de():
    slots = [Slot("2026-06-10", "10:30", "loc-1", "svc-A", "t")]
    text = render_digest_text(_sub("de"), slots,
                              unsubscribe_url="https://x/unsubscribe/tok",
                              public_base_url="https://x", kofi_url="https://ko-fi.com/me")
    assert "2026-06-10" in text
    assert "10:30" in text
    assert "schneller Klick" in text  # burst-congestion line
    assert "https://x/unsubscribe/tok" in text

def test_render_digest_en():
    slots = [Slot("2026-06-10", "10:30", "loc-1", "svc-A", "t")]
    text = render_digest_text(_sub("en"), slots,
                              unsubscribe_url="https://x/unsubscribe/tok",
                              public_base_url="https://x", kofi_url="https://ko-fi.com/me")
    assert "click wins" in text.lower()
```

- [ ] **Step 2: Run, verify fail**

- [ ] **Step 3: Create i18n bundles**

`app/i18n/__init__.py`:

```python
from __future__ import annotations
import json
from functools import lru_cache
from pathlib import Path

I18N_ROOT = Path(__file__).parent

@lru_cache(maxsize=4)
def load(lang: str) -> dict[str, str]:
    if lang not in ("de", "en"):
        lang = "de"
    return json.loads((I18N_ROOT / f"{lang}.json").read_text(encoding="utf-8"))

def t(lang: str, key: str, **kwargs) -> str:
    bundle = load(lang)
    template = bundle.get(key, key)
    return template.format(**kwargs)
```

`app/i18n/de.json` (only digest-relevant keys here; expand later):

```json
{
  "digest.subject": "Neue Termine verfügbar",
  "digest.greeting": "Hallo,",
  "digest.intro": "Es wurden neue passende Termine gefunden:",
  "digest.burst_warning": "Viele Termine öffnen sich gleichzeitig — schneller Klick gewinnt.",
  "digest.unsubscribe": "Abmelden: {unsubscribe_url}",
  "digest.kofi": "Hat dir diese Benachrichtigung geholfen? Du kannst mir einen Kaffee spendieren: {kofi_url} (aber nur, wenn du magst — der Dienst bleibt kostenlos.)"
}
```

`app/i18n/en.json`:

```json
{
  "digest.subject": "New appointments available",
  "digest.greeting": "Hello,",
  "digest.intro": "We found new matching appointments:",
  "digest.burst_warning": "Many appointments open at once — the fastest click wins.",
  "digest.unsubscribe": "Unsubscribe: {unsubscribe_url}",
  "digest.kofi": "Did this notification help? You can buy me a coffee: {kofi_url} (only if you want — the service stays free.)"
}
```

- [ ] **Step 4: Replace `app/digest.py`**

```python
from __future__ import annotations
import sqlite3
from app.i18n import t
from app.models import Subscription, Slot
from app.mail import send, _idem_key

def render_digest_text(sub: Subscription, slots: list[Slot], *,
                       unsubscribe_url: str, public_base_url: str,
                       kofi_url: str) -> str:
    lang = sub.language
    lines = [t(lang, "digest.greeting"), "", t(lang, "digest.intro"), ""]
    # Group by day
    by_day: dict[str, list[Slot]] = {}
    for s in slots:
        by_day.setdefault(s.date, []).append(s)
    for day in sorted(by_day):
        lines.append(day)
        for s in by_day[day]:
            go_url = f"{public_base_url}/go/{s.booking_token}"
            lines.append(f"  {s.time_str}  →  {go_url}")
        lines.append("")
    lines.append(t(lang, "digest.burst_warning"))
    lines.append("")
    lines.append(t(lang, "digest.unsubscribe", unsubscribe_url=unsubscribe_url))
    lines.append("")
    lines.append(t(lang, "digest.kofi", kofi_url=kofi_url))
    return "\n".join(lines)

def send_digest(*, conn: sqlite3.Connection, subscription: Subscription,
                matched_slots: list[Slot], cycle_id: str, cfg) -> None:
    """Send a digest. `cfg` is the loaded Config (passed in by callers
    that already have it loaded — never re-read from os.environ here)."""
    from app.tokens import sign
    unsub_token = sign(subscription.id, "unsubscribe",
                       primary=cfg.token_secret_primary,
                       previous=cfg.token_secret_previous)
    unsub_url = f"{cfg.public_base_url}/unsubscribe/{unsub_token}"
    body = render_digest_text(subscription, matched_slots,
                              unsubscribe_url=unsub_url,
                              public_base_url=cfg.public_base_url,
                              kofi_url=cfg.kofi_url)
    subj = t(subscription.language, "digest.subject")
    key = _idem_key(subscription.id,
                    [s.hash() for s in matched_slots],
                    cycle_id)
    send(conn, subscription.email, subj, body, idem_key=key)
```

- [ ] **Step 5: Run, verify pass**

- [ ] **Step 6: Commit**

```bash
git add app/i18n/ app/digest.py app/emails/ tests/test_digest.py
git commit -m "feat(digest): i18n digest email composition with burst-warning + ko-fi"
```

---

## Phase 8: Web App

### Task 8.1: Flask app skeleton + form + healthz

**Files:**
- Create: `tests/test_web_form.py`
- Create: `app/web.py`
- Create: `app/templates/base.html`
- Create: `app/templates/form.html`

- [ ] **Step 1: Write the failing test**

`tests/test_web_form.py`:

```python
import pytest
from app.web import create_app
from app.db import connect, init_schema

@pytest.fixture
def client(tmp_path, monkeypatch):
    db_path = str(tmp_path / "t.db")
    monkeypatch.setenv("DB_PATH", db_path)
    conn = connect(db_path)
    init_schema(conn)
    app = create_app()
    app.config["TESTING"] = True
    return app.test_client()

def test_healthz(client):
    r = client.get("/healthz")
    assert r.status_code == 200

def test_form_renders(client):
    r = client.get("/")
    assert r.status_code == 200
    assert b"E-Mail" in r.data
    assert b"website" in r.data  # honeypot field name

def test_form_offers_de_and_en(client):
    r_de = client.get("/?lang=de")
    r_en = client.get("/?lang=en")
    assert r_de.status_code == 200 and r_en.status_code == 200
    assert b"Anmelden" in r_de.data or b"abonnieren" in r_de.data.lower()
```

- [ ] **Step 2: Run, verify fail**

- [ ] **Step 3: Implement `app/web.py`**

```python
from __future__ import annotations
import os
from flask import Flask, request, render_template
from app.db import connect, init_schema
from app.catalog import load_catalog

def create_app() -> Flask:
    app = Flask(__name__,
                template_folder="templates",
                static_folder=None)
    # Load config ONCE at startup. Missing env vars surface here, not on
    # the first real request.
    app.config["TERMINE_CONFIG"] = load_config()

    @app.route("/healthz")
    def healthz():
        cfg = app.config["TERMINE_CONFIG"]
        conn = connect(cfg.db_path)
        conn.execute("SELECT 1").fetchone()
        return "ok", 200

    @app.route("/")
    def index():
        lang = request.args.get("lang", "de")
        if lang not in ("de", "en"):
            lang = "de"
        city = request.args.get("city", "leipzig")
        catalog = load_catalog(city)
        return render_template("form.html",
                               lang=lang,
                               city=city,
                               appointment_types=catalog.appointment_types,
                               locations=catalog.locations,
                               kofi_url=app.config["TERMINE_CONFIG"].kofi_url)

    return app

# NOTE: do NOT instantiate `app = create_app()` at module level. Doing so
# calls load_config() at import time, which raises KeyError if any env var
# is missing — including during test collection, where fixtures haven't
# yet had a chance to monkeypatch.setenv(). Gunicorn supports the
# application-factory pattern directly: `gunicorn app.web:create_app()`.
```

- [ ] **Step 4: Create `app/templates/base.html`**

```html
<!doctype html>
<html lang="{{ lang or 'de' }}">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>{% block title %}Termine-Notifier{% endblock %}</title>
  <style>
    body { font-family: system-ui, sans-serif; max-width: 42rem; margin: 2rem auto; padding: 0 1rem; }
    .hidden { position: absolute; left: -9999px; }
    label { display: block; margin: 0.5rem 0 0.25rem; }
    input, select, textarea, button { font-size: 1rem; padding: 0.4rem; }
    footer { margin-top: 4rem; font-size: 0.875rem; color: #666; }
    .disclaimer { background:#f6f6f6; padding:1rem; border-left:3px solid #888; margin-bottom:1.5rem; }
  </style>
</head>
<body>
  {% block content %}{% endblock %}
  <footer>
    Made with ❤️ in Leipzig ·
    <a href="/impressum">Impressum</a> ·
    <a href="/datenschutz">Datenschutz</a> ·
    <a href="{{ kofi_url or 'https://ko-fi.com/jakubwaller' }}">☕ Kaffee spendieren</a>
  </footer>
</body>
</html>
```

- [ ] **Step 5: Create `app/templates/form.html`**

```html
{% extends "base.html" %}
{% block content %}
  <h1>
    {% if lang == 'en' %}Never miss a Leipzig appointment again
    {% else %}Nie wieder freie Bürgerbüro-Termine verpassen{% endif %}
  </h1>

  <p class="disclaimer">
    {% if lang == 'en' %}
      This website is not officially affiliated with the City of Leipzig or
      its Bürgerbüros. We are an independent service that only informs
      about available appointments.
    {% else %}
      Diese Website ist nicht offiziell mit der Stadt Leipzig oder den
      Bürgerbüros der Stadt Leipzig verbunden. Wir sind ein unabhängiger
      Dienst, der ausschließlich über verfügbare Termine informiert.
    {% endif %}
  </p>

  <form action="/subscribe" method="post">
    <input type="hidden" name="lang" value="{{ lang }}">
    <input type="hidden" name="city" value="{{ city }}">

    <label>{% if lang == 'en' %}Email{% else %}E-Mail{% endif %}
      <input type="email" name="email" required autocomplete="email">
    </label>

    <label>{% if lang == 'en' %}Appointment type{% else %}Anliegen{% endif %}
      <select name="appointment_type" required>
        {% for name, uuid in appointment_types.items() %}
          <option value="{{ uuid }}">{{ name }}</option>
        {% endfor %}
      </select>
    </label>

    <label>
      <input type="checkbox" name="all_locations" value="1" checked>
      {% if lang == 'en' %}All locations{% else %}Alle Standorte{% endif %}
    </label>

    <fieldset>
      <legend>{% if lang == 'en' %}Specific locations (uncheck "all" first){% else %}Bestimmte Standorte (oben Häkchen entfernen){% endif %}</legend>
      {% for name, uuid in locations.items() %}
        <label><input type="checkbox" name="locations" value="{{ uuid }}"> {{ name }}</label>
      {% endfor %}
    </fieldset>

    <label>{% if lang == 'en' %}Time window{% else %}Zeitfenster{% endif %}
      <input type="time" name="time_start" value="00:00">
      –
      <input type="time" name="time_end" value="23:59">
    </label>

    <fieldset>
      <legend>{% if lang == 'en' %}Days of week{% else %}Wochentage{% endif %}</legend>
      {% for n, label_de, label_en in [(1,'Mo','Mon'),(2,'Di','Tue'),(3,'Mi','Wed'),(4,'Do','Thu'),(5,'Fr','Fri'),(6,'Sa','Sat'),(7,'So','Sun')] %}
        <label><input type="checkbox" name="weekdays" value="{{ n }}" {% if n <= 5 %}checked{% endif %}>
          {% if lang == 'en' %}{{ label_en }}{% else %}{{ label_de }}{% endif %}</label>
      {% endfor %}
    </fieldset>

    <!-- honeypot -->
    <label class="hidden" aria-hidden="true" tabindex="-1">
      Leave this field empty.
      <input type="text" name="website" autocomplete="off" tabindex="-1">
    </label>

    <button type="submit">
      {% if lang == 'en' %}Subscribe{% else %}Anmelden{% endif %}
    </button>
  </form>
{% endblock %}
```

- [ ] **Step 6: Run, verify pass**

- [ ] **Step 7: Commit**

```bash
git add app/web.py app/templates/ tests/test_web_form.py
git commit -m "feat(web): Flask skeleton, form template, healthz, bilingual content"
```

---

### Task 8.2: Rate-limit + honeypot + /subscribe POST

**Files:**
- Create: `tests/test_subscribe.py`
- Create: `app/ratelimit.py`
- Modify: `app/web.py` (add `/subscribe` route)

- [ ] **Step 1: Write the failing test**

`tests/test_subscribe.py`:

```python
import pytest
from app.web import create_app
from app.db import connect, init_schema

@pytest.fixture
def client(tmp_path, monkeypatch):
    monkeypatch.setenv("DB_PATH", str(tmp_path / "t.db"))
    monkeypatch.setenv("TOKEN_SECRET_PRIMARY", "x"*32)
    monkeypatch.setenv("TOKEN_SECRET_PREVIOUS", "")
    monkeypatch.setenv("SUBSCRIPTION_TTL_DAYS", "90")
    monkeypatch.setenv("SUBSCRIBE_RATELIMIT_PER_IP_PER_HOUR", "2")
    monkeypatch.setenv("SUBSCRIBE_RATELIMIT_PER_EMAIL_PER_DAY", "1")
    monkeypatch.setenv("MAILJET_API_KEY", "mj"); monkeypatch.setenv("MAILJET_API_SECRET", "mj")
    monkeypatch.setenv("MAILJET_FROM_EMAIL", "x@x"); monkeypatch.setenv("MAILJET_FROM_NAME", "x")
    monkeypatch.setenv("MAILJET_DAILY_QUOTA", "6000")
    monkeypatch.setenv("RESEND_API_KEY", "re")
    monkeypatch.setenv("ADMIN_TOKEN", "a"*32)
    monkeypatch.setenv("PUBLIC_BASE_URL", "https://x")
    monkeypatch.setenv("DEDUP_WINDOW_HOURS","24");monkeypatch.setenv("RATE_LIMIT_MINUTES","15")
    monkeypatch.setenv("RENEWAL_REMINDER_DAYS_BEFORE","10");monkeypatch.setenv("MAX_PLANS_PER_CITY","10")
    monkeypatch.setenv("PARSER_CANARY_THRESHOLD_HOURS","2")
    monkeypatch.setenv("DEVELOPER_EMAIL","dev@x");monkeypatch.setenv("KOFI_URL","https://k")
    conn = connect(str(tmp_path / "t.db"))
    init_schema(conn)
    app = create_app(); app.config["TESTING"]=True
    return app.test_client()

def _form(email="alice@example.com"):
    return {
        "lang":"de","city":"leipzig",
        "email": email,
        "appointment_type": "29cd0a26-fe7a-4d65-88cd-1e05fd749c71",
        "all_locations": "1",
        "time_start":"00:00","time_end":"23:59",
        "weekdays": ["1","2","3","4","5"],
        "website":"",  # honeypot empty
    }

def test_subscribe_success_with_mocked_mail(client):
    from unittest.mock import patch
    with patch("app.web._send_confirmation_email") as send:
        r = client.post("/subscribe", data=_form())
    assert r.status_code in (200, 302)
    send.assert_called_once()

def test_honeypot_silently_drops_and_does_not_email(client):
    from unittest.mock import patch
    f = _form()
    f["website"] = "im-a-spam-bot"
    with patch("app.web._send_confirmation_email") as send:
        r = client.post("/subscribe", data=f)
    assert r.status_code in (200, 302)
    send.assert_not_called()

def test_ip_ratelimit(client):
    from unittest.mock import patch
    with patch("app.web._send_confirmation_email"):
        # PER_IP_PER_HOUR=2 set above
        for _ in range(2):
            r = client.post("/subscribe", data=_form(email="b@x.com"),
                            headers={"X-Forwarded-For":"1.2.3.4"})
            assert r.status_code in (200, 302)
        r = client.post("/subscribe", data=_form(email="c@x.com"),
                        headers={"X-Forwarded-For":"1.2.3.4"})
        assert r.status_code == 429
```

- [ ] **Step 2: Run, verify fail**

- [ ] **Step 3: Implement `app/ratelimit.py`**

The per-IP limiter stays in-memory (soft deterrent; gunicorn workers each
hold their own state, so effective per-IP limit is `workers × limit` — that
is intentional and documented). The per-email limiter is DB-backed, since
it has to be reliable (anti-harassment) and shared across workers.

```python
from __future__ import annotations
import sqlite3
import time
from collections import deque

class IPRateLimiter:
    """In-memory sliding-window per-IP counter. Per-process state; with N
    gunicorn workers the effective limit is N×limit. Acceptable for a
    soft bot deterrent. Do NOT use for security-critical decisions."""
    def __init__(self):
        self._events: dict[str, deque[float]] = {}

    def hit(self, key: str, limit: int, window_seconds: int) -> bool:
        now = time.time()
        dq = self._events.setdefault(key, deque())
        while dq and now - dq[0] > window_seconds:
            dq.popleft()
        if len(dq) >= limit:
            return False
        dq.append(now)
        return True

GLOBAL_IP_LIMITER = IPRateLimiter()

def email_rate_limit_ok(conn: sqlite3.Connection, email: str,
                        per_day_limit: int) -> bool:
    """DB-backed per-email rate limit (shared across workers).

    Counts confirmation emails attempted to this address in the last 24h
    by reading the `subscriptions` table (pending and confirmed) plus the
    `sent_idempotency` table for any confirm-* keys. Returns True if
    under the limit.
    """
    row = conn.execute(
        "SELECT COUNT(*) AS n FROM subscriptions "
        "WHERE LOWER(email) = LOWER(?) "
        "AND created_at > datetime('now','-1 day')",
        (email,),
    ).fetchone()
    return (row["n"] if row else 0) < per_day_limit
```

- [ ] **Step 4: Modify `app/web.py` — add subscribe route**

Append to `app/web.py`:

```python
from datetime import time as time_cls
import os
from flask import request, redirect, url_for, abort
from app.config import load_config
from app.db import connect
from app.models import Filter
from app.repo import insert_pending
from app.ratelimit import GLOBAL_IP_LIMITER, email_rate_limit_ok
from app.tokens import sign
from app.mail import send

# (move create_app definition logic so this hangs off the same app instance)
```

Then inside `create_app()` add the routes BEFORE `return app`:

```python
    @app.route("/subscribe", methods=["POST"])
    def subscribe():
        # 1. honeypot
        if request.form.get("website", ""):
            return ("", 200)
        cfg = app.config["TERMINE_CONFIG"]
        ip = request.headers.get("X-Forwarded-For", request.remote_addr or "")
        email = request.form.get("email", "").strip().lower()
        if not email or "@" not in email:
            return ("Invalid email", 400)
        # 2. per-IP rate limit (in-memory, soft)
        if not GLOBAL_IP_LIMITER.hit(f"ip:{ip}",
                                     cfg.subscribe_ratelimit_per_ip_per_hour,
                                     3600):
            return ("Rate limit exceeded", 429)
        # 3. per-email rate limit (DB-backed, hard — shared across workers)
        conn_for_check = connect(cfg.db_path)
        if not email_rate_limit_ok(conn_for_check, email,
                                   cfg.subscribe_ratelimit_per_email_per_day):
            return ("Rate limit exceeded", 429)
        # 4. parse filter from form
        lang = request.form.get("lang", "de")
        city = request.form.get("city", "leipzig")
        atype = request.form.get("appointment_type", "").strip()
        if not atype:
            return ("Missing appointment_type", 400)
        all_locs = request.form.get("all_locations") == "1"
        loc_list = request.form.getlist("locations")
        locations = "all" if all_locs or not loc_list else loc_list
        weekdays = [int(d) for d in request.form.getlist("weekdays") if d.isdigit()]
        if not weekdays:
            weekdays = [1,2,3,4,5,6,7]
        ts = request.form.get("time_start", "00:00")
        te = request.form.get("time_end", "23:59")
        f = Filter(
            appointment_types=[atype],
            locations=locations,
            weekdays=weekdays,
            time_window_start=_parse_hhmm(ts),
            time_window_end=_parse_hhmm(te),
        )
        # 5. plan-cap overflow check + insert atomically (spec §3.2.6).
        # SQLite serializes writers, so wrapping both in one transaction
        # prevents two concurrent /subscribe requests from racing past the
        # cap.
        conn = connect(cfg.db_path)
        from app.repo import active_subscriptions
        from app.planning import would_exceed_cap
        from app.db import transaction
        with transaction(conn):
            existing = [(s.city, s.sub_filter) for s in active_subscriptions(conn)]
            if would_exceed_cap(existing, city, f,
                                max_plans_per_city=cfg.max_plans_per_city):
                return (("Aktuell ist die Warteliste voll. "
                         "Bitte in ein paar Tagen erneut versuchen.") if lang == "de"
                        else "The wait-list is currently full. Please try again in a few days.",
                        503)
            sub_id = insert_pending(conn, email=email, city=city,
                                    language=lang, filter_=f,
                                    ttl_days=cfg.subscription_ttl_days)
        _send_confirmation_email(conn, sub_id, email, lang, cfg)
        return redirect("/?confirmed=pending")

def _parse_hhmm(s: str) -> time_cls:
    h, m = s.split(":")
    return time_cls(int(h), int(m))

def _send_confirmation_email(conn, sub_id: int, email: str, lang: str, cfg):
    from app.i18n import t
    tok = sign(sub_id, "confirm",
               primary=cfg.token_secret_primary,
               previous=cfg.token_secret_previous)
    url = f"{cfg.public_base_url}/confirm/{tok}"
    body_de = f"Bitte bestätige dein Abonnement: {url}"
    body_en = f"Please confirm your subscription: {url}"
    body = body_en if lang == "en" else body_de
    subj = "Bestätigung benötigt" if lang == "de" else "Confirmation needed"
    from app.mail import send as mail_send, _idem_key
    key = _idem_key(sub_id, [], f"confirm-{sub_id}")
    mail_send(conn, email, subj, body, idem_key=key)
```

- [ ] **Step 5: Run, verify pass**

- [ ] **Step 6: Commit**

```bash
git add app/ratelimit.py app/web.py tests/test_subscribe.py
git commit -m "feat(web): /subscribe with honeypot, IP+email rate limits, confirm email"
```

---

### Task 8.3: /confirm, /unsubscribe, /manage routes

**Files:**
- Create: `tests/test_token_routes.py`
- Modify: `app/web.py`

- [ ] **Step 1: Write the failing test**

`tests/test_token_routes.py`:

```python
import pytest
from app.web import create_app
from app.db import connect, init_schema
from app.repo import insert_pending
from app.models import Filter
from datetime import time
import os

@pytest.fixture
def client(tmp_path, monkeypatch):
    db_path = str(tmp_path / "t.db")
    monkeypatch.setenv("DB_PATH", db_path)
    monkeypatch.setenv("TOKEN_SECRET_PRIMARY", "x"*32)
    monkeypatch.setenv("TOKEN_SECRET_PREVIOUS", "")
    # ... (set all other required env vars as in test_subscribe.py — duplicate here)
    for k, v in {
        "SUBSCRIPTION_TTL_DAYS":"90","SUBSCRIBE_RATELIMIT_PER_IP_PER_HOUR":"99",
        "SUBSCRIBE_RATELIMIT_PER_EMAIL_PER_DAY":"99",
        "MAILJET_API_KEY":"m","MAILJET_API_SECRET":"m","MAILJET_FROM_EMAIL":"x@x",
        "MAILJET_FROM_NAME":"x","MAILJET_DAILY_QUOTA":"6000","RESEND_API_KEY":"r",
        "ADMIN_TOKEN":"a"*32,"PUBLIC_BASE_URL":"https://x",
        "DEDUP_WINDOW_HOURS":"24","RATE_LIMIT_MINUTES":"15",
        "RENEWAL_REMINDER_DAYS_BEFORE":"10","MAX_PLANS_PER_CITY":"10",
        "PARSER_CANARY_THRESHOLD_HOURS":"2","DEVELOPER_EMAIL":"d@x","KOFI_URL":"https://k",
    }.items():
        monkeypatch.setenv(k, v)
    conn = connect(db_path); init_schema(conn)
    f = Filter(appointment_types=["A"], locations="all", weekdays=[1,2,3,4,5,6,7],
               time_window_start=time(0,0), time_window_end=time(23,59))
    sid = insert_pending(conn, email="a@x.com", city="leipzig", language="de",
                         filter_=f, ttl_days=90)
    app = create_app(); app.config["TESTING"]=True
    return app.test_client(), sid

def _sign(sid, purpose):
    from app.tokens import sign
    return sign(sid, purpose, primary="x"*32, previous="")

def test_confirm_marks_subscription_confirmed(client):
    c, sid = client
    tok = _sign(sid, "confirm")
    r = c.get(f"/confirm/{tok}")
    assert r.status_code in (200, 302)
    # second confirm is idempotent (no error)
    r2 = c.get(f"/confirm/{tok}")
    assert r2.status_code in (200, 302)

def test_unsubscribe_soft_deletes(client):
    c, sid = client
    _confirm_tok = _sign(sid, "confirm")
    c.get(f"/confirm/{_confirm_tok}")
    unsub = _sign(sid, "unsubscribe")
    r = c.get(f"/unsubscribe/{unsub}")
    assert r.status_code in (200, 302)
    from app.db import connect
    conn = connect(os.environ["DB_PATH"])
    row = conn.execute("SELECT deleted_at FROM subscriptions WHERE id=?", (sid,)).fetchone()
    assert row["deleted_at"] is not None

def test_invalid_token_rejected(client):
    c, sid = client
    r = c.get("/confirm/garbage")
    assert r.status_code == 400
```

- [ ] **Step 2: Run, verify fail**

- [ ] **Step 3: Append routes to `app/web.py`** inside `create_app()`:

```python
    @app.route("/confirm/<token>")
    def confirm_route(token):
        from app.tokens import verify, InvalidToken
        cfg = app.config["TERMINE_CONFIG"]
        try:
            sub_id = verify(token, "confirm",
                            primary=cfg.token_secret_primary,
                            previous=cfg.token_secret_previous)
        except InvalidToken:
            return ("Invalid token", 400)
        from app.repo import confirm
        conn = connect(cfg.db_path)
        confirm(conn, sub_id)
        _send_manage_link_email(conn, sub_id, cfg)
        return ("Subscription confirmed.", 200)

    @app.route("/unsubscribe/<token>")
    def unsubscribe_route(token):
        from app.tokens import verify, InvalidToken
        from app.repo import soft_delete
        cfg = app.config["TERMINE_CONFIG"]
        try:
            sub_id = verify(token, "unsubscribe",
                            primary=cfg.token_secret_primary,
                            previous=cfg.token_secret_previous)
        except InvalidToken:
            return ("Invalid token", 400)
        conn = connect(cfg.db_path)
        soft_delete(conn, sub_id)
        return ("Unsubscribed.", 200)

    @app.route("/manage/<token>", methods=["GET", "POST"])
    def manage_route(token):
        from app.tokens import verify, InvalidToken
        cfg = app.config["TERMINE_CONFIG"]
        try:
            sub_id = verify(token, "manage",
                            primary=cfg.token_secret_primary,
                            previous=cfg.token_secret_previous)
        except InvalidToken:
            return ("Invalid token", 400)
        conn = connect(cfg.db_path)
        if request.method == "POST":
            # Treat as full filter replacement, like /subscribe form
            atype = request.form.get("appointment_type", "").strip()
            if not atype:
                return ("Missing appointment_type", 400)
            all_locs = request.form.get("all_locations") == "1"
            loc_list = request.form.getlist("locations")
            locations = "all" if all_locs or not loc_list else loc_list
            weekdays = [int(d) for d in request.form.getlist("weekdays") if d.isdigit()] or [1,2,3,4,5,6,7]
            ts = request.form.get("time_start","00:00")
            te = request.form.get("time_end","23:59")
            from app.models import Filter
            f = Filter(appointment_types=[atype], locations=locations,
                       weekdays=weekdays,
                       time_window_start=_parse_hhmm(ts),
                       time_window_end=_parse_hhmm(te))
            conn.execute("UPDATE subscriptions SET filters_json=? WHERE id=?",
                         (f.to_json(), sub_id))
            return ("Updated.", 200)
        # GET: render manage form (reuse form.html, prefilled — minimal version here)
        row = conn.execute("SELECT * FROM subscriptions WHERE id=?", (sub_id,)).fetchone()
        if not row or row["deleted_at"] is not None:
            return ("Subscription not found", 404)
        catalog = load_catalog(row["city"])
        return render_template("manage.html",
                               lang=row["language"], city=row["city"],
                               appointment_types=catalog.appointment_types,
                               locations=catalog.locations, token=token)

def _send_manage_link_email(conn, sub_id: int, cfg):
    """Sends a separate email with the /manage link — NEVER in digests."""
    from app.tokens import sign
    from app.mail import send as mail_send, _idem_key
    row = conn.execute("SELECT email, language FROM subscriptions WHERE id=?",
                       (sub_id,)).fetchone()
    tok = sign(sub_id, "manage",
               primary=cfg.token_secret_primary,
               previous=cfg.token_secret_previous)
    url = f"{cfg.public_base_url}/manage/{tok}"
    body = (f"Dein Verwaltungs-Link: {url}\nMit diesem Link kannst du deine "
            f"Einstellungen jederzeit ändern oder dich abmelden."
            if row["language"] == "de" else
            f"Your management link: {url}\nUse it any time to change your "
            f"settings or unsubscribe.")
    subj = "Verwaltungs-Link" if row["language"] == "de" else "Management link"
    key = _idem_key(sub_id, [], f"manage-link-{sub_id}")
    mail_send(conn, row["email"], subj, body, idem_key=key)
```

- [ ] **Step 4: Create `app/templates/manage.html`** as a minimal copy of `form.html` posting to `/manage/{{ token }}`. (Use the same fields; `<form action="/manage/{{ token }}" method="post">`. Omit the lang/city hidden inputs since they're locked to the existing subscription.)

- [ ] **Step 5: Run, verify pass**

- [ ] **Step 6: Commit**

```bash
git add app/web.py app/templates/manage.html tests/test_token_routes.py
git commit -m "feat(web): /confirm, /unsubscribe, /manage with token verification"
```

---

### Task 8.4: /renew, /go, /admin, /datenschutz, /impressum

**Files:**
- Modify: `app/web.py`
- Create: `app/admin.py`
- Create: `app/templates/admin.html`
- Create: `app/templates/datenschutz.html`
- Create: `app/templates/impressum.html`
- Create: `tests/test_admin.py`

- [ ] **Step 1: Write the admin test**

`tests/test_admin.py`:

```python
import pytest
from app.web import create_app
from app.db import connect, init_schema
import os

@pytest.fixture
def client(tmp_path, monkeypatch):
    monkeypatch.setenv("DB_PATH", str(tmp_path/"t.db"))
    # ... same env-var setup as test_token_routes.py (duplicate inline) ...
    for k,v in {
        "TOKEN_SECRET_PRIMARY":"x"*32,"TOKEN_SECRET_PREVIOUS":"",
        "SUBSCRIPTION_TTL_DAYS":"90","SUBSCRIBE_RATELIMIT_PER_IP_PER_HOUR":"99",
        "SUBSCRIBE_RATELIMIT_PER_EMAIL_PER_DAY":"99",
        "MAILJET_API_KEY":"m","MAILJET_API_SECRET":"m","MAILJET_FROM_EMAIL":"x@x",
        "MAILJET_FROM_NAME":"x","MAILJET_DAILY_QUOTA":"6000","RESEND_API_KEY":"r",
        "ADMIN_TOKEN":"admin-tok","PUBLIC_BASE_URL":"https://x",
        "DEDUP_WINDOW_HOURS":"24","RATE_LIMIT_MINUTES":"15",
        "RENEWAL_REMINDER_DAYS_BEFORE":"10","MAX_PLANS_PER_CITY":"10",
        "PARSER_CANARY_THRESHOLD_HOURS":"2","DEVELOPER_EMAIL":"d@x","KOFI_URL":"https://k",
    }.items():
        monkeypatch.setenv(k, v)
    conn = connect(str(tmp_path/"t.db")); init_schema(conn)
    app = create_app(); app.config["TESTING"]=True
    return app.test_client()

def test_admin_requires_token(client):
    r = client.get("/admin")
    assert r.status_code == 401

def test_admin_with_token(client):
    r = client.get("/admin?token=admin-tok")
    assert r.status_code == 200
    assert b"active_subscriptions" in r.data

def test_admin_wrong_token(client):
    r = client.get("/admin?token=nope")
    assert r.status_code == 401

def test_go_route_redirects_on_cache_hit(client):
    import os
    from app.db import connect
    conn = connect(os.environ["DB_PATH"])
    conn.execute(
        "INSERT INTO slots_cache (slot_token, city, upstream_url) "
        "VALUES ('tok1', 'leipzig', 'https://example.eu/book/123')"
    )
    r = client.get("/go/tok1", follow_redirects=False)
    assert r.status_code == 302
    assert r.headers["Location"] == "https://example.eu/book/123"

def test_go_route_returns_410_on_miss(client):
    r = client.get("/go/nonexistent-token", follow_redirects=False)
    assert r.status_code == 410
```

- [ ] **Step 2: Run, verify fail**

- [ ] **Step 3: Implement `app/admin.py`**

```python
from __future__ import annotations
import sqlite3

def stats(conn: sqlite3.Connection) -> dict:
    def scalar(q, *args):
        row = conn.execute(q, args).fetchone()
        return row[0] if row else 0
    def meta_val(key):
        row = conn.execute("SELECT value FROM meta WHERE key=?", (key,)).fetchone()
        return row["value"] if row else None
    # Per-city active subscriptions and current distinct plans
    by_city_subs: dict[str, int] = {}
    by_city_plans: dict[str, int] = {}
    rows = conn.execute(
        "SELECT city, COUNT(*) AS n FROM subscriptions "
        "WHERE deleted_at IS NULL AND confirmed_at IS NOT NULL "
        "AND expires_at > CURRENT_TIMESTAMP "
        "GROUP BY city"
    ).fetchall()
    for r in rows:
        by_city_subs[r["city"]] = r["n"]
    # Per-city distinct plans computed via build_plans on a sample of filters
    try:
        from app.repo import active_subscriptions
        from app.planning import build_plans
        import os
        max_cap = int(os.environ.get("MAX_PLANS_PER_CITY", "10"))
        subs = active_subscriptions(conn)
        plans = build_plans([(s.city, s.sub_filter) for s in subs],
                            max_plans_per_city=max_cap)
        for p in plans:
            by_city_plans[p.city] = by_city_plans.get(p.city, 0) + 1
    except Exception:
        pass
    # Per-city canary marker
    canary_rows = conn.execute(
        "SELECT city, zero_match_since FROM city_state "
        "WHERE zero_match_since IS NOT NULL"
    ).fetchall()
    canary = {r["city"]: r["zero_match_since"] for r in canary_rows}
    return {
        "active_subscriptions":
            scalar("SELECT COUNT(*) FROM subscriptions WHERE deleted_at IS NULL "
                   "AND confirmed_at IS NOT NULL AND expires_at > CURRENT_TIMESTAMP"),
        "active_subscriptions_by_city": by_city_subs,
        "current_plan_count_by_city": by_city_plans,
        "parser_zero_match_since_by_city": canary,
        "pending_confirmation":
            scalar("SELECT COUNT(*) FROM subscriptions WHERE confirmed_at IS NULL "
                   "AND deleted_at IS NULL"),
        "signups_last_24h":
            scalar("SELECT COUNT(*) FROM subscriptions "
                   "WHERE created_at > datetime('now','-1 day')"),
        "signups_last_7d":
            scalar("SELECT COUNT(*) FROM subscriptions "
                   "WHERE created_at > datetime('now','-7 days')"),
        "digests_sent_last_7d":
            scalar("SELECT COUNT(*) FROM sent_idempotency "
                   "WHERE sent_at > datetime('now','-7 days') "
                   "AND provider != 'pending'"),
        "last_housekeeping_at": meta_val("last_housekeeping_at"),
        "last_backup_at":       meta_val("last_backup_at"),
    }
```

- [ ] **Step 4: Append `/admin` route** to `app/web.py` inside `create_app()`:

```python
    @app.route("/admin")
    def admin_route():
        cfg = app.config["TERMINE_CONFIG"]
        token = (request.args.get("token") or
                 (request.headers.get("Authorization", "").removeprefix("Bearer ").strip()))
        # Hash both sides to equal length first — `hmac.compare_digest`
        # short-circuits on length mismatch, leaking the secret's length.
        import hmac as _hmac, hashlib as _hl
        provided = _hl.sha256(token.encode("utf-8")).hexdigest()
        expected = _hl.sha256(cfg.admin_token.encode("utf-8")).hexdigest()
        if not _hmac.compare_digest(provided, expected):
            return ("Unauthorized", 401)
        from app.admin import stats
        conn = connect(cfg.db_path)
        return render_template("admin.html", stats=stats(conn))

    @app.route("/renew/<token>")
    def renew_route(token):
        from app.tokens import verify, InvalidToken
        cfg = app.config["TERMINE_CONFIG"]
        try:
            sid = verify(token, "renew",
                         primary=cfg.token_secret_primary,
                         previous=cfg.token_secret_previous)
        except InvalidToken:
            return ("Invalid token", 400)
        conn = connect(cfg.db_path)
        conn.execute(
            "UPDATE subscriptions SET expires_at=datetime('now', ?) "
            "WHERE id=? AND deleted_at IS NULL",
            (f"+{cfg.subscription_ttl_days} days", sid),
        )
        return ("Subscription renewed.", 200)

    @app.route("/go/<slot_token>")
    def go_route(slot_token):
        # Look up the precomputed upstream URL written into slots_cache at
        # digest-send time. This keeps `/go` city-agnostic — each new
        # scraper plugin just needs to teach `_build_upstream_url` in
        # `app/cycle.py` how to render its city's upstream URL.
        cfg = app.config["TERMINE_CONFIG"]
        conn = connect(cfg.db_path)
        row = conn.execute(
            "SELECT upstream_url FROM slots_cache WHERE slot_token=?",
            (slot_token,),
        ).fetchone()
        if not row:
            return ("This appointment link has expired.", 410)
        # NOTE: if the launch-blocker test in §6 shows the upstream URL is
        # session-bound, replace this with a re-acquire-wsid-then-redirect
        # flow that calls the scraper's `acquire_session` helper.
        return redirect(row["upstream_url"], code=302)

    @app.route("/datenschutz")
    def datenschutz_route():
        return render_template("datenschutz.html")

    @app.route("/impressum")
    def impressum_route():
        return render_template("impressum.html")
```

- [ ] **Step 5: Create `app/templates/admin.html`**

```html
{% extends "base.html" %}
{% block content %}
  <h1>Admin</h1>
  <ul>
    {% for k, v in stats.items() %}
      <li><strong>{{ k }}:</strong> {{ v }}</li>
    {% endfor %}
  </ul>
{% endblock %}
```

- [ ] **Step 6: Create `app/templates/datenschutz.html`** and `app/templates/impressum.html`**

`datenschutz.html`:

```html
{% extends "base.html" %}
{% block content %}
  <h1>Datenschutz</h1>
  <p>Verantwortlicher: Jakub Waller, Hamburg, Deutschland.</p>
  <h2>Welche Daten werden gespeichert?</h2>
  <p>E-Mail, Stadt, Sprache (de/en), gewählte Filter, Zeitstempel. Keine IP, kein Name, keine Cookies, kein Tracking.</p>
  <h2>Rechtsgrundlage</h2>
  <p>Art. 6(1)(a) DSGVO — Einwilligung per Double-Opt-In.</p>
  <h2>Speicherdauer</h2>
  <p>Abonnements laufen 90 Tage nach der Erstanmeldung automatisch ab, sofern sie nicht über den Erneuerungslink verlängert werden. Gelöschte Datensätze werden 30 Tage nach der Löschung endgültig entfernt.</p>
  <h2>Auftragsverarbeiter</h2>
  <p>Mailjet (EU-Server) und Resend (EU-Server) zum Versand der E-Mails. AVV-Verträge liegen vor.</p>
  <h2>Deine Rechte</h2>
  <p>Auskunft (Art. 15), Berichtigung (Art. 16), Löschung (Art. 17), Einschränkung (Art. 18), Datenübertragbarkeit (Art. 20), Widerruf der Einwilligung (Art. 7(3)). Kontakt: jakub@jakubwaller.eu.</p>
{% endblock %}
```

`impressum.html`:

```html
{% extends "base.html" %}
{% block content %}
  <h1>Impressum</h1>
  <p>Angaben gemäß § 5 TMG:</p>
  <p>Jakub Waller<br>
  Hamburg, Deutschland<br>
  Kontakt: jakub@jakubwaller.eu</p>
  <p>Diese Website ist nicht offiziell mit der Stadt Leipzig oder den Bürgerbüros der Stadt Leipzig verbunden.</p>
{% endblock %}
```

- [ ] **Step 7: Run all web tests, verify pass**

```bash
pytest tests/test_web_form.py tests/test_token_routes.py tests/test_admin.py tests/test_subscribe.py -v
```

- [ ] **Step 8: Commit**

```bash
git add app/admin.py app/web.py app/templates/ tests/test_admin.py
git commit -m "feat(web): /renew, /go, /admin (token-gated), /datenschutz, /impressum"
```

---

## Phase 9: Housekeeping & Entry Points

### Task 9.1: Housekeeping (renewals, heartbeats, summary email)

**Files:**
- Create: `tests/test_housekeeping.py`
- Modify: `app/housekeeping.py`

- [ ] **Step 1: Write the failing test**

`tests/test_housekeeping.py`:

```python
from datetime import datetime, time, timedelta
from unittest.mock import patch
import pytest
from app.db import connect, init_schema
from app.models import Filter
from app.repo import insert_pending, confirm
from app.housekeeping import run_once

@pytest.fixture
def db(tmp_path, monkeypatch):
    db_path = str(tmp_path / "t.db")
    monkeypatch.setenv("DB_PATH", db_path)
    monkeypatch.setenv("RENEWAL_REMINDER_DAYS_BEFORE", "10")
    monkeypatch.setenv("SUBSCRIPTION_TTL_DAYS", "90")
    monkeypatch.setenv("TOKEN_SECRET_PRIMARY", "x"*32)
    monkeypatch.setenv("TOKEN_SECRET_PREVIOUS", "")
    monkeypatch.setenv("PUBLIC_BASE_URL", "https://x")
    monkeypatch.setenv("MAILJET_API_KEY", "m"); monkeypatch.setenv("MAILJET_API_SECRET", "m")
    monkeypatch.setenv("MAILJET_FROM_EMAIL","x@x"); monkeypatch.setenv("MAILJET_FROM_NAME","x")
    monkeypatch.setenv("MAILJET_DAILY_QUOTA","6000"); monkeypatch.setenv("RESEND_API_KEY","r")
    monkeypatch.setenv("ADMIN_TOKEN","a"*32)
    monkeypatch.setenv("DEDUP_WINDOW_HOURS","24"); monkeypatch.setenv("RATE_LIMIT_MINUTES","15")
    monkeypatch.setenv("MAX_PLANS_PER_CITY","10"); monkeypatch.setenv("PARSER_CANARY_THRESHOLD_HOURS","2")
    monkeypatch.setenv("SUBSCRIBE_RATELIMIT_PER_IP_PER_HOUR","99")
    monkeypatch.setenv("SUBSCRIBE_RATELIMIT_PER_EMAIL_PER_DAY","99")
    monkeypatch.setenv("DEVELOPER_EMAIL","dev@x"); monkeypatch.setenv("KOFI_URL","https://k")
    conn = connect(db_path); init_schema(conn)
    return conn

def _f():
    return Filter(appointment_types=["A"], locations="all", weekdays=[1,2,3,4,5,6,7],
                  time_window_start=time(0,0), time_window_end=time(23,59))

def test_expired_subscriptions_soft_deleted(db):
    sid = insert_pending(db, email="a@x.com", city="leipzig", language="de",
                         filter_=_f(), ttl_days=90)
    confirm(db, sid)
    db.execute("UPDATE subscriptions SET expires_at=datetime('now','-1 day') WHERE id=?", (sid,))
    with patch("app.mail.send"):
        run_once(db)
    row = db.execute("SELECT deleted_at FROM subscriptions WHERE id=?", (sid,)).fetchone()
    assert row["deleted_at"] is not None

def test_old_deleted_rows_hard_purged(db):
    sid = insert_pending(db, email="a@x.com", city="leipzig", language="de",
                         filter_=_f(), ttl_days=90)
    db.execute("UPDATE subscriptions SET deleted_at=datetime('now','-31 days') WHERE id=?",
               (sid,))
    with patch("app.mail.send"):
        run_once(db)
    row = db.execute("SELECT id FROM subscriptions WHERE id=?", (sid,)).fetchone()
    assert row is None

def test_renewal_reminder_sent_in_window(db):
    sid = insert_pending(db, email="a@x.com", city="leipzig", language="de",
                         filter_=_f(), ttl_days=90)
    confirm(db, sid)
    # Move expires_at to 5 days from now (within 10-day reminder window)
    db.execute("UPDATE subscriptions SET expires_at=datetime('now','+5 days') WHERE id=?", (sid,))
    with patch("app.mail.send") as send:
        run_once(db)
    # send() should be called at least once (for renewal reminder)
    assert send.called
```

- [ ] **Step 2: Run, verify fail**

- [ ] **Step 3: Implement `app/housekeeping.py`**

```python
from __future__ import annotations
import sqlite3
from datetime import datetime, timedelta
from app.config import load_config
from app.mail import send as mail_send, _idem_key
from app.tokens import sign

def run_once(conn: sqlite3.Connection) -> None:
    cfg = load_config()
    _purge_hard(conn)
    _soft_delete_expired(conn)
    _send_renewal_reminders(conn, cfg)
    _send_heartbeats(conn, cfg, milestone_days=30, milestone_col="heartbeat_30d_at")
    _send_heartbeats(conn, cfg, milestone_days=60, milestone_col="heartbeat_60d_at")
    _prune_seen_slots(conn)
    _prune_idempotency(conn)
    _prune_slots_cache(conn)
    _check_parser_canary(conn, cfg)
    _check_backup_health(conn, cfg)
    _send_summary_email(conn, cfg)
    conn.execute(
        "INSERT INTO meta (key,value) VALUES ('last_housekeeping_at', ?) "
        "ON CONFLICT (key) DO UPDATE SET value=excluded.value, updated_at=CURRENT_TIMESTAMP",
        (datetime.utcnow().isoformat(),),
    )

def _purge_hard(conn):
    conn.execute("DELETE FROM subscriptions "
                 "WHERE deleted_at IS NOT NULL "
                 "AND deleted_at < datetime('now','-30 days')")

def _soft_delete_expired(conn):
    conn.execute("UPDATE subscriptions SET deleted_at=CURRENT_TIMESTAMP "
                 "WHERE deleted_at IS NULL AND expires_at < CURRENT_TIMESTAMP")

def _send_renewal_reminders(conn, cfg):
    rows = conn.execute(
        "SELECT id, email, language FROM subscriptions "
        "WHERE deleted_at IS NULL AND confirmed_at IS NOT NULL "
        "AND reminder_sent_at IS NULL "
        "AND expires_at BETWEEN CURRENT_TIMESTAMP AND datetime('now', ?)",
        (f"+{cfg.renewal_reminder_days_before} days",),
    ).fetchall()
    for row in rows:
        tok = sign(row["id"], "renew",
                   primary=cfg.token_secret_primary,
                   previous=cfg.token_secret_previous)
        url = f"{cfg.public_base_url}/renew/{tok}"
        body = (f"Dein Abonnement läuft bald ab. Verlängern: {url}"
                if row["language"] == "de" else
                f"Your subscription will expire soon. Renew: {url}")
        subj = ("Abonnement läuft bald ab" if row["language"] == "de"
                else "Subscription expiring soon")
        from app.db import transaction
        try:
            with transaction(conn):
                # Mark sent BEFORE the API call: the idempotency table in
                # mail.py prevents a second Mailjet call, and this transaction
                # ensures the reminder_sent_at flag is visible even if the
                # process is killed right after the API call returns.
                conn.execute("UPDATE subscriptions SET reminder_sent_at=CURRENT_TIMESTAMP "
                             "WHERE id=?", (row["id"],))
                mail_send(conn, row["email"], subj, body,
                          idem_key=_idem_key(row["id"], [], f"renewal-{row['id']}"))
        except Exception:
            # transaction rolled back; the row is eligible for retry next pass.
            pass

def _send_heartbeats(conn, cfg, *, milestone_days: int, milestone_col: str):
    # Send to subscribers who are past the milestone age AND haven't been
    # notified recently. "Recently" = within the milestone window, so a
    # subscriber notified once 5 days after signup still gets a heartbeat
    # at day 30 if no further notifications happened in between.
    rows = conn.execute(
        f"SELECT id, email, language FROM subscriptions "
        f"WHERE deleted_at IS NULL AND confirmed_at IS NOT NULL "
        f"AND {milestone_col} IS NULL "
        f"AND (last_notified_at IS NULL "
        f"     OR last_notified_at < datetime('now','-{milestone_days} days')) "
        f"AND confirmed_at < datetime('now','-{milestone_days} days')"
    ).fetchall()
    for row in rows:
        manage_tok = sign(row["id"], "manage",
                          primary=cfg.token_secret_primary,
                          previous=cfg.token_secret_previous)
        manage_url = f"{cfg.public_base_url}/manage/{manage_tok}"
        body = (f"Du bist weiterhin abonniert — dein Filter passt einfach noch nicht. "
                f"Hier verwalten: {manage_url}"
                if row["language"] == "de" else
                f"You're still subscribed — your filter just hasn't matched yet. "
                f"Manage here: {manage_url}")
        subj = ("Abo-Update" if row["language"] == "de"
                else "Subscription check-in")
        from app.db import transaction
        try:
            with transaction(conn):
                conn.execute(
                    f"UPDATE subscriptions SET {milestone_col}=CURRENT_TIMESTAMP "
                    f"WHERE id=?",
                    (row["id"],),
                )
                mail_send(conn, row["email"], subj, body,
                          idem_key=_idem_key(row["id"], [],
                                             f"heartbeat-{milestone_days}d-{row['id']}"))
        except Exception:
            pass

def _prune_seen_slots(conn):
    conn.execute("DELETE FROM seen_slots WHERE sent_at < datetime('now','-7 days')")

def _prune_idempotency(conn):
    conn.execute("DELETE FROM sent_idempotency WHERE sent_at < datetime('now','-14 days')")

def _prune_slots_cache(conn):
    # Slots are short-lived in the upstream system; 14 days is generous.
    conn.execute("DELETE FROM slots_cache WHERE cached_at < datetime('now','-14 days')")

def _check_parser_canary(conn, cfg):
    """Email developer if any city has been all-zero for > threshold during business hours."""
    from datetime import datetime, timedelta
    now = datetime.utcnow()
    # Skip outside typical-load hours (08:00–20:00 Europe/Berlin ≈ 06:00–18:00 UTC)
    if not (6 <= now.hour <= 18):
        return
    threshold = timedelta(hours=cfg.parser_canary_threshold_hours)
    rows = conn.execute(
        "SELECT city, zero_match_since, last_canary_alert_at "
        "FROM city_state WHERE zero_match_since IS NOT NULL"
    ).fetchall()
    for row in rows:
        try:
            since = datetime.fromisoformat(row["zero_match_since"])
        except (TypeError, ValueError):
            continue
        if now - since < threshold:
            continue
        city = row["city"]
        if row["last_canary_alert_at"]:
            try:
                last = datetime.fromisoformat(row["last_canary_alert_at"])
                if now - last < timedelta(hours=24):
                    continue
            except ValueError:
                pass
        body = (f"Parser canary: city '{city}' has produced zero matches "
                f"since {row['zero_match_since']} "
                f"(> {cfg.parser_canary_threshold_hours}h).")
        try:
            mail_send(conn, cfg.developer_email,
                      f"[termine-notifier] parser canary: {city}",
                      body,
                      idem_key=_idem_key(0, [], f"canary-{city}-{now.date()}"))
            conn.execute(
                "UPDATE city_state SET last_canary_alert_at=? WHERE city=?",
                (now.isoformat(), city),
            )
        except Exception:
            pass

def _check_backup_health(conn, cfg):
    """Alert if backup hasn't written meta.last_backup_at in > 48h, OR if
    the backup container left a BACKUP-FAIL / BACKUP-METAFAIL sentinel."""
    from datetime import datetime, timedelta
    row = conn.execute(
        "SELECT value FROM meta WHERE key='last_backup_at'"
    ).fetchone()
    stale = True
    if row:
        try:
            last = datetime.fromisoformat(row["value"].rstrip("Z"))
            stale = (datetime.utcnow() - last) > timedelta(hours=48)
        except ValueError:
            pass
    if not stale:
        return
    try:
        mail_send(conn, cfg.developer_email,
                  "[termine-notifier] backup is stale",
                  "meta.last_backup_at is missing or older than 48h. "
                  "Check the backup container logs and /mnt/backup for "
                  "BACKUP-FAIL / BACKUP-METAFAIL sentinel files.",
                  idem_key=_idem_key(0, [], f"backup-stale-{datetime.utcnow().date()}"))
    except Exception:
        pass

def _send_summary_email(conn, cfg):
    from app.admin import stats
    s = stats(conn)
    body = "\n".join(f"{k}: {v}" for k, v in s.items())
    try:
        mail_send(conn, cfg.developer_email, "Nightly summary", body,
                  idem_key=_idem_key(0, [], f"summary-{datetime.utcnow().date()}"))
    except Exception:
        pass
```

- [ ] **Step 4: Run, verify pass**

- [ ] **Step 5: Commit**

```bash
git add app/housekeeping.py tests/test_housekeeping.py
git commit -m "feat(housekeeping): expiries, purges, renewals, heartbeats, summary email"
```

---

### Task 9.2: Poller entrypoint

**Files:**
- Create: `app/poller.py`

- [ ] **Step 1: Implement `app/poller.py`**

```python
from __future__ import annotations
import os
import time as time_mod
from datetime import datetime, timedelta
import requests
from app.config import load_config
from app.db import connect, init_schema
from app.cycle import run_cycle
from app.housekeeping import run_once as housekeeping_run

def main() -> None:
    cfg = load_config()
    conn = connect(cfg.db_path)
    init_schema(conn)
    http = requests.Session()
    consecutive_failures = 0
    while True:
        # Sleep until next minute boundary
        now = time_mod.time()
        sleep_s = 60 - (now % 60)
        time_mod.sleep(sleep_s)
        cycle_id = datetime.utcnow().strftime("%Y%m%dT%H%M")
        try:
            _maybe_housekeeping(conn)
            run_cycle(conn,
                      max_plans_per_city=cfg.max_plans_per_city,
                      rate_limit_minutes=cfg.rate_limit_minutes,
                      cycle_id=cycle_id,
                      cfg=cfg,
                      http=http)
            consecutive_failures = 0
        except Exception as exc:
            consecutive_failures += 1
            print(f"cycle {cycle_id} failed (consecutive={consecutive_failures}): {exc}",
                  flush=True)
            if consecutive_failures >= 3:
                _maybe_alert(conn, cfg, str(exc))

def _maybe_alert(conn, cfg, last_error: str) -> None:
    """Send a developer-alert email at most once per 24h."""
    from datetime import datetime, timedelta
    row = conn.execute(
        "SELECT value FROM meta WHERE key='last_failure_alert_at'"
    ).fetchone()
    if row:
        try:
            if datetime.utcnow() - datetime.fromisoformat(row["value"]) < timedelta(hours=24):
                return
        except ValueError:
            pass
    try:
        from app.mail import send as mail_send, _idem_key
        body = (f"Poller has been failing repeatedly. Last error:\n\n{last_error}\n\n"
                f"Check logs: docker compose logs poller")
        mail_send(conn, cfg.developer_email,
                  "[termine-notifier] poller failure burst",
                  body,
                  idem_key=_idem_key(0, [], f"alert-{datetime.utcnow().date()}"))
        conn.execute(
            "INSERT INTO meta (key, value) VALUES ('last_failure_alert_at', ?) "
            "ON CONFLICT (key) DO UPDATE SET value=excluded.value, "
            "updated_at=CURRENT_TIMESTAMP",
            (datetime.utcnow().isoformat(),),
        )
    except Exception as inner:
        print(f"failed to send alert email: {inner}", flush=True)

def _maybe_housekeeping(conn) -> None:
    row = conn.execute(
        "SELECT value FROM meta WHERE key='last_housekeeping_at'"
    ).fetchone()
    if not row:
        housekeeping_run(conn)
        return
    last = datetime.fromisoformat(row["value"])
    if datetime.utcnow() - last > timedelta(hours=24):
        housekeeping_run(conn)

if __name__ == "__main__":
    main()
```

- [ ] **Step 2: No test required for the loop itself** (tested indirectly through `run_cycle` and `housekeeping.run_once`).

- [ ] **Step 3: Commit**

```bash
git add app/poller.py
git commit -m "feat(poller): minute-aligned cycle loop with daily housekeeping check"
```

---

## Phase 10: Docker & Caddy

### Task 10.1: Dockerfiles + docker-compose + Caddyfile

**Files:**
- Create: `Dockerfile.web`
- Create: `Dockerfile.poller`
- Create: `Dockerfile.backup`
- Create: `scripts/backup-loop.sh`
- Create: `docker-compose.yml`
- Create: `Caddyfile`

- [ ] **Step 1: Write `Dockerfile.web`**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml ./
RUN pip install --no-cache-dir .
COPY app/ app/
COPY catalog/ catalog/
ENV PYTHONUNBUFFERED=1
# Application-factory form: gunicorn calls create_app() at startup, so
# load_config() runs only when the container actually boots — not on import.
CMD ["gunicorn", "-w", "2", "-b", "0.0.0.0:8000", "app.web:create_app()"]
```

- [ ] **Step 2: Write `Dockerfile.poller`**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml ./
RUN pip install --no-cache-dir .
COPY app/ app/
COPY catalog/ catalog/
ENV PYTHONUNBUFFERED=1
CMD ["python", "-m", "app.poller"]
```

- [ ] **Step 3: Write `Dockerfile.backup`**

```dockerfile
FROM alpine:3.20
RUN apk add --no-cache sqlite bash gzip findutils
COPY scripts/backup-loop.sh /usr/local/bin/backup-loop.sh
RUN chmod +x /usr/local/bin/backup-loop.sh
CMD ["/usr/local/bin/backup-loop.sh"]
```

- [ ] **Step 4: Write `scripts/backup-loop.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
DB=${DB_PATH:-/data/app.db}
DEST=/backup
RETENTION_DAYS=${BACKUP_RETENTION_DAYS:-30}
while true; do
  ts=$(date +%F)
  iso=$(date -u +%FT%TZ)
  tmp="$DEST/app-${ts}.db"
  echo "[backup] $iso snapshot → $tmp"
  if sqlite3 "$DB" ".backup '$tmp'"; then
    gzip -f "$tmp"
    find "$DEST" -name 'app-*.db.gz' -mtime "+$RETENTION_DAYS" -delete || true
    # Record success in meta. Retry up to 3× with backoff to handle a
    # transient SQLITE_BUSY when the poller or web container is mid-write.
    # If we still fail, write a sentinel file so the housekeeping pass
    # can surface the failure via the developer-alert path.
    recorded=0
    for attempt in 1 2 3; do
      if sqlite3 "$DB" "INSERT INTO meta (key, value) \
        VALUES ('last_backup_at', '$iso') \
        ON CONFLICT (key) DO UPDATE SET value=excluded.value, \
        updated_at=CURRENT_TIMESTAMP" 2>/dev/null; then
        recorded=1
        break
      fi
      sleep "$attempt"
    done
    if [ "$recorded" = "0" ]; then
      echo "[backup] WARN: could not record last_backup_at after 3 tries"
      echo "$iso snapshot OK but meta write failed" > "$DEST/BACKUP-METAFAIL-${ts}.txt"
    fi
  else
    echo "[backup] FAIL: sqlite3 .backup exited non-zero"
    echo "$iso snapshot failed" > "$DEST/BACKUP-FAIL-${ts}.txt"
  fi
  sleep 86400
done
```

- [ ] **Step 5: Write `docker-compose.yml`**

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
    expose:
      - "8000"

  poller:
    build: { context: ., dockerfile: Dockerfile.poller }
    env_file: .env
    volumes:
      - ./data:/data
    restart: unless-stopped

  backup:
    build: { context: ., dockerfile: Dockerfile.backup }
    environment:
      - DB_PATH=/data/app.db
    volumes:
      # RW (not :ro) so the backup loop can record meta.last_backup_at after
      # each successful snapshot. The script only INSERTs/UPDATEs that one
      # row; the .backup operation itself reads.
      - ./data:/data
      - /mnt/backup:/backup
    restart: unless-stopped

volumes:
  caddy_data:
  caddy_config:
```

- [ ] **Step 6: Write `Caddyfile`**

```
termine.jakubwaller.eu {
    reverse_proxy web:8000
}
```

- [ ] **Step 7: Smoke-test the compose project locally** (Linux/Mac with Docker installed):

```bash
mkdir -p data
cp .env.example .env  # then fill in real values
docker compose build
docker compose up -d
docker compose logs --tail=20
curl http://localhost:8000/healthz  # if running locally without TLS
docker compose down
```

(For Caddy + TLS this requires a public domain; locally you can `curl` web directly via its port if you also publish 8000 temporarily.)

- [ ] **Step 8: Commit**

```bash
git add Dockerfile.web Dockerfile.poller Dockerfile.backup scripts/backup-loop.sh docker-compose.yml Caddyfile
git commit -m "feat(ops): Dockerfiles, docker-compose, Caddyfile, backup container"
```

---

### Task 10.2: Deployment doc

**Files:**
- Create: `docs/deployment.md`
- Create: `scripts/smartcheck.sh`

- [ ] **Step 1: Write `scripts/smartcheck.sh`**

```bash
#!/usr/bin/env bash
# Runs on the Pi host (NOT inside Docker) via a systemd timer.
# Appends SMART output to a file under the bind-mounted backup volume,
# where the backup container can pick it up and email it.
set -euo pipefail
DEVICE=${1:-/dev/sda}
OUTDIR=${OUTDIR:-/mnt/backup/smart}
mkdir -p "$OUTDIR"
ts=$(date +%F)
smartctl -a "$DEVICE" > "$OUTDIR/smart-${ts}.txt" || true
# Alert via diff of reallocated-sector count from previous snapshot
prev=$(ls -1t "$OUTDIR"/smart-*.txt 2>/dev/null | sed -n 2p || true)
if [ -n "$prev" ]; then
  prev_reall=$(grep -E 'Reallocated_Sector_Ct' "$prev"   | awk '{print $10}' || echo 0)
  curr_reall=$(grep -E 'Reallocated_Sector_Ct' "$OUTDIR/smart-${ts}.txt" | awk '{print $10}' || echo 0)
  if [ "${curr_reall:-0}" != "${prev_reall:-0}" ]; then
    echo "SMART: reallocated sectors changed from ${prev_reall} to ${curr_reall}" \
      > "$OUTDIR/ALERT-${ts}.txt"
  fi
fi
```

- [ ] **Step 2: Write `docs/deployment.md`**

```markdown
# Deployment

## Prerequisites

- Raspberry Pi running Docker + Docker Compose.
- Domain `termine.jakubwaller.eu` (or replacement) pointing to the Pi's
  public IP. Ports 80 and 443 forwarded to the Pi.
- USB HDD mounted at `/mnt/backup` (auto-mount via `/etc/fstab`).
- Mailjet and Resend accounts with verified sender domain.
- SPF / DKIM / DMARC records configured on `jakubwaller.eu` before any send.

## First deploy

1. Clone the repo to the Pi.
2. Copy `.env.example` to `.env` and fill in real secrets:
   - 32-byte `TOKEN_SECRET_PRIMARY` and `ADMIN_TOKEN` (e.g., `openssl rand -hex 32`).
   - Mailjet and Resend API keys.
3. Verify the USB HDD is mounted at `/mnt/backup`.
4. `docker compose up -d`.
5. Watch logs: `docker compose logs -f`.
6. Verify healthz: `curl https://termine.jakubwaller.eu/healthz`.

## Token-secret rotation

1. Set `TOKEN_SECRET_PREVIOUS=$TOKEN_SECRET_PRIMARY` in `.env`.
2. Generate a new secret: `openssl rand -hex 32` → `TOKEN_SECRET_PRIMARY`.
3. `docker compose restart web poller`.
4. Existing tokens remain valid; next rotation invalidates them.

## SMART monitoring (host-side systemd timer)

Install `smartmontools`. Add `/etc/systemd/system/termine-smart.service`:

```
[Unit]
Description=Termine-Notifier SMART check

[Service]
Type=oneshot
ExecStart=/path/to/termine-notifier/scripts/smartcheck.sh /dev/sda
```

And `/etc/systemd/system/termine-smart.timer`:

```
[Unit]
Description=Weekly SMART check

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target
```

`systemctl enable --now termine-smart.timer`.

## Off-Pi backup (secondary)

From your laptop:

```
rsync -av jakub@pi:/mnt/backup/ ~/termine-backups/
```

Cron weekly.

## IP-block runbook

If `terminvereinbarung.leipzig.de` starts returning 403 for the Pi's IP:

1. Stop the poller: `docker compose stop poller`.
2. Email the city: `verwaltung@leipzig.de`. Subject: "Anfrage zu
   Terminvereinbarung-Notifier". Explain: free notification service, no
   booking, polling rate ≤ 10 req/min, GDPR-compliant, open source at
   `github.com/jakubwaller/termine-notifier`. Ask if there is a way to
   continue operation that the city would accept.
3. Do NOT attempt to rotate IPs or use proxies — this is ethically
   worse than the polling itself and undermines the legal posture.
```

- [ ] **Step 3: Commit**

```bash
git add docs/deployment.md scripts/smartcheck.sh
git commit -m "docs: deployment guide, SMART timer, IP-block runbook"
```

---

## Phase 11: End-to-End Integration Test

### Task 11.1: Walking subscribe → confirm → poll → digest → unsubscribe

**Files:**
- Create: `tests/test_e2e_flow.py`

- [ ] **Step 1: Write the integration test**

```python
import os
from datetime import time
from unittest.mock import patch
import pytest
from app.web import create_app
from app.db import connect, init_schema
from app.models import Slot

@pytest.fixture
def env(tmp_path, monkeypatch):
    db_path = str(tmp_path / "t.db")
    for k, v in {
        "DB_PATH": db_path,
        "TOKEN_SECRET_PRIMARY":"x"*32,"TOKEN_SECRET_PREVIOUS":"",
        "ADMIN_TOKEN":"a"*32,
        "MAILJET_API_KEY":"m","MAILJET_API_SECRET":"m","MAILJET_FROM_EMAIL":"x@x",
        "MAILJET_FROM_NAME":"x","MAILJET_DAILY_QUOTA":"6000","RESEND_API_KEY":"r",
        "PUBLIC_BASE_URL":"https://x","DEDUP_WINDOW_HOURS":"24","RATE_LIMIT_MINUTES":"15",
        "SUBSCRIPTION_TTL_DAYS":"90","RENEWAL_REMINDER_DAYS_BEFORE":"10",
        "MAX_PLANS_PER_CITY":"10","PARSER_CANARY_THRESHOLD_HOURS":"2",
        "SUBSCRIBE_RATELIMIT_PER_IP_PER_HOUR":"99","SUBSCRIBE_RATELIMIT_PER_EMAIL_PER_DAY":"99",
        "DEVELOPER_EMAIL":"dev@x","KOFI_URL":"https://k",
    }.items():
        monkeypatch.setenv(k, v)
    conn = connect(db_path); init_schema(conn)
    return db_path

def test_full_flow(env):
    app = create_app(); app.config["TESTING"] = True
    c = app.test_client()

    sent_mails: list[tuple[str, str]] = []

    def fake_send(conn, to, subject, body, *, idem_key):
        sent_mails.append((to, subject))

    # 1. subscribe (mock mail)
    with patch("app.mail.send", side_effect=fake_send):
        r = c.post("/subscribe", data={
            "lang":"de","city":"leipzig",
            "email":"alice@example.com",
            "appointment_type":"29cd0a26-fe7a-4d65-88cd-1e05fd749c71",
            "all_locations":"1",
            "time_start":"00:00","time_end":"23:59",
            "weekdays":["1","2","3","4","5"],
            "website":"",
        })
        assert r.status_code in (200, 302)
    assert any("Bestätigung" in s for _, s in sent_mails)

    # 2. confirm
    conn = connect(env)
    sid = conn.execute("SELECT id FROM subscriptions WHERE email='alice@example.com'").fetchone()["id"]
    from app.tokens import sign
    tok = sign(sid, "confirm", primary="x"*32, previous="")
    with patch("app.mail.send", side_effect=fake_send):
        r = c.get(f"/confirm/{tok}")
    assert r.status_code in (200, 302)
    assert any("Verwaltungs-Link" in s or "Management link" in s for _, s in sent_mails)

    # 3. run a polling cycle with a synthetic slot
    from app.cycle import run_cycle
    fake_slots = [Slot("2026-06-10", "10:30",
                       "loc-1",
                       "29cd0a26-fe7a-4d65-88cd-1e05fd749c71",
                       "tok")]
    from unittest.mock import MagicMock
    scraper = MagicMock(); scraper.poll.return_value = fake_slots
    with patch("app.cycle.get_scraper", return_value=scraper), \
         patch("app.mail.send", side_effect=fake_send):
        run_cycle(conn, max_plans_per_city=10, rate_limit_minutes=15,
                  cycle_id="e2e-1")
    digest_seen = any("Neue Termine" in s for _, s in sent_mails)
    assert digest_seen

    # 4. run a SECOND cycle with the same slot — must dedup, no new digest
    sent_count_before = sum(1 for _, s in sent_mails if "Neue Termine" in s)
    with patch("app.cycle.get_scraper", return_value=scraper), \
         patch("app.mail.send", side_effect=fake_send):
        run_cycle(conn, max_plans_per_city=10, rate_limit_minutes=0,
                  cycle_id="e2e-2")
    sent_count_after = sum(1 for _, s in sent_mails if "Neue Termine" in s)
    assert sent_count_after == sent_count_before, \
        "dedup failed: same slot triggered a second digest"

    # 5. unsubscribe
    unsub_tok = sign(sid, "unsubscribe", primary="x"*32, previous="")
    r = c.get(f"/unsubscribe/{unsub_tok}")
    assert r.status_code in (200, 302)
    row = conn.execute("SELECT deleted_at FROM subscriptions WHERE id=?", (sid,)).fetchone()
    assert row["deleted_at"] is not None
```

- [ ] **Step 2: Run, verify pass**

```bash
pytest tests/test_e2e_flow.py -v
```

Expected: PASS. Iterate on any cross-component issues.

- [ ] **Step 3: Run the full suite**

```bash
pytest -v
```

Expected: all tests PASS.

- [ ] **Step 4: Commit**

```bash
git add tests/test_e2e_flow.py
git commit -m "test(e2e): full subscribe → confirm → poll → digest → unsubscribe flow"
```

---

## Phase 12: Repo Polish & First Deploy

### Task 12.1: Copy design + findings docs into the new repo

- [ ] **Step 1: Copy specs**

```bash
mkdir -p docs/specs
cp ../leipzigappointmentsbotpremium/docs/superpowers/specs/2026-05-03-leipzig-termine-notifier-design.md docs/specs/
cp ../leipzigappointmentsbotpremium/docs/superpowers/specs/2026-05-03-leipzig-termine-notifier-findings.md docs/specs/
```

- [ ] **Step 2: Commit**

```bash
git add docs/specs/
git commit -m "docs: copy design spec and findings notes into project"
```

---

### Task 12.2: Create private GitHub repo and push

- [ ] **Step 1: Create the repo**

```bash
gh repo create jakubwaller/termine-notifier --private --source=. --remote=origin --push
```

If `gh` is not installed, create the repo manually at <https://github.com/new>
as `termine-notifier`, private, then:

```bash
git remote add origin git@github.com:jakubwaller/termine-notifier.git
git push -u origin main
```

- [ ] **Step 2: Set up branch protection** (later, when going public).

No commit needed — this is a remote operation.

---

### Task 12.3: Run launch-blocker tests before going public

Per the design spec §6 rollout step 1:

- [ ] **Step 1: Booking-link session-bound check**

Deploy to the Pi, subscribe yourself, wait for a real digest email, click a slot link in a fresh private browser window. If it 404s or "Session abgelaufen", implement the `/go/<slot_token>` re-acquire flow in `app/web.py:go_route`. Re-test. **Don't proceed without this resolved.**

- [ ] **Step 2: Email deliverability check**

Send a real digest to test addresses on gmail.com, outlook.com, gmx.de, web.de, posteo.de. Check that all five land in inbox, not spam. If any spam, fix SPF/DKIM/DMARC and retry.

- [ ] **Step 3: Parser canary check**

Inspect `meta.parser_zero_match_since` and `/admin` after 4 hours of running during the day. Confirm canary fires correctly if forced (e.g., by temporarily breaking the parser regex).

- [ ] **Step 4: Backup restore test**

```bash
cp /mnt/backup/app-$(date +%F).db.gz /tmp/
gunzip /tmp/app-*.db.gz
sqlite3 /tmp/app-*.db "SELECT COUNT(*) FROM subscriptions"
```

- [ ] **Step 5: Document any deviations from spec in `docs/specs/` as a "v1 deviations" appendix.**

No code commit required from these steps unless issues are found.

---

### Task 12.4: Flip the repo public + send the declarative email

- [ ] **Step 1: Flip the GitHub repo to public**

```bash
gh repo edit jakubwaller/termine-notifier --visibility=public --accept-visibility-change-consequences
```

- [ ] **Step 2: Send the declarative email to Stadt Leipzig**

Per the design spec §6 rollout step 3. One-page text, no asks, just notification. Address: `verwaltung@leipzig.de`. Subject: "Hinweis: kostenloser Benachrichtigungsdienst für Bürgerbüro-Termine".

(Manual step — no code change.)

- [ ] **Step 3: Soft launch**

Post on r/Leipzig and pitch leipglo.com.

---

## Self-Review

(Skipping the writing per the skill — this section is performed by the plan author after writing. See checklist below.)

### Spec coverage check

- §1 Goal — covered by §0 README, §3 architecture
- §2 Non-goals — README + §4.3 license norms
- §3.1 Topology — Phase 10 (docker-compose, Caddyfile)
- §3.2 Components — Phases 1–9
- §3.3 Data model — Task 1.2
- §3.4 Configuration — Task 1.1 + `.env.example`
- §3.5 DSGVO posture — Task 8.4 (`datenschutz.html`)
- §3.6 Error handling, observability, ops — Task 9.1 housekeeping, Task 10.2 deployment, Task 12.3 launch-blockers
- §3.7 IP/brand-assets rule — Task 0.1 README + Task 8.1 templates
- §4 Repo & deployment — Phases 10–12
- §5 Testing — every task includes tests; Task 11.1 is E2E
- §6 Rollout plan — Task 12.3 + 12.4
- §7 References — Task 12.1 copies the specs in

Gaps:
- The `/go/<slot_token>` redirect's session-rebinding behavior isn't fully implemented — Task 12.3 explicitly leaves it as a launch-blocker pending real-world test. Listed as such, not a hidden gap.
- DKIM/SPF/DMARC are operational tasks in `docs/deployment.md` rather than code — appropriate, since they are DNS records, not Python.

### Type consistency

- `Filter`, `Slot`, `Subscription`, `PollPlan` defined in Task 1.3, used consistently downstream.
- `send_digest(*, conn, subscription, matched_slots, cycle_id, cfg)` — same signature in Task 6.3 stub and Task 7.1 implementation.
- `_idem_key(subscription_id, slot_hashes, cycle_id)` — same args in Tasks 5.1 and 7.1.

No drift detected.

### Placeholder scan

No "TBD", "TODO", or "implement later" markers in the plan. Where engineer-side decisions remain (e.g., golden HTML capture, deliverability test results, optional `/go` redirect rebuild), they are spelled out as concrete steps within the right task.
