# How Pi-hole Works
### A Deep Dive: From Layman's Overview to Hands-On Experimentation

---

## Part 1: What Is Pi-hole? (Plain English)

Imagine every website you visit is like a house on a street. When you type `google.com` into your browser, your computer asks a "phone book service" called DNS (Domain Name System): *"What's the address for google.com?"* The DNS service looks it up and says *"It's at 142.250.80.46"* and your browser goes there.

Now imagine that every ad on the internet — every banner, every tracker, every surveillance pixel — is also a house on that same street. They have names too: things like `ads.doubleclick.net` or `tracking.facebook.com`.

**Pi-hole is a DNS filter that sits between your devices and the internet's phone book.** When your browser asks *"Where is ads.doubleclick.net?"* Pi-hole intercepts that question and says *"That address doesn't exist"* — before your browser can ever connect. The ad never loads, the tracker never fires, the page loads faster.

What makes Pi-hole powerful:
- It works at the **network level**, meaning every device on your WiFi is protected — phones, TVs, smart fridges — without installing any app
- It blocks things **before they load**, not after (unlike browser extensions that block after download)
- It's **free, open-source**, and runs on a $10 Raspberry Pi
- The block lists are crowd-sourced and community-maintained, updated regularly
- You get a **dashboard** showing every DNS query every device on your network ever made

---

## Part 2: Inventory of Components

Here is every major piece of the repository and what it does:

```
pi-hole/
├── pihole                          ← Main CLI entry point (the "pihole" command)
├── gravity.sh                      ← Heart of the system: downloads & compiles blocklists
│
├── automated install/
│   ├── basic-install.sh            ← Full system installer (handles deps, users, systemd)
│   └── uninstall.sh               ← Complete removal script
│
├── advanced/
│   ├── Scripts/
│   │   ├── api.sh                  ← Authentication + REST API calls to FTL daemon
│   │   ├── list.sh                 ← Add/remove/list allow & deny domain entries
│   │   ├── query.sh               ← Search whether a domain is on a blocklist
│   │   ├── utils.sh               ← Shared helper functions (colors, checks, etc.)
│   │   ├── update.sh              ← Check + apply updates to Pi-hole components
│   │   ├── version.sh             ← Display installed vs available versions
│   │   ├── updatecheck.sh         ← Cron task: poll GitHub for new releases
│   │   ├── piholeDebug.sh         ← Collect full system diagnostic report
│   │   ├── piholeLogFlush.sh      ← Rotate / clear DNS query logs
│   │   ├── piholeNetworkFlush.sh  ← Clear the ARP/network cache table
│   │   ├── piholeCheckout.sh      ← Switch between dev/release branches
│   │   ├── COL_TABLE              ← Terminal color codes (sourced by all scripts)
│   │   └── database_migration/
│   │       └── gravity-db.sh      ← Incremental SQLite schema upgrade logic
│   │
│   └── Templates/
│       ├── gravity.db.sql          ← SQLite schema: all tables, views, triggers
│       ├── gravity_copy.sql        ← SQL to copy gravity data between databases
│       ├── pihole.cron             ← Cron job definitions (gravity, log, version checks)
│       ├── pihole-FTL.service      ← Systemd unit file for the FTL DNS daemon
│       └── logrotate              ← Log rotation configuration
│
├── test/                           ← Docker-based test suite (Python/pytest)
│   ├── Dockerfile.*               ← Containers for each supported Linux distro
│   ├── tox.ini                    ← Test runner configuration
│   └── *.py                       ← Test files and fixtures
│
└── manpages/                       ← Unix manual pages for `man pihole`
```

**External components (separate repos, not in this code):**
- **pihole-FTL** — the compiled C binary that IS the DNS server. This repo manages it; FTL runs it.
- **pi-hole/web** — the web dashboard (PHP/JavaScript). This repo serves it; web is the UI.

---

## Part 3: How It All Works — A Detailed Walkthrough

### 3.1 The DNS Blocking Loop

Every time a device on your network makes a DNS request, here is the exact sequence of events:

```
Your Phone asks: "Where is ads.example.com?"
        ↓
pihole-FTL receives the UDP packet on port 53
        ↓
FTL checks: is "ads.example.com" in the gravity table? → YES → BLOCK
        ↓
FTL checks: is it in the allowlist (antigravity)? → If YES → OVERRIDE, allow it
        ↓
FTL checks: is it in the exact denylist? → If YES → BLOCK
        ↓
FTL checks: does it match any regex deny pattern? → If YES → BLOCK
        ↓
FTL checks: does it match any regex allow pattern? → If YES → ALLOW
        ↓
None matched → Forward to upstream DNS (e.g., 1.1.1.1) → Return real answer
        ↓
All queries logged to /var/log/pihole/pihole.log + SQLite FTL database
```

