# Pi-hole Architecture Wiki

> **Source repository:** `~/pi-hole` (xunema/pi-hole fork of Pi-hole/Pi-hole)
> **Database schema version:** 20
> **Last analysed:** 2026-02-23

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Component Map](#2-component-map)
3. [The Gravity Database (gravity.db)](#3-the-gravity-database-gravitydb)
   - 3.1 [Core Tables](#31-core-tables)
   - 3.2 [Junction / Many-to-Many Tables](#32-junction--many-to-many-tables)
   - 3.3 [Query-Time Views](#33-query-time-views)
   - 3.4 [Automatic Triggers](#34-automatic-triggers)
4. [The Group-Based Permission Model](#4-the-group-based-permission-model)
5. [Gravity Update Pipeline (gravity.sh)](#5-gravity-update-pipeline-gravitysh)
6. [FTL – The DNS Resolver Engine](#6-ftl--the-dns-resolver-engine)
7. [REST API Layer (api.sh)](#7-rest-api-layer-apish)
8. [CLI Management (list.sh / pihole command)](#8-cli-management-listsh--pihole-command)
9. [Database Migration System](#9-database-migration-system)
10. [File Layout on Disk](#10-file-layout-on-disk)

---

## RESEARCH SECTION A — Pi-hole as a User Management Portal (vs. Captive Portal)

11. [What a Captive Portal Does vs. What Pi-hole Does](#11-what-a-captive-portal-does-vs-what-pi-hole-does)
12. [Where Pi-hole Can Replace Captive Portal Functions](#12-where-pi-hole-can-replace-captive-portal-functions)
13. [Where Pi-hole Cannot Replace a Captive Portal](#13-where-pi-hole-cannot-replace-a-captive-portal)
14. [Hybrid Architecture Recommendation](#14-hybrid-architecture-recommendation)

---

## RESEARCH SECTION B — Blocklist Features

15. [Blocklist Types and Sources](#15-blocklist-types-and-sources)
16. [Per-Group Blocklist Assignment](#16-per-group-blocklist-assignment)
17. [Domain List Types (Allow / Deny / Regex)](#17-domain-list-types-allow--deny--regex)
18. [The Gravity Table and How Blocking Works at Query Time](#18-the-gravity-table-and-how-blocking-works-at-query-time)

---

## RESEARCH SECTION C — Quality of Service (QoS)

19. [Does Pi-hole Have QoS?](#19-does-pi-hole-have-qos)
20. [What Pi-hole Controls vs. What QoS Controls](#20-what-pi-hole-controls-vs-what-qos-controls)
21. [Recommended QoS Companions](#21-recommended-qos-companions)

---

---

# Part 1 — Architecture

---

## 1. System Overview

Pi-hole is a **network-wide DNS sinkhole**. It intercepts DNS queries from every device on the local network and decides — before a connection is ever made — whether to resolve a domain (allow it) or return a null/blocked response (deny it).

It does **not** operate at the TCP/IP packet layer. It has no visibility into raw traffic, bandwidth, connection state, or HTTP content. Its entire power sits in the DNS layer.

```
Client Device
    │
    │  DNS query: "ads.example.com ?"
    ▼
Pi-hole (pihole-FTL)
    │
    ├─ Is this domain in vw_denylist or vw_gravity for this client's groups?
    │       YES → return 0.0.0.0  (blocked / sinkholed)
    │       NO  → forward to upstream DNS resolver
    │
    └─ Upstream DNS (e.g. 1.1.1.1, 8.8.8.8)
            │
            ▼
        Real IP returned to client → connection proceeds
```

**Core principle:** Block at the name-resolution level, before any network connection is established.

---

## 2. Component Map

```
┌─────────────────────────────────────────────────────────────────┐
│                        Pi-hole Stack                            │
│                                                                 │
│  ┌──────────────┐   ┌────────────────┐   ┌──────────────────┐  │
│  │  Web UI /    │   │  pihole CLI    │   │   gravity.sh     │  │
│  │  Dashboard   │   │  (list.sh,     │   │  (blocklist      │  │
│  │  (lighttpd + │   │   pihole cmd)  │   │   updater)       │  │
│  │  PHP/React)  │   └───────┬────────┘   └────────┬─────────┘  │
│  └──────┬───────┘           │                     │            │
│         │                   │  REST API calls      │ SQLite3    │
│         │            ┌──────▼─────────────────────▼─────────┐  │
│         └───────────►│           pihole-FTL                 │  │
│                      │  (DNS resolver + API server +        │  │
│                      │   SQLite3 engine + DHCP optional)    │  │
│                      └──────────────────┬────────────────── ┘  │
│                                         │                      │
│                              ┌──────────▼──────────┐           │
│                              │     gravity.db       │           │
│                              │   (SQLite3)          │           │
│                              │  groups, clients,    │           │
│                              │  adlists, domains,   │           │
│                              │  gravity table       │           │
│                              └─────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

Key processes running on the Pi-hole host:
| Process | Role |
|---|---|
| `pihole-FTL` | The DNS resolver, REST API server, and SQLite3 query engine — the single most important binary |
| `lighttpd` | Serves the web dashboard (UI) |
| `gravity.sh` | Shell script that downloads + compiles blocklists into `gravity.db` |
| `pihole` (CLI) | Shell wrapper invoking FTL's REST API for administrative tasks |

---

## 3. The Gravity Database (`gravity.db`)

**Location:** `/etc/pihole/gravity.db`
**Engine:** SQLite3 (accessed exclusively via `pihole-FTL`'s embedded SQLite3 — never raw system SQLite to avoid version mismatches)

This single file is the entire data store for Pi-hole's policy engine.

---

### 3.1 Core Tables

#### `group`
Defines named groups that act as policy containers.

```sql
CREATE TABLE "group" (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    enabled     BOOLEAN NOT NULL DEFAULT 1,
    name        TEXT UNIQUE NOT NULL,
    date_added  INTEGER NOT NULL DEFAULT (cast(strftime('%s', 'now') as int)),
    date_modified INTEGER NOT NULL DEFAULT (cast(strftime('%s', 'now') as int)),
    description TEXT
);
-- The Default group (id=0) is hardcoded and cannot be deleted
INSERT INTO "group" (id, enabled, name, description)
    VALUES (0, 1, 'Default', 'The default group');
```

- The `Default` group (id=0) always exists — it is protected by trigger `tr_group_zero` which re-inserts it if someone tries to delete it.
- All new clients, adlists, and domains are automatically placed into the Default group via INSERT triggers.

---

#### `client`
Represents a network device (identified by IP address). This is Pi-hole's concept of a "user".

```sql
CREATE TABLE client (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    ip          TEXT NOT NULL UNIQUE,   -- e.g. "192.168.1.25"
    date_added  INTEGER NOT NULL DEFAULT (cast(strftime('%s', 'now') as int)),
    date_modified INTEGER NOT NULL DEFAULT (cast(strftime('%s', 'now') as int)),
    comment     TEXT
);
```

> **Important:** Pi-hole identifies "users" by IP address only. There is no concept of a username, login session, or device identity beyond the IP. If DHCP leases change, group membership may break.

---

#### `adlist`
Stores URLs of external blocklists that gravity downloads periodically.

```sql
CREATE TABLE adlist (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    address         TEXT NOT NULL,        -- URL of the blocklist
    enabled         BOOLEAN NOT NULL DEFAULT 1,
    date_added      INTEGER NOT NULL DEFAULT (cast(strftime('%s', 'now') as int)),
    date_modified   INTEGER NOT NULL DEFAULT (cast(strftime('%s', 'now') as int)),
    comment         TEXT,
    date_updated    INTEGER,
    number          INTEGER NOT NULL DEFAULT 0,      -- domain count
    invalid_domains INTEGER NOT NULL DEFAULT 0,
    status          INTEGER NOT NULL DEFAULT 0,      -- 0=unknown,1=updated,2=unchanged,3=cached,4=failed
    abp_entries     INTEGER NOT NULL DEFAULT 0,
    type            INTEGER NOT NULL DEFAULT 0,      -- 0=blocklist, 1=allowlist (antigravity)
    UNIQUE(address, type)
);
```

The `type` field distinguishes between:
- `type=0` → entries go into the `gravity` table (block these domains)
- `type=1` → entries go into the `antigravity` table (always allow these domains, even if blocked elsewhere)

---

#### `domainlist`
Stores manually entered individual domains.

```sql
CREATE TABLE domainlist (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    type        INTEGER NOT NULL DEFAULT 0,
    domain      TEXT NOT NULL,
    enabled     BOOLEAN NOT NULL DEFAULT 1,
    date_added  INTEGER NOT NULL DEFAULT (cast(strftime('%s', 'now') as int)),
    date_modified INTEGER NOT NULL DEFAULT (cast(strftime('%s', 'now') as int)),
    comment     TEXT,
    UNIQUE(domain, type)
);
```

`type` values:
| Value | Meaning |
|---|---|
| `0` | Exact allowlist (always permit this domain) |
| `1` | Exact denylist (always block this domain) |
| `2` | Regex allowlist |
| `3` | Regex denylist |

---

#### `gravity`
The compiled block domain table — populated by `gravity.sh` at update time.

```sql
CREATE TABLE gravity (
    domain    TEXT NOT NULL,
    adlist_id INTEGER NOT NULL REFERENCES adlist (id)
);
-- Index for fast lookup at query time:
CREATE INDEX idx_gravity ON gravity (domain, adlist_id);
```

This table is **rebuilt from scratch on every `pihole -g` run**. It is never manually edited.

---

#### `antigravity`
The compiled allow domain table from adlists of type=1.

```sql
CREATE TABLE antigravity (
    domain    TEXT NOT NULL,
    adlist_id INTEGER NOT NULL REFERENCES adlist (id)
);
```

---

#### `info`
Key-value store for database metadata.

```sql
CREATE TABLE info (
    property TEXT PRIMARY KEY,
    value    TEXT NOT NULL
);
-- Current entries:
-- version          → schema version number (currently 20)
-- gravity_count    → number of unique blocked domains
-- updated          → unix timestamp of last gravity update
-- gravity_restored → backup restoration flag
```

---

### 3.2 Junction / Many-to-Many Tables

These three tables are the **heart of the group permission model**.

```sql
-- Which groups does each client belong to?
CREATE TABLE client_by_group (
    client_id INTEGER NOT NULL REFERENCES client (id) ON DELETE CASCADE,
    group_id  INTEGER NOT NULL REFERENCES "group" (id) ON DELETE CASCADE,
    PRIMARY KEY (client_id, group_id)
);

-- Which groups does each adlist apply to?
CREATE TABLE adlist_by_group (
    adlist_id INTEGER NOT NULL REFERENCES adlist (id) ON DELETE CASCADE,
    group_id  INTEGER NOT NULL REFERENCES "group" (id) ON DELETE CASCADE,
    PRIMARY KEY (adlist_id, group_id)
);

-- Which groups does each domain entry apply to?
CREATE TABLE domainlist_by_group (
    domainlist_id INTEGER NOT NULL REFERENCES domainlist (id) ON DELETE CASCADE,
    group_id      INTEGER NOT NULL REFERENCES "group" (id) ON DELETE CASCADE,
    PRIMARY KEY (domainlist_id, group_id)
);
```

A client can belong to **multiple groups**. If a domain is blocked in any group the client belongs to, it is blocked — unless an allowlist entry in one of their groups overrides it.

---

### 3.3 Query-Time Views

These views are what `pihole-FTL` actually queries on every DNS lookup. They join the policy tables through the group system so that every result is automatically scoped to the relevant groups.

```sql
-- Domains blocked via downloaded adlists, filtered by group membership
CREATE VIEW vw_gravity AS
    SELECT domain, adlist.id AS adlist_id, adlist_by_group.group_id AS group_id
    FROM gravity
    LEFT JOIN adlist_by_group ON adlist_by_group.adlist_id = gravity.adlist_id
    LEFT JOIN adlist           ON adlist.id = gravity.adlist_id
    LEFT JOIN "group"          ON "group".id = adlist_by_group.group_id
    WHERE adlist.enabled = 1
      AND (adlist_by_group.group_id IS NULL OR "group".enabled = 1);

-- Manually entered exact block domains, filtered by group
CREATE VIEW vw_denylist AS
    SELECT domain, domainlist.id AS id, domainlist_by_group.group_id AS group_id
    FROM domainlist
    LEFT JOIN domainlist_by_group ON domainlist_by_group.domainlist_id = domainlist.id
    LEFT JOIN "group"              ON "group".id = domainlist_by_group.group_id
    WHERE domainlist.enabled = 1
      AND (domainlist_by_group.group_id IS NULL OR "group".enabled = 1)
      AND domainlist.type = 1;

-- Manually entered exact allow domains (overrides blocks)
CREATE VIEW vw_allowlist AS    [ same join, type = 0 ]

-- Regex deny patterns
CREATE VIEW vw_regex_denylist AS  [ same join, type = 3 ]

-- Regex allow patterns
CREATE VIEW vw_regex_allowlist AS [ same join, type = 2 ]

-- Antigravity (bulk allowlists from adlists of type=1)
CREATE VIEW vw_antigravity AS
    SELECT domain, adlist.id AS adlist_id, adlist_by_group.group_id AS group_id
    FROM antigravity
    ...
    WHERE adlist.enabled = 1
      AND (adlist_by_group.group_id IS NULL OR "group".enabled = 1)
      AND adlist.type = 1;
```

**Query resolution order (enforced by FTL):**

```
1. vw_allowlist / vw_regex_allowlist / vw_antigravity  → ALLOW (highest priority)
2. vw_denylist  / vw_regex_denylist                    → BLOCK
3. vw_gravity                                          → BLOCK
4. No match                                            → forward to upstream DNS
```

---

### 3.4 Automatic Triggers

Triggers keep the junction tables consistent automatically:

```sql
-- New client → auto-add to Default group
CREATE TRIGGER tr_client_add AFTER INSERT ON client
    BEGIN
      INSERT INTO client_by_group (client_id, group_id) VALUES (NEW.id, 0);
    END;

-- New adlist → auto-add to Default group
CREATE TRIGGER tr_adlist_add AFTER INSERT ON adlist
    BEGIN
      INSERT INTO adlist_by_group (adlist_id, group_id) VALUES (NEW.id, 0);
    END;

-- New domain entry → auto-add to Default group
CREATE TRIGGER tr_domainlist_add AFTER INSERT ON domainlist
    BEGIN
      INSERT INTO domainlist_by_group (domainlist_id, group_id) VALUES (NEW.id, 0);
    END;

-- Protect the Default group — re-insert it if someone deletes it
CREATE TRIGGER tr_group_zero AFTER DELETE ON "group"
    BEGIN
      INSERT OR IGNORE INTO "group" (id, enabled, name) VALUES (0, 1, 'Default');
    END;

-- Cascade-delete junction records when a domain is deleted
CREATE TRIGGER tr_domainlist_delete AFTER DELETE ON domainlist
    BEGIN
      DELETE FROM domainlist_by_group WHERE domainlist_id = OLD.id;
    END;
```

---

## 4. The Group-Based Permission Model

This is the full data flow for per-client policy enforcement:

```
SETUP TIME:
  1. Create groups:   INSERT INTO "group" (name) VALUES ('Kids'), ('Adults'), ('IoT');
  2. Add clients:     INSERT INTO client (ip) VALUES ('192.168.1.10');
  3. Assign clients:  INSERT INTO client_by_group VALUES (client_id, group_id);
  4. Add adlists:     INSERT INTO adlist (address) VALUES ('https://...');
  5. Assign adlists:  INSERT INTO adlist_by_group VALUES (adlist_id, group_id);
  6. Run pihole -g    → downloads adlists, fills gravity table

QUERY TIME (per DNS request):
  1. DNS query arrives from 192.168.1.10 for "social.example.com"
  2. FTL looks up:    SELECT group_id FROM client_by_group
                      JOIN client ON client.id = client_by_group.client_id
                      WHERE client.ip = '192.168.1.10'
                      → returns [1, 3]  (e.g. "Kids" and "Default")
  3. FTL checks:      SELECT 1 FROM vw_allowlist
                      WHERE domain = 'social.example.com'
                      AND group_id IN (1, 3)
                      → no rows → not in allowlist
  4. FTL checks:      SELECT 1 FROM vw_gravity
                      WHERE domain = 'social.example.com'
                      AND group_id IN (1, 3)
                      → row found → BLOCKED, return 0.0.0.0
```

**A client in multiple groups inherits the union of all their blocklists but the highest-priority allowlist always wins.**

---

## 5. Gravity Update Pipeline (`gravity.sh`)

`pihole -g` triggers the full update pipeline:

```
Step 1: DNS check
        gravity_CheckDNSResolutionAvailable()
        Waits up to 120s for connectivity to raw.githubusercontent.com

Step 2: Migrate legacy files
        migrate_to_database()
        Imports old whitelist.txt / blacklist.txt / adlists.list
        into gravity.db if upgrading from pre-v5.0

Step 3: Prepare temp database
        pihole-FTL sqlite3 gravity.db_temp < gravity.db.sql
        Creates a brand-new empty gravity database in a temp file

Step 4: Copy user config to temp db  (gravity_copy.sql)
        Copies groups, clients, adlists, domainlists FROM old db
        INTO new temp db — preserving all user-configured policy

Step 5: Download each adlist
        For each enabled adlist URL:
          - curl with ETag / If-Modified-Since caching
          - Falls back to cached list if download fails
          - Calls pihole-FTL gravity parseList → inserts into gravity table

Step 6: Build search index
        CREATE INDEX idx_gravity ON gravity (domain, adlist_id)
        Needed for sub-millisecond lookups during DNS queries

Step 7: Optimize
        PRAGMA analysis_limit=0; ANALYZE
        Updates SQLite3 query planner statistics

Step 8: Atomic swap
        mv gravity.db → gravity_backups/gravity.db.1
        mv gravity.db_temp → gravity.db
        Keeps up to 10 rolling backups

Step 9: Cleanup
        Remove temp files, stale cache files
```

The atomic swap ensures Pi-hole is never in a partial state — the old database serves queries right up until the swap.

---

## 6. FTL – The DNS Resolver Engine

`pihole-FTL` is a custom binary (written in C, closed source but open binary) that is the single most important component. It does everything:

| Capability | Detail |
|---|---|
| DNS resolver | Listens on port 53, resolves queries, applies blocking |
| REST API server | Serves the Web UI and CLI at `http://localhost/api/` |
| DHCP server | Optional built-in DHCP (helps with static IP-to-client mapping) |
| SQLite3 engine | Embedded SQLite3 — all database access goes through FTL |
| Long-term statistics | Second database `pihole-FTL.db` stores per-query history |
| Query logging | Tracks every DNS query with client IP, domain, result, timestamp |

FTL is started as the `pihole` system user with specific Linux capabilities set:

```sh
setcap CAP_NET_BIND_SERVICE,CAP_NET_RAW,CAP_NET_ADMIN,\
       CAP_SYS_NICE,CAP_IPC_LOCK,CAP_CHOWN,CAP_SYS_TIME+eip \
       /usr/bin/pihole-FTL
```

This allows it to bind to port 53 and manage network interfaces without running as root.

**FTL discovers its own API URL dynamically** — the CLI scripts query FTL using a DNS CHAOS record:

```sh
dig +short -p 53 chaos txt local.api.ftl @127.0.0.1
# Returns: "http://localhost:80/api/" "https://localhost:443/api/"
```

---

## 7. REST API Layer (`api.sh`)

All administrative operations go through a token-authenticated REST API served by FTL.

**Auth flow:**

```
1. POST /api/auth  { "password": "...", "totp": null }
   → returns session ID (SID)

2. All subsequent requests include:  -H "sid: <SID>"

3. DELETE /api/auth  → invalidates session

4. Supports TOTP (2FA) for additional security
   CLI reads password from /etc/pihole/cli_pw to skip interactive auth
```

**Key API endpoints used by the shell scripts:**

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/auth` | POST | Authenticate, create session |
| `/api/auth` | DELETE | Logout |
| `/api/domains/{type}/{kind}` | POST | Add domain to allow/deny list |
| `/api/domains:batchDelete` | POST | Remove domains |
| `/api/domains/{type}/{kind}` | GET | List domains |

---

## 8. CLI Management (`list.sh` / `pihole` command)

The `pihole` command is the primary administrative interface.

```bash
# Block a domain for all groups (Default group)
pihole deny ads.example.com

# Allow a domain (overrides blocks)
pihole allow safe-site.com

# Regex block
pihole --regex ".*\.(ads|tracker)\..*"

# Wildcard block (converted to regex internally)
pihole --wild doubleclick.net
# becomes regex: (\.|^)doubleclick\.net$

# Remove from denylist
pihole deny remove ads.example.com

# List what's currently on the allowlist
pihole allow --list

# Update blocklists
pihole -g

# Show system status
pihole status
```

Internally, every `pihole deny` / `pihole allow` call authenticates with the REST API and POSTs to `/api/domains/{type}/{kind}`.

---

## 9. Database Migration System

Located in `advanced/Scripts/database_migration/gravity/`. Each file is a numbered SQL script (e.g. `1_to_2.sql`, `19_to_20.sql`) that upgrades the schema.

`upgrade_gravityDB()` in `gravity-db.sh` reads the `info.version` value and runs each migration script in sequence. This allows Pi-hole to safely upgrade from any schema version to the latest (currently v20).

Notable migration v19→v20 renamed views:
- `vw_whitelist` → `vw_allowlist`
- `vw_blacklist` → `vw_denylist`
- `vw_regex_whitelist` → `vw_regex_allowlist`
- `vw_regex_blacklist` → `vw_regex_denylist`

---

## 10. File Layout on Disk

```
/etc/pihole/
├── gravity.db              ← main policy + blocklist database
├── gravity_old.db          ← previous gravity database (kept if disk has space)
├── gravity_backups/
│   ├── gravity.db.1        ← rolling backups (up to 10)
│   └── gravity.db.2
├── listsCache/
│   └── list.<id>.<domain>.domains  ← cached downloaded blocklist files
│       list.<id>.<domain>.domains.sha1
│       list.<id>.<domain>.domains.etag
├── pihole-FTL.db           ← query log / statistics database (separate from gravity.db)
├── cli_pw                  ← hashed CLI password for non-interactive API auth
└── pihole-FTL.conf         ← FTL configuration (blocking mode, upstream DNS, etc.)

/opt/pihole/
├── gravity.sh              ← blocklist update script
├── utils.sh                ← shared shell utilities (getFTLConfigValue, etc.)
├── list.sh                 ← allow/deny domain management
├── api.sh                  ← REST API client helpers
└── COL_TABLE               ← terminal colour codes

/usr/bin/pihole-FTL         ← the main FTL binary
/usr/local/bin/pihole       ← CLI entry point

/etc/.pihole/
└── advanced/
    ├── Templates/
    │   ├── gravity.db.sql       ← database schema (used to create fresh gravity.db)
    │   └── gravity_copy.sql     ← SQL to copy config from old db to new temp db
    └── Scripts/
        └── database_migration/
            └── gravity/         ← schema upgrade scripts (1_to_2.sql … 19_to_20.sql)
```

---
---

# Research Section A — Pi-hole as a User Management Portal

---

## 11. What a Captive Portal Does vs. What Pi-hole Does

Before assessing whether Pi-hole can replace a captive portal, it is important to be precise about what each system does.

| Capability | Captive Portal | Pi-hole |
|---|---|---|
| **Identity** | Authenticates users (username/password, voucher, OAuth) | Identifies devices by IP address only — no login |
| **Access gate** | Blocks ALL traffic until the user authenticates | Blocks specific DNS queries — never stops all traffic |
| **Session management** | Creates timed sessions, tracks login/logout | No session concept — policy applies continuously |
| **Per-user policy** | Different rules per authenticated user | Different rules per IP (group assignment) |
| **Content filtering** | Can filter at HTTP/HTTPS level (with SSL inspection) | Filters only at DNS level (domain names only) |
| **Bandwidth control** | Can throttle, shape, or quota bandwidth per user | No bandwidth awareness whatsoever |
| **Redirect to login page** | HTTP 302 redirect to splash/login page | Sinkholed DNS returns 0.0.0.0 — no redirect |
| **Guest/public WiFi** | Core use case — verify ToS acceptance, log users | Not designed for this |
| **Audit trail** | Tracks which user accessed what and when | Tracks which IP queried which domain and when |

**Summary: These are fundamentally different tools.** A captive portal is an access control gateway. Pi-hole is a DNS filter.

---

## 12. Where Pi-hole CAN Replace Captive Portal Functions

Pi-hole has genuine value as a **partial user management layer** in these specific scenarios:

### A — Per-Client Policy Without Authentication

If your network uses **static IP addressing** or **DHCP reservations** (fixed IP per device/MAC address), you can:

1. Register each device as a `client` in Pi-hole by IP
2. Create named groups (`Kids`, `Staff`, `Guests`, `IoT`, etc.)
3. Assign different blocklists to each group
4. Assign devices to their appropriate group

**This gives you group-based internet policy without any login required.** Every device gets its own filtering profile just by being on the network with a known IP.

```
192.168.1.10  → "Kids" group    → has social media + adult content blocklists
192.168.1.20  → "Adults" group  → has only malware/ad blocklists
192.168.1.30  → "IoT" group     → has all outbound blocked except necessary services
```

### B — Blocking Specific Content Categories Per Group

Pi-hole natively supports assigning **separate blocklists** to each group. You can source category-specific blocklists (social media, adult content, gambling, ads, malware) and assign only the relevant ones to each group. This is a real, functional per-group content policy system.

### C — Time-Based Blocking (with cron)

While not built in, Pi-hole's REST API is fully scriptable. You can write cron jobs that:
- Move a client from one group to another at specific times
- Enable/disable specific blocklists on a schedule
- Implement "school hours" vs. "free time" policies

```bash
# 08:00 — move kids devices to strict group
pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "UPDATE client_by_group SET group_id=2 WHERE client_id=1;"

# 15:00 — move back to normal group
pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "UPDATE client_by_group SET group_id=1 WHERE client_id=1;"
```

### D — DNS Query Audit Log

Pi-hole logs every DNS query in `pihole-FTL.db` with:
- Client IP
- Domain queried
- Whether it was blocked or allowed
- Timestamp

This gives you a per-device browsing audit trail — useful for parental monitoring or basic compliance logging.

---

## 13. Where Pi-hole CANNOT Replace a Captive Portal

These are hard limitations based on how Pi-hole works architecturally:

### X — No Authentication / Login Wall

Pi-hole has **no mechanism to prompt a user to log in before accessing the internet**. It cannot:
- Display a splash page
- Collect usernames/passwords
- Issue time-limited access tokens
- Accept terms of service agreement
- Integrate with RADIUS, LDAP, Active Directory, or OAuth

If you need users to authenticate before getting internet access — **Pi-hole cannot do this alone**.

### Y — No IP-to-User Binding on Dynamic Networks

Pi-hole's entire group model is based on IP addresses. On networks where:
- DHCP is dynamic and IPs rotate
- Multiple users share a device
- Users connect from multiple devices

...the per-group policy will break or be incorrect. The system only knows "device 192.168.1.10" not "user Alice".

### Z — No HTTPS Content Filtering

Pi-hole operates on domain names only. It **cannot**:
- Block a specific URL path (e.g. block `youtube.com/shorts` but allow `youtube.com`)
- Inspect HTTPS content
- Filter based on HTTP headers, cookies, or page content
- Block specific search queries

If `youtube.com` is not blocked, Pi-hole allows all of YouTube — it cannot block a subcategory within a domain.

### W — No Bandwidth Quotas or Rate Limiting

Pi-hole has no awareness of bandwidth, connection speed, or data consumption. It cannot grant "1 hour of Netflix then throttle" or "100 MB daily data cap per user".

### V — Cannot Block Non-DNS Traffic

Devices that use hardcoded DNS servers (e.g. `8.8.8.8` in their OS settings) or DNS-over-HTTPS (DoH) built into browsers bypass Pi-hole entirely. A captive portal with firewall rules catches all traffic regardless.

---

## 14. Hybrid Architecture Recommendation

For a system that uses Pi-hole's blocklists but also needs real user management, the recommended architecture is:

```
┌──────────────────────────────────────────────────────────┐
│                    Network Gateway                        │
│                                                          │
│  ┌────────────────┐     ┌──────────────────────────┐    │
│  │ Captive Portal │     │       Pi-hole            │    │
│  │ (pfSense,      │────►│  (DNS filtering engine)  │    │
│  │  OpenWRT,      │     │                          │    │
│  │  NoDogSplash,  │     │  Groups map to user      │    │
│  │  Authelia)     │     │  roles/tiers from        │    │
│  │                │     │  captive portal           │    │
│  │  Handles:      │     │                          │    │
│  │  - Login/auth  │     │  Handles:                │    │
│  │  - Sessions    │     │  - Ad blocking           │    │
│  │  - Bandwidth   │     │  - Category filtering    │    │
│  │  - ToS accept  │     │  - Per-group policies    │    │
│  └────────────────┘     └──────────────────────────┘    │
│           │                           │                  │
│           │ On user login:            │                  │
│           │ 1. Auth user             │                  │
│           │ 2. Assign IP to          │                  │
│           │    Pi-hole group ────────►                  │
│           │    via REST API                             │
│           │ 3. Open firewall rule                       │
└──────────────────────────────────────────────────────────┘
```

**Integration point:** When a user logs into the captive portal and is assigned a tier (e.g. "Guest", "Premium", "Staff"), the captive portal makes a call to Pi-hole's REST API to add that client's IP to the appropriate Pi-hole group. This gives you both authenticated access control AND per-group DNS filtering.

**Pi-hole REST API call to set group membership:**

```bash
# On user login, captive portal calls Pi-hole API:
curl -X POST http://pihole/api/groups/clients \
  -H "sid: <session_token>" \
  -d '{"client": "192.168.1.55", "group": "premium-users"}'
```

---
---

# Research Section B — Blocklist Features

---

## 15. Blocklist Types and Sources

Pi-hole supports two types of adlist entries:

| `adlist.type` | Name | Behaviour |
|---|---|---|
| `0` | Blocklist | Domains populate the `gravity` table → blocked for assigned groups |
| `1` | Allowlist (Antigravity) | Domains populate the `antigravity` table → always allowed for assigned groups, even if appearing in a blocklist |

**Supported source protocols for blocklist URLs:**

| Protocol | Example |
|---|---|
| `https://` | `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts` |
| `http://` | Legacy, discouraged |
| `ftp://` / `ftps://` / `sftp://` | Remote FTP sources |
| `file://` | Local file on the Pi-hole host (must have `a+r` permissions) |

**Supported list formats** (parsed by `pihole-FTL gravity parseList`):
- Hosts file format: `0.0.0.0 ads.example.com`
- Domain-only format: `ads.example.com`
- ABP (Adblock Plus) format: `||ads.example.com^`

---

## 16. Per-Group Blocklist Assignment

This is the most powerful feature for differentiated filtering. The `adlist_by_group` junction table allows each blocklist to be assigned to any combination of groups.

**Practical example:**

```
Blocklist A: General Ads            → assigned to groups: Default, Kids, Adults
Blocklist B: Social Media           → assigned to groups: Kids only
Blocklist C: Adult Content          → assigned to groups: Kids only
Blocklist D: Malware / Phishing     → assigned to groups: Default, Kids, Adults, IoT
Blocklist E: IoT telemetry domains  → assigned to groups: IoT only
```

Result:
- A "Kids" group device → blocked by A + B + C + D
- An "Adults" device → blocked by A + D only (can access social media)
- An "IoT" device → blocked by D + E only

**All of this is controlled purely by the `adlist_by_group` table.** No additional configuration files.

---

## 17. Domain List Types (Allow / Deny / Regex)

Beyond bulk adlists, individual domains can be managed with four list types:

### Exact Allowlist (`type=0` in `domainlist`)
Domains that are **always permitted**, even if they appear in a blocklist. This is the highest priority rule.

```bash
pihole allow api.necessary-service.com
```

### Exact Denylist (`type=1`)
Domains that are **always blocked**, even if not in any adlist.

```bash
pihole deny social-media.com
```

### Regex Allowlist (`type=2`)
A regular expression pattern — any domain matching it is allowed.

```bash
pihole --allow-regex "^safe\..*\.com$"
```

### Regex Denylist (`type=3`)
A regular expression pattern — any domain matching it is blocked.

```bash
pihole --regex ".*\.(doubleclick|googlesyndication)\.net$"
```

### Wildcard (shorthand for Regex Deny)
Pi-hole converts wildcard entries to regex automatically:

```bash
pihole --wild ads.example.com
# Internally stored as regex: (\.|^)ads\.example\.com$
# This blocks ads.example.com AND any subdomain: x.ads.example.com
```

**Priority resolution order:**

```
Allowlist / Regex Allowlist / Antigravity  ← wins (always allow)
         ↓
Denylist / Regex Denylist                  ← second check
         ↓
Gravity (adlist-derived blocks)            ← third check
         ↓
No match → forward to upstream DNS         ← pass through
```

---

## 18. The Gravity Table and How Blocking Works at Query Time

At query time, FTL performs this lookup chain per-client (simplified):

```sql
-- Step 1: Get this client's groups
SELECT group_id FROM client_by_group
JOIN client ON client.id = client_by_group.client_id
WHERE client.ip = ?;

-- Step 2: Is it in the allowlist for these groups?
SELECT 1 FROM vw_allowlist
WHERE domain = ? AND group_id IN (?);
-- YES → ALLOW immediately

-- Step 3: Is it in the antigravity (bulk allowlist)?
SELECT 1 FROM vw_antigravity
WHERE domain = ? AND group_id IN (?);
-- YES → ALLOW immediately

-- Step 4: Is it in the manual denylist?
SELECT 1 FROM vw_denylist
WHERE domain = ? AND group_id IN (?);
-- YES → BLOCK

-- Step 5: Does a regex denylist pattern match?
-- (FTL caches compiled regex patterns in memory)
-- YES → BLOCK

-- Step 6: Is it in gravity (adlist-derived blocks)?
SELECT 1 FROM vw_gravity
WHERE domain = ? AND group_id IN (?);
-- YES → BLOCK

-- Step 7: No match → forward upstream → ALLOW
```

The `idx_gravity` index on `(domain, adlist_id)` makes step 6 extremely fast even with millions of entries.

---
---

# Research Section C — Quality of Service (QoS)

---

## 19. Does Pi-hole Have QoS?

**No. Pi-hole has zero QoS capability.**

This is not a limitation that can be worked around — it is a fundamental architectural constraint. Pi-hole operates exclusively at the **DNS application layer (Layer 7, name resolution only)**. QoS operates at **Layers 3–4 (IP packets, TCP/UDP flows, bandwidth shaping)**.

These two systems do not intersect.

---

## 20. What Pi-hole Controls vs. What QoS Controls

| Feature | Pi-hole | QoS |
|---|---|---|
| Block a domain name | YES | No |
| Block a category of sites | YES (via blocklists) | No (only by IP range) |
| Allow/deny specific devices | YES (by IP, via groups) | Partially |
| Limit bandwidth per user | **NO** | YES |
| Prioritise video calls over downloads | **NO** | YES |
| Set daily data quotas | **NO** | YES |
| Throttle a device after X MB | **NO** | YES |
| Pause internet at a scheduled time | Partially (via cron + block-all list) | YES |
| Shape traffic by application type | **NO** | YES |
| Inspect packet contents | **NO** | YES (DPI) |
| See real-time bandwidth per device | **NO** | YES |

Pi-hole **only** answers the question: "Should this domain name resolve?" Once a domain resolves to an IP, Pi-hole has no further involvement in the connection.

---

## 21. Recommended QoS Companions

To get QoS alongside Pi-hole, use one of these tools — they are all compatible with Pi-hole running as the DNS server:

### Router/Firewall-level QoS

| Tool | Platform | QoS Features |
|---|---|---|
| **pfSense / OPNsense** | x86 / ARM router | Traffic shaping, HFSC/PRIQ/CBQ queues, per-IP limits, Netflow |
| **OpenWRT** | Consumer routers | SQM (Smart Queue Management), cake, fq_codel, per-device limits |
| **MikroTik RouterOS** | MikroTik hardware | Mangle + Queue Trees, per-user bandwidth limits, burst handling |
| **VyOS** | x86 / VM | Traffic policies, rate limiting, DSCP marking |

### Dedicated Bandwidth Management

| Tool | Description |
|---|---|
| **tc (Linux traffic control)** | Kernel-level tool: HTB queues, rate limiting, fq_codel — works on any Linux box |
| **wondershaper** | Simple wrapper around `tc` for per-interface rate limiting |
| **NetLimiter** (Windows) | Per-process / per-app bandwidth control |
| **ntopng** | Network traffic monitoring + flow analysis (works with Pi-hole for combined visibility) |

### Combined Pi-hole + QoS on a Single Host

If running Pi-hole on a Linux gateway (not just a sidecar DNS server), you can combine:

```
Pi-hole (DNS filtering)  +  tc/HTB (bandwidth shaping)  +  iptables/nftables (firewall rules)
```

This gives you:
- Pi-hole for per-group DNS blocking
- `tc` for per-IP bandwidth limits and queue priorities
- `iptables` for blocking specific TCP/UDP ports and enforcing DNS redirection (forcing all devices to use Pi-hole)

**Forcing all devices to use Pi-hole DNS (recommended iptables rule):**

```bash
# Redirect all DNS queries to Pi-hole (prevents bypassing with 8.8.8.8)
iptables -t nat -A PREROUTING -i br0 ! -d 192.168.1.1 \
  -p udp --dport 53 -j DNAT --to-destination 192.168.1.1:53
iptables -t nat -A PREROUTING -i br0 ! -d 192.168.1.1 \
  -p tcp --dport 53 -j DNAT --to-destination 192.168.1.1:53
```

This ensures no device can bypass Pi-hole by setting its own DNS server, which is a critical companion rule to make Pi-hole's per-group filtering authoritative.

---

*End of Wiki — Pi-hole Architecture, User Management Research, Blocklist Features, QoS Research*