**Blocking modes** (configurable via `dns.blocking.mode`):
- `NXDOMAIN` — tells the device the domain flat-out doesn't exist
- `NULL` — returns 0.0.0.0 (nothing is there)
- `IP` — returns your Pi-hole's own IP (useful for custom block pages)
- `NODATA` — returns an empty valid response

### 3.2 The Gravity System — Building the Blocklist

`gravity.sh` is the most complex and important script. It runs weekly (Sunday 01:59 cron) or manually via `pihole -g`. Here's exactly what it does in order:

**Step 1: Preparation**
- Creates a fresh temporary SQLite database from the schema template
- Copies the existing valid gravity data into the temp database
- Makes up to 10 rolling backups of the previous good database

**Step 2: Download Phase**
```
For each URL in the adlist table (enabled=1):
    1. Check if the source domain itself is blocked by Pi-hole
       → If yes, bypass Pi-hole and use a direct upstream DNS to fetch it
    2. Send HTTP request with ETag / If-Modified-Since headers
       → This avoids re-downloading lists that haven't changed
    3. Compute SHA1 of downloaded content
       → Compare to cached version; skip processing if identical
    4. Save to /etc/pihole/listsCache/list.{id}.{domain}.domains
    5. Call: pihole-FTL gravity parseList <file> <db> <id>
       → FTL parses the raw file and inserts valid domains into gravity table
    6. Record download status:
       1 = updated, 2 = unchanged, 3 = cached/fallback, 4 = failed
```

**Step 3: Index Building**
```sql
CREATE INDEX idx_gravity ON gravity (domain, adlist_id);
```
This is what makes lookups fast. Without this index, checking 5 million domains on every DNS query would be impossibly slow.

**Step 4: Atomic Swap**
- The new database is validated and VACUUM'd
- The old database is backed up
- The new database is moved into place atomically (no downtime)
- FTL is signaled via SIGRTMIN to reload its lists (no cache flush, no restart)

### 3.3 The Database — gravity.db

The entire blocking system lives in a single SQLite file at `/etc/pihole/gravity.db`.

**Key tables:**

| Table | What it stores |
|-------|---------------|
| `gravity` | ~millions of blocked domains, each linked to its source adlist |
| `antigravity` | Allowlist domains that override gravity |
| `adlist` | URLs of blocklist sources + their status, counts, timestamps |
| `domainlist` | Exact and regex allow/deny entries added manually by the user |
| `group` | Named groups for per-client or per-device rules |
| `info` | Metadata: schema version, last gravity update time, domain count |

**Key views (pre-built queries FTL uses at runtime):**

| View | Purpose |
|------|---------|
| `vw_gravity` | Only enabled, group-filtered gravity entries |
| `vw_antigravity` | Only enabled allowlist entries |
| `vw_denylist` | Exact manual deny entries |
| `vw_allowlist` | Exact manual allow entries |
| `vw_regex_denylist` | Regex deny patterns |
| `vw_regex_allowlist` | Regex allow patterns |
| `vw_adlist` | Only enabled blocklist sources |

**Schema versioning:**
The `info` table stores `version = 20` (current). `gravity-db.sh` contains migration SQL for every version from 1→20. Every time gravity runs, it checks the version and applies any missing migrations automatically.

### 3.4 The CLI — `pihole` Command

The `pihole` script at `/usr/local/bin/pihole` is a Bash router. Every subcommand dispatches to a specialized handler:

```bash
pihole -g               → gravity.sh          (rebuild blocklists)
pihole -q domain        → query.sh            (is this domain blocked?)
pihole allow domain     → list.sh + api.sh    (whitelist a domain)
pihole deny domain      → list.sh + api.sh    (blacklist a domain)
pihole -d               → piholeDebug.sh      (collect diagnostics)
pihole -f               → piholeLogFlush.sh   (clear logs)
pihole -up              → update.sh           (update Pi-hole)
pihole status           → inline + FTL check  (is FTL running?)
pihole api              → api.sh              (raw API access)
```

### 3.5 The API Layer

Pi-hole's FTL daemon exposes a local REST API (HTTP/HTTPS). The `api.sh` script handles auth and communication. Here's how auth works:

1. Probe `local.api.ftl` via DNS CHAOS query to discover the API port
2. Try a request — if `401 Unauthorized` comes back, authentication is needed
3. Read `/etc/pihole/cli_pw` if the file exists and is readable
4. Otherwise, prompt the user for their web password
5. POST credentials to `/api/auth`, receive a session token
6. Use that token in all subsequent requests
7. DELETE the session token on logout

Key endpoints:
```
GET  /api/search/{domain}           → Is this domain blocked?
POST /api/domains/{type}/{kind}     → Add allow/deny entry
DEL  /api/domains/{type}/{kind}     → Remove entry
POST /api/action/flush/network      → Clear network table
GET  /api/stats/summary             → Blocking statistics
```

### 3.6 Scheduling — What Runs Automatically

```
Cron job                              When it runs
──────────────────────────────────────────────────────────────────────
pihole updateGravity                  Sunday 01:59 (weekly blocklist update)
pihole flush once quiet               Daily 00:00 (log rotation)
pihole updatechecker                  Daily 17:59 UTC (check GitHub for updates)
pihole updatechecker reboot           On every system boot (+30s delay)
logrotate (pihole config)             On every system boot
```

### 3.7 The Full Data Path — Request to Response

```
[Device]──DNS query──►[pihole-FTL:53]
                              │
                    ┌─────────▼──────────┐
                    │  gravity.db lookup  │
                    │  (SQLite, indexed)  │
                    └─────────┬──────────┘
                       ┌──────┴──────┐
                    BLOCKED       NOT BLOCKED
                       │              │
                  Return          Forward to
                NXDOMAIN/NULL    upstream DNS
                       │              │
                    ┌──┴──────────────┴──┐
                    │   Log the query     │
                    │  (pihole.log +      │
                    │   FTL database)     │
                    └─────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Web Dashboard     │
                    │  (reads FTL db)     │
                    │  Shows stats/graphs │
                    └────────────────────┘
```

---

## Part 4: Experimentation — Testing Our Understanding

> **The most important principle in this section:**
> Reading code and documentation gives us a *hypothesis* about how a system works. Only by running the system, making changes, and observing results do we discover where our understanding is correct — and where it's wrong. Every experiment is also a test of this document's accuracy.

### 4.1 Experiments to Run Before Making Changes

Run these to verify your mental model before modifying anything:

**Experiment 1: Watch a DNS query being blocked in real time**
```bash
# Terminal 1: Watch the live query log
sudo tail -f /var/log/pihole/pihole.log

# Terminal 2: Query a known-blocked domain
nslookup ads.doubleclick.net 127.0.0.1
# Expected: returns 0.0.0.0 or NXDOMAIN
# Verify: Terminal 1 should show the query with "blocked" status
```

**Experiment 2: Inspect the gravity database directly**
```bash
# How many domains are blocked?
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db "SELECT COUNT(*) FROM gravity;"

# What blocklists are configured?
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT id, address, enabled, number FROM adlist ORDER BY number DESC;"

# Is a specific domain in the blocklist?
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT domain, adlist_id FROM gravity WHERE domain = 'ads.doubleclick.net';"
```

**Experiment 3: Test the query command**
```bash
pihole -q ads.doubleclick.net
# Expected: shows which list it matched and whether it's blocked or allowed

pihole -q --all google.com
# Expected: shows google.com is NOT blocked, passes through
```

**Experiment 4: Manually add and verify a deny entry**
```bash
# Add a fake domain to the denylist
pihole deny test-block-this-domain.example.com

# Verify it's in the database
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT domain, type, enabled FROM domainlist WHERE domain LIKE '%test-block%';"

# Test it resolves as blocked
nslookup test-block-this-domain.example.com 127.0.0.1
# Expected: NXDOMAIN or 0.0.0.0

# Clean up
pihole allow test-block-this-domain.example.com  # or remove from denylist
```

**Experiment 5: Examine a blocklist cache file**
```bash
ls -la /etc/pihole/listsCache/
# See the cached list files

sudo head -50 /etc/pihole/listsCache/list.1.*.domains
# Raw domain format that gravity.sh downloads and parses
```

### 4.2 Common Discovery Moments (Where Assumptions Break)

After running experiments, you'll likely discover:

- **"pihole-FTL" isn't just a process name** — it's a multi-function binary used for DNS, SQLite access, and list parsing. The same binary does many things depending on arguments.
- **Blocking happens in FTL, not in gravity.sh** — gravity.sh only builds the database. FTL reads it. This means you can manually edit `gravity.db` and send `SIGRTMIN` to FTL and blocking changes immediately without rerunning gravity.
- **The allowlist has higher priority than gravity** — if a domain is in both `gravity` and `antigravity`, it's allowed. The view logic enforces this.
- **Regex rules are evaluated last** — the order of precedence is: exact allowlist → exact denylist → gravity (blocklist) → regex patterns.

### 4.3 How to Safely Test Modifications

Before making any changes to a production system:

```bash
# Always check what gravity looks like before and after
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT COUNT(*) FROM gravity;" > /tmp/before.txt

# Make your change...

sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT COUNT(*) FROM gravity;" > /tmp/after.txt

diff /tmp/before.txt /tmp/after.txt
```

---

## Part 5: Feature — Snort Threat Intelligence Integration

### 5.1 What Is Snort's Threat List?

Snort is Cisco's open-source network intrusion detection system. They publish a free "community ruleset" that includes threat intelligence: known malicious IPs, C2 (command-and-control) servers, phishing domains, malware delivery URLs, and botnet infrastructure.

The **Snort Community Blocklist** relevant to Pi-hole is a plain-text domain blocklist that can be pulled as a URL and fed into gravity — the same way you'd add any other blocklist source.

The free Snort-compatible threat domain feeds include:
- **abuse.ch URLhaus** — malware distribution URLs/domains (free, no registration)
- **abuse.ch Feodo/Bazaar** — botnet C2 domains
- **Emerging Threats** — open ruleset domains (free edition)

For this workflow, we will use **abuse.ch URLhaus domains** as a concrete, free, no-registration-required example because it:
1. Publishes a Pi-hole compatible format (one domain per line)
2. Updates multiple times per day
3. Requires no API key
4. Has a stable, documented URL

The concept and code generalizes to any Snort-compatible feed.

### 5.2 Feature PRD: Snort/Threat Intelligence Feed Integration

---

**Feature:** Automated Threat Intelligence Blocklist Sync
**Version:** 1.0
**Status:** Draft → Implementation

#### Problem Statement

Pi-hole's default blocklists are excellent at blocking ads and trackers, but they do not include active threat intelligence: known malware domains, phishing infrastructure, botnet C2 servers, and newly-registered malicious domains. These threats exist on a different time scale (hours, not weeks) and require different data sources.

#### Goal

Enable Pi-hole to automatically pull from Snort-compatible and threat intelligence domain feeds, treating them as first-class gravity sources that are kept fresh on a faster cadence than the standard weekly gravity update.

#### Success Criteria

1. At least one threat intelligence domain list is added as a gravity adlist source
2. Domains from that list appear in `gravity.db` and are actively blocking
3. The list is verified to update on schedule (at minimum weekly, ideally daily)
4. A test domain from the list resolves as blocked from a client device
5. The feature survives a full `pihole -g` (gravity rebuild) without errors

#### Non-Goals

- Snort IDS rule integration (Pi-hole is DNS-only; IP/port/payload rules are out of scope)
- Paid Snort subscription rules
- Custom parsing of Snort `.rules` file format

#### Sources

| Feed | URL | Format | Update Cadence | Notes |
|------|-----|--------|----------------|-------|
| URLhaus Domains | `https://urlhaus.abuse.ch/downloads/hostfile/` | hosts file | Every few hours | Free, no key |
| Emerging Threats (Open) | `https://rules.emergingthreats.net/fwrules/emerging-Block-IPs.txt` | IPs only | Daily | IPs not domains |
| abuse.ch Malware Domains | `https://malware-filter.pages.dev/urlhaus-filter-hosts.txt` | hosts file | Daily | Mirror, very stable |

**Recommended primary source:** `https://urlhaus.abuse.ch/downloads/hostfile/`

#### Implementation Steps

1. Add the threat feed URL to the gravity adlist via `pihole` CLI
2. Trigger gravity rebuild to pull and compile the list
3. Verify domains from the list are in gravity.db
4. Add a more frequent cron job (daily instead of weekly) for threat feed refresh
5. Verify the system survives list update on next run

#### Acceptance Test

```bash
# A domain known to be on the URLhaus list (confirmed at time of writing):
nslookup <domain-from-urlhaus-list> 127.0.0.1
# → Must return NXDOMAIN or 0.0.0.0
```

---

### 5.3 Implementation

#### Step 1: Add the Threat Intelligence Feed to Pi-hole

```bash
# Add URLhaus malware domain blocklist as a new gravity source
# Type 0 = blocking list (as opposed to 1 = allowlist)
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "INSERT INTO adlist (address, enabled, comment) VALUES (
    'https://urlhaus.abuse.ch/downloads/hostfile/',
    1,
    'URLhaus abuse.ch - active malware distribution domains'
  );"

# Verify it was added
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT id, address, enabled, comment FROM adlist ORDER BY id DESC LIMIT 3;"
```

Alternatively, if you have the web interface running:
1. Go to `http://pi.hole/admin` → Login
2. Adlists → Add new adlist
3. Paste URL: `https://urlhaus.abuse.ch/downloads/hostfile/`
4. Comment: `URLhaus - Active malware domains`
5. Click Add

#### Step 2: Rebuild Gravity to Pull the New List

```bash
# This will download all lists including your new one
# -g = updateGravity
sudo pihole -g

# Watch the output — you should see your new list being downloaded
# and a domain count for it
```

Expected output snippet:
```
[i] URLhaus abuse.ch - active malware distribution domains
    Status: Updated          Domains: 15234
```

#### Step 3: Verify the List Is Active

```bash
# Check domain count from the new adlist
# First get the adlist ID you added
ADLIST_ID=$(sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT id FROM adlist WHERE address LIKE '%urlhaus%';")

echo "Adlist ID: $ADLIST_ID"

# Count domains from that specific list
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT COUNT(*) FROM gravity WHERE adlist_id = $ADLIST_ID;"

# Check the status field (1=updated, 2=unchanged, 3=cached, 4=failed)
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT status, number, date_updated FROM adlist WHERE id = $ADLIST_ID;"
```

#### Step 4: Test That Blocking Is Active

URLhaus publishes its list actively. To test without relying on a specific domain (which may be removed from the list):

```bash
# Method 1: Query a domain from the list cache file
ADLIST_ID=$(sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT id FROM adlist WHERE address LIKE '%urlhaus%';")

# Find the cached list file (format: list.{id}.{domain}.domains)
ls /etc/pihole/listsCache/ | grep "^list.${ADLIST_ID}."

# Get the first domain from it
TEST_DOMAIN=$(sudo head -5 /etc/pihole/listsCache/list.${ADLIST_ID}.*.domains | \
  grep -v '^#' | grep -v '^$' | head -1)

echo "Testing domain: $TEST_DOMAIN"

# Test DNS resolution — should be blocked
nslookup "$TEST_DOMAIN" 127.0.0.1
# Expected: 0.0.0.0 or NXDOMAIN

# Also test via pihole query command
pihole -q "$TEST_DOMAIN"
```

#### Step 5: Set Up More Frequent Updates (Daily for Threat Feeds)

The standard gravity cron runs weekly. Threat intelligence needs daily updates. Create a supplemental cron job:

```bash
# View the current cron
cat /etc/cron.d/pihole

# Add a daily gravity update specifically for threat intelligence
# This runs gravity every day at 06:00 AM
sudo tee /etc/cron.d/pihole-threat-intel << 'EOF'
# Daily gravity refresh for threat intelligence feeds
# Runs at 06:00 every day (offset from standard 01:59 Sunday run)
0 6 * * * root /usr/local/bin/pihole updateGravity >> /var/log/pihole/gravity-daily.log 2>&1
EOF

sudo chmod 644 /etc/cron.d/pihole-threat-intel

# Verify cron was created
cat /etc/cron.d/pihole-threat-intel
```

> **Note on performance:** Running `pihole -g` daily downloads ALL lists, not just the new one. Pi-hole's ETag/If-Modified-Since logic means unchanged lists are skipped at the network level, but the database is still rebuilt. On a Raspberry Pi this takes ~60-90 seconds. Acceptable for a daily cadence.

#### Step 6: Verify the Complete Setup

```bash
# Full verification checklist:

echo "=== 1. Adlist registered ==="
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT id, address, enabled, number, status FROM adlist WHERE address LIKE '%urlhaus%';"

echo "=== 2. Domains in gravity ==="
ADLIST_ID=$(sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT id FROM adlist WHERE address LIKE '%urlhaus%';")
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT COUNT(*) as domain_count FROM gravity WHERE adlist_id = $ADLIST_ID;"

echo "=== 3. Total gravity count ==="
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db "SELECT COUNT(*) FROM gravity;"

echo "=== 4. Last gravity update ==="
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT value FROM info WHERE property = 'updated';"

echo "=== 5. Cron jobs active ==="
ls -la /etc/cron.d/pihole*

echo "=== 6. FTL is running ==="
pihole status
```

---

## Part 6: Testing — Where Real Understanding Comes From

> **The meta-lesson of this entire document:**
> Everything written here — every architecture description, every data flow diagram, every "expected output" — is a *hypothesis*. It was derived by reading the source code and reasoning about what it should do. But code has bugs. Documentation has gaps. Systems have state. The environment matters.
>
> **The AI that produced this document can be wrong.** Not because it's lying, but because:
> - It may misread code paths that have conditional branches
> - It may describe how a component *should* work rather than how it *does* work with your specific version
> - It may miss environmental dependencies (Linux distro, FTL version, database state)
> - Its training data may include outdated versions of this codebase
>
> **Every experiment you run is a test of this document.** When results match — great. When they don't — that's even better. Document the discrepancy. Correct the hypothesis. Update this file.

### 6.1 The Experimental Loop

```
Read the code / this doc
       ↓
Form a hypothesis:
"If I do X, I expect Y"
       ↓
Run the experiment
       ↓
Observe the actual result
       ↓
Does it match?
  YES → Hypothesis confirmed → Move to next
  NO  → Find the gap → Correct understanding → Update docs → Repeat
```

### 6.2 Experiments for the Snort Integration Feature

**Test 1: Does the URLhaus URL return the expected format?**
```bash
# Download just the first 20 lines to check format
curl -s --max-time 10 'https://urlhaus.abuse.ch/downloads/hostfile/' | head -20

# Expected: hosts file format like:
# 0.0.0.0 malicious-domain.com
# or just:
# malicious-domain.com
```

**Test 2: Does gravity correctly parse the format?**
```bash
# After pihole -g, check for parse errors
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT invalid_domains FROM adlist WHERE address LIKE '%urlhaus%';"
# Expected: a low number (some entries in the feed are IPs, not domains, which get rejected)
```

**Test 3: Does the daily cron job actually run without errors?**
```bash
# Manually trigger it as root (simulates cron execution)
sudo /usr/local/bin/pihole updateGravity >> /tmp/gravity-test.log 2>&1
cat /tmp/gravity-test.log
# Expected: no errors, updated counts shown, exit 0
```

**Test 4: Does an allowlisted domain override a threat feed block?**
```bash
# Add a domain from the urlhaus list to the allowlist
TEST_DOMAIN="<domain-from-urlhaus>"
pihole allow "$TEST_DOMAIN"

# Test resolution — should now be ALLOWED despite being in blocklist
nslookup "$TEST_DOMAIN" 127.0.0.1
# Expected: actual IP returned (not 0.0.0.0)

# Remove the allowlist entry
pihole allow --delete "$TEST_DOMAIN"
```

**Test 5: Verify gravity survives the list being temporarily unavailable**
```bash
# Simulate a failed download by using a bad URL temporarily
ADLIST_ID=$(sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT id FROM adlist WHERE address LIKE '%urlhaus%';")

# Change to a broken URL
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "UPDATE adlist SET address = 'https://urlhaus.abuse.ch/downloads/hostfile_BAD/' WHERE id = $ADLIST_ID;"

# Run gravity
sudo pihole -g

# Check status — should be 4 (failed) but gravity should still succeed overall
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT status FROM adlist WHERE id = $ADLIST_ID;"
# Expected: 4 (failed download), but cached domains still remain

# Fix the URL back
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "UPDATE adlist SET address = 'https://urlhaus.abuse.ch/downloads/hostfile/' WHERE id = $ADLIST_ID;"
```

### 6.3 Discrepancy Journal

Use this section to record what you find when reality differs from expectation:

| Test | Expected | Actual | Hypothesis Correction |
|------|----------|--------|----------------------|
| *(run tests and fill this in)* | | | |

---

## Part 7: Quick Reference

```bash
# Rebuild blocklists
sudo pihole -g

# Query a domain
pihole -q domain.com

# Add to denylist (block manually)
pihole deny domain.com

# Add to allowlist (unblock)
pihole allow domain.com

# Check Pi-hole status
pihole status

# View stats
pihole -c   # chronometer (live terminal dashboard)

# Inspect gravity database
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db

# Flush DNS logs
sudo pihole -f

# Full debug report
sudo pihole -d

# Check for updates
pihole -up --check-only

# Tail DNS query log live
sudo tail -f /var/log/pihole/pihole.log
```

---

*This document is a living artifact. As you run experiments and discover corrections, update the "Discrepancy Journal" and revise any sections that proved incorrect. The document's accuracy improves with every experiment you run.*
