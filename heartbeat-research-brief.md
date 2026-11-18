Research is in. I’m writing `readme.md` as the product brief and `build.md` as the full build spec, including the agent layer. The files are the deliverable.

```markdown readme.md
# Heartbeat — Research Brief

**What this is:** a research brief for cloning the *idea* of a cron / heartbeat monitoring service (Healthchecks.io, with Cronitor as the paid comparison), then adding an AI-agent layer on top.

**What this is not:** permission to copy their source, trademarks, copy, or visual design. Healthchecks is BSD-3 and *can* be forked if you keep the license notice. Cronitor is proprietary. This brief is for building your own product that does the same job.

**Researched:** 2026-11-18. Pricing and stats go stale. Recheck vendor pages before you quote them in marketing.

---

## 1. Direct answer

A heartbeat monitor is a **dead man's switch**. The job pings a unique URL when it finishes. If the ping does not arrive inside the expected window plus a grace period, the service alerts you. Silence is the failure.

That is the whole product. Everything else (cron expressions, start/fail signals, logs, Slack, badges, status pages) is packaging around that one idea.

**What to copy, in order:**

1. The state machine: `new → up → late (grace) → down → up`, plus `paused`.
2. A ping URL that is a secret, answering `200 OK` in a few milliseconds, even when the database is slow.
3. Three schedule types: simple period, cron, systemd OnCalendar.
4. Optional `/start`, `/fail`, `/<exitcode>`, and a log body (first 100 KB).
5. Per-check alert channels, not one global inbox.
6. A management API so agents (and Terraform, and CI) can create checks without the UI.
7. Projects, roles, badges, email reports.

**What to add so you are not a worse Healthchecks:** an agent that reads the job, writes the ping wrapper, picks period and grace from history, files a ticket with the last log when a check goes down, and proposes a fix. Neither Healthchecks nor Cronitor does that as the product. Cronitor has a marketing page for monitoring *agent runs*; it does not diagnose your backup script.

**Do not build in v1:** global uptime probes, RUM, SAML, phone calls, WhatsApp. Cronitor's extra surface is why it costs more and why Healthchecks still wins the "just tell me the cron died" buyer.

---

## 2. How the product actually works

### 2.1 The contract

| Piece | Meaning |
|---|---|
| Check | One monitored job. Has a name, slug, tags, description, schedule, grace, ping URL, and a set of channels. |
| Ping | HTTP (or email) from the job. Success, start, fail, log, or exit code. |
| Period / schedule | When the next success is due. |
| Grace | Extra wait before "late" becomes "down" and alerts fire. Also the max runtime if you send `/start`. |
| Integration / channel | Where the alert goes. Email, Slack, webhook, and so on. |
| Project | A bucket of checks + channels + team + API keys. Billing limit is on the *account*, shared across projects you own. |

Healthchecks states the job in one sentence: it listens for pings, stays silent while they arrive on time, and alerts when one does not. It is explicitly **not** a website prober, an APM, or a log aggregator. [1]

Typical jobs: backups, task queues, replication checks, report scripts, imports, antivirus scans, DDNS updates, SSL renewal. [1]

### 2.2 States

Every check is in exactly one state. [1] [25]

| State | Meaning | Alert? |
|---|---|---|
| `new` | Created, never pinged. | No. |
| `up` | Last success arrived on time. | Recovery alert if it was down. |
| `late` / `grace` | Success is due, grace has not elapsed. API field is `grace`. | No. |
| `down` | Grace elapsed, or an explicit fail, or start with no success inside grace. | Yes, on the transition into down. |
| `paused` | Monitoring off. Sticky if `manual_resume` is set. | No. |

A running job (last signal was `start`, no matching success yet) is shown as three dots under the icon. It does not replace the status. [1]

**Simple schedule.** Period = 1 hour, grace = 5 minutes, last success at 12:00. Late at 13:00. Down, and alerts, at 13:05. [1]

**Cron.** Schedule `10 * * * *`, grace 5 minutes, last success 12:30. Late at the next match, 13:10. Down at 13:15. Cron means "at this wall-clock time," not "this many minutes after the last ping." That distinction is the product. [1] [29]

**OnCalendar.** Same as cron, but systemd timer syntax. Multiple expressions, one per line; a ping is expected when *any* line matches. [29]

**Start without success.** Grace is also the max allowed gap between `/start` and success. Miss it and the check is down even if the period has not elapsed. [1] [29]

### 2.3 Ping API (the hot path)

Healthchecks hosts pings on a separate host, `hc-ping.com`, away from the dashboard. Two URL shapes: [32]

- `https://hc-ping.com/<uuid>` — UUID assigned at create time, immutable. Lose the check, lose the UUID.
- `https://hc-ping.com/<project-ping-key>/<slug>` — human slug. Slug is **not** unique; a collision returns `409` and the ping is ignored.

Suffixes: `/start`, `/fail`, `/log`, `/<exit-status>` where `0` is success and `1–255` is failure. [32]

Query flags that matter:

- `rid=<uuid>` — client-chosen run id, so duration is matched to the right start when jobs overlap.
- `create=1` — auto-provision a check if the slug does not exist (slug URLs only, off by default).

Body: UTF-8, stored, capped. Hosted default shown in responses is `Ping-Body-Limit: 100000` (100 KB). Self-hosted default is 10 KB via `PING_BODY_LIMIT`. [32] [40]

Methods: HEAD, GET, POST. You can lock a check to POST only, because chat and email preview bots will GET a URL they see and fake a success. [29]

Email pings: `<uuid>@<ping-domain>`. Needs an SMTP listener on port 25 and DNS. Self-hosted ships `manage.py smtpd`. [14] [40]

Rate limit: more than about 5 pings per minute to one check may be dropped. UUID "not found" still returns **200**, so a typo in a cron line does not fail the job. Slug "not found" returns **404**. That split is deliberate. [32]

Cronitor's equivalent is `https://cronitor.link/p/<API_KEY>/<key>` with `?state=fail`, `env=`, `message=`, and `metric=name:value`. Auto-provision on first ping is the default happy path. [37]

### 2.4 Keyword filtering

For email subject, email body, or the first 100 KB of an HTTP body, match keywords in this order: failure, then success, then start. If nothing matches, either ignore the ping or treat it as failure (`filter_default_fail`). Matching is case-sensitive. [29] [25]

This exists so a backup that always exits 0 but writes "ERROR" in the log can still be a failure.

### 2.5 Management API

Project-scoped. No account-wide key. Read-write and read-only. Auth: `X-Api-Key` header, or `api_key` in a JSON POST body. JSON only, not form posts. Soft limit: 100 requests/minute, then 429. [25]

| Action | Method |
|---|---|
| List / get checks | `GET /api/v3/checks/` and `GET /api/v3/checks/<uuid>` |
| Create | `POST /api/v3/checks/` |
| Update | `POST /api/v3/checks/<uuid>` (not PUT) |
| Pause / resume | `POST .../pause` and `.../resume` |
| Delete | `DELETE /api/v3/checks/<uuid>` |
| Ping log | `GET .../pings/` and `.../pings/<n>/body` |
| Status flips | `GET .../flips/` |
| Channels | `GET /api/v3/channels/` |
| Badges | `GET /api/v3/badges/` |
| DB health | `GET /api/v3/status/` |

Create fields worth copying exactly: `name` (≤100), `slug` (`a-z0-9_-`, ≤100), `tags` (space-delimited string, not an array), `desc` (≤10,000), `timeout` (seconds, 60 to 31,536,000, default 86,400), `grace` (same bounds, default 3,600), `schedule`, `tz` (default UTC), `manual_resume`, `methods` (`""` or `"POST"`), `channels` (`"*"` or comma-separated UUIDs or exact names), `unique` (upsert on `name|slug|tags|timeout|grace`), plus the keyword-filter fields. [25]

If both `timeout` and `schedule` are sent, schedule wins. [25]

Read-only responses **omit** `uuid`, `ping_url`, `update_url`, `pause_url`, `resume_url`, `channels`, and add `unique_key`. Status values in the API are `new`, `up`, `grace`, `down`, `paused`. [25]

Hitting the plan's check cap returns 403. [25]

### 2.6 Who can do what

Roles, per project: Owner, Manager, Team Member, Read-only. [53]

- Team Member: CRUD checks and integrations, rename project, see and rotate API keys. Cannot invite, change owner, touch billing, or delete the project.
- Manager: Team Member plus invite and remove members. Still cannot touch owner or billing.
- Read-only: view checks, logs, integrations. No API keys (a key would be a back door to writes).
- Transfer a single check between projects and keep its ping URL.
- Transfer a whole project: owner starts it, the new owner must confirm by email. [53]

Check quota counts only checks in projects you **own**. Being invited to someone else's project does not eat your quota. [53]

Login on the hosted service is a magic email link, not a password. [3]

### 2.7 Alerts, badges, reports, metrics

Alerts fire on **state change**, not on a timer. Repeated nagging is a separate feature: hourly or daily email while anything is down, or you hand the incident to PagerDuty / Opsgenie / Pushover emergency and let them escalate. [31]

Channels are project-scoped. The same Slack webhook used in two projects is configured twice. [31]

Hosted channel list, from their own comparison (updated 2026-09-22): email, webhook (POST), Discord, Google Chat, Lark, Matrix, Mattermost, Microsoft Teams, ntfy, Opsgenie, PagerDuty, phone calls, Pushbullet, Pushover, Signal, Slack, SMS, Spike.sh, Telegram, Trello, Splunk On-Call (VictorOps), WhatsApp, Zulip. [41]

Self-hosted also wires GitHub Issues, Gotify, LINE Notify, Rocket.Chat, Apprise, and PagerTree, each behind an env flag. Outbound HTTP to private IPs is **off** by default (`INTEGRATIONS_ALLOW_PRIVATE_IPS`) so a user webhook cannot scan your LAN. [40]

SMS, WhatsApp, and phone calls have a monthly credit cap. Unused credits do not roll over. Over the cap: warning email, warning in the UI, no send. [3] [31]

Badges: public, unguessable URL. Formats `svg`, `json`, `shields` (up/down, grace counts as up) and `svg3` / `json3` / `shields3` (up / late / down). A `*` tag is the project-wide rollup. Tag-scoped badges too. [25] [28]

Email reports: daily, weekly (Monday), or monthly (1st). Per check: status, downtime count and duration for the last two months, grouped by project, including projects you were invited to. [31] [53]

Prometheus, per project, read-only key in the path or as a Bearer token for Grafana Cloud: [52]

- `hc_check_up`, `hc_check_started`, `hc_check_grace`, `hc_check_paused` (labels: `name`, `tags`, `unique_key`)
- `hc_tag_up`, `hc_checks_total`, `hc_checks_down_total`
- Cloaked details URL: `/cloaked/{unique_key}/`

### 2.8 How they run it

| | Healthchecks.io | Self-hosted Healthchecks |
|---|---|---|
| License | BSD-3. Hosted by SIA Monkey See Monkey Do, Pēteris Caune, Valmiera / Riga, Latvia. [13] [15] | Same code. |
| Stack | Python, Django. Docs say Django 6.0 and Python 3.12+. PostgreSQL in production; SQLite for dev; MySQL supported. [14] | `manage.py sendalerts` is a **separate process**. If it is not running, nothing alerts. [14] |
| Cron parser | `cronsim`. [13] | Same. |
| Ping body storage | External object storage (S3-compatible). `pruneobjects` cleans orphans. [14] [40] | Optional. |
| Ops | Hetzner bare metal in Germany, load-balanced app, PostgreSQL hot standby, daily backups to S3. Failover is **manual**. One-person ops. They say best-effort only, and that in practice availability has been over 99.9% since the July 2015 launch. [15] | You. |
| Cleanup | Keep newest N pings per check (100 free, 1000 paid). `pruneusers` drops never-logged-in accounts older than a month. `prunetokenbucket` drops rate-limit rows older than a day. [14] [3] | Same commands. |

Values they publish and you should steal as product principles, not as marketing copy: do one thing; no analytics JavaScript; no marketing email; account close is easy to find; simple and boring technology; 5% of MRR donated to upstream open source via GitHub Sponsors. [15]

---

## 3. Pricing and the business

### 3.1 Healthchecks.io (pricing page, read 2026-11-18)

| Plan | Price | Checks | Ping log / check | SMS + WhatsApp / month | Phone calls / month |
|---|---|---|---|---|---|
| Hobbyist | $0 | 20 | 100 | — | — |
| Supporter | $5 | 20 (same limits; a tip jar) | 100 | — | — |
| Business | $20 ($192/yr, 20% off) | 100 | 1,000 | 50 | 20 |
| Business Plus | $80 ($768/yr) | 1,000 | 1,000 | 500 | 100 |

Cards and PayPal. PDF invoices with optional company name, address, VAT ID. No prorated refunds. They email on failed charges and retry. Open-source projects and nonprofits can get Business free (118 such accounts). Multiple free accounts to dodge the 20-check cap is treated as abuse. [3] [15]

### 3.2 Cronitor (pricing page, read 2026-11-18)

| Plan | Price | What you get |
|---|---|---|
| Hacker | $0 | 5 monitors, email + Slack, 1 basic status page, 1 user. No SMS, no PagerDuty/Opsgenie. Fastest *active* check: 5 minutes. |
| Business | $2 / monitor / month + $5 / user / month | 14-day trial, then usage. SMS, on-call, 12-month retention, unlimited API, roles. SAML is +$5/user. Fastest check: 30 seconds. |
| Enterprise | from $6,000/yr | Dedicated engineer, custom integrations, invoice. |

No base fee. Their own example: 20 monitors × $2 + 3 users × $5 = **$55/month**. Status-page add-ons: branded $25/page, private $50/page. Browser checks $1 / 1,000. RUM $10 / 100,000 events after 100,000 free. [7]

Cronitor also sells website and API checks from 12+ regions, synthetic browser checks, real-user monitoring, status pages, environments (`env=staging`), custom metrics with assertions, failure tolerance, schedule tolerance, and alert surge protection. [36] [37] [7]

### 3.3 Same 20 jobs, three people

| Vendor | Monthly, about |
|---|---|
| Healthchecks Hobbyist | $0 (cap is 20 checks; a 4th person is free) |
| Healthchecks Business | $20 (100 checks; SMS included) |
| Cronitor Business | $55 (20×$2 + 3×$5) [7] |
| Dead Man's Snitch, cited by a 2026 roundup | $19 [24] |
| Better Stack, same roundup | $20, with heartbeats sharing a monitor quota [24] [60] |

The roundup's dollar figures are secondary. The vendor pricing pages are the primary source for Healthchecks and Cronitor. [3] [7] [24]

### 3.4 Their own numbers (Healthchecks About page, "updated manually, last updated November 1, 2026")

| Metric | Value |
|---|---|
| Years | 11 (public launch July 2015) |
| Free accounts | 199,000 |
| OSS / nonprofit accounts | 118 |
| Paying accounts | 1,420 |
| MRR | $29,300 |
| ARR (MRR × 12) | $351,600 |
| Checks | 645,000 |
| Pings / day | 134,000,000 |
| Notifications / day | 209,000 |

Derived from those figures, not published by them:

- ARPU ≈ **$20.63** per paying account. That sits on the Business plan ($20, or $16 if billed annually). Most payers are not on Plus.
- Paying share of accounts ≈ **0.7%**. This is a free-heavy, tiny-conversion business. You do not get rich on conversion rate. You get a living on being the default.
- ≈ 208 pings per check per day, about one ping every 7 minutes if spread evenly. Real traffic is bursty (top of the hour).
- ≈ 0.16% of pings become a notification. Alert volume is small. Ping volume is the scaling problem.
- ≈ $0.045 MRR per check if you smear revenue across all 645k checks. The free tier is the distribution channel.

Treat the About-page counts as self-reported and manually updated. [15]

### 3.5 Who else is in the aisle

Healthchecks (2015, EU, open source, cron-only), Cronitor (2014, US, closed, cron + uptime + status + RUM), Dead Man's Snitch (the original simple snitch), Better Stack (heartbeats inside a bigger observability suite), Sentry Crons (if errors already go to Sentry), Hyperping, UptimeRobot, Oh Dear, Cronhub, and a long tail of clones (WatchCron, Drumbeats, Cron Crew, QuietPulse). Uptime Kuma is the self-hosted uptime tool people already run; it is not a cron product. [19] [21] [41] [15]

Sentry comparison exists on Healthchecks' own docs (last updated 2026-09-21). Read it before you add "error tracking." They already decided that is a different product. [42]

---

## 4. Feature matrix you must hit to feel like "that site"

Checked against Healthchecks docs and their Cronitor comparison (2026-09-22). [1] [25] [29] [32] [41] [53]

| Capability | Healthchecks | Cronitor | Your v1 | Your v2 |
|---|---|---|---|---|
| Dead-man ping URL | Yes | Yes | Yes | Yes |
| Simple period + grace | Yes | Yes (`grace_seconds`, "every N minutes") | Yes | Yes |
| Cron expression + timezone | Yes | Yes | Yes | Yes |
| systemd OnCalendar | Yes | No | Yes | Yes |
| `/start` duration | Yes | Yes (job telemetry) | Yes | Yes |
| `/fail` and exit code | Yes | `state=fail` | Yes | Yes |
| Run id for overlapping jobs | Yes (`rid`) | Yes (job model) | Yes | Yes |
| Store ping body | 100 KB hosted | CLI / message, not a full body log | 100 KB | 100 KB + S3 |
| Keyword classify | Yes | No (they use metrics assertions instead) | Yes | Yes |
| Auto-provision on ping | Opt-in `create=1` | Default | Opt-in | Opt-in |
| Per-check channel toggle | Yes | Notification lists | Yes | Yes |
| Pause, sticky pause | Yes | Mute per monitor / group / env | Yes | Yes |
| Projects + 4 roles | Yes | Admin / user / read-only, paid per user | Owner + member + read-only | Full 4 |
| Read-only API key that hides ping URLs | Yes | Separate keys | Yes | Yes |
| Badges (2-state and 3-state) | Yes | Status page instead | SVG + JSON | Shields-compatible |
| Email reports | Daily / weekly / monthly | Daily / weekly / monthly | Weekly | All three |
| Prometheus | Yes | No | Yes | Yes |
| Management API + upsert | v3 | Monitor API, versioned (`2025-11-28`) | v1, version header | Yes |
| Email-in pings | Yes | No | No | If you want parity |
| SMS / call / WhatsApp | Paid credits, Twilio | Paid | No | Twilio later |
| Slack, Discord, webhook, email, ntfy | Yes | Yes (Slack from free) | Yes | Add Telegram, Pushover |
| PagerDuty, Opsgenie, Splunk | Yes | Paid plans | Webhook only | Native |
| Website uptime probes | No, on purpose | Yes, 12+ regions | No | No, unless you change the product |
| Status pages | No | Yes, paid extras | No | A single public page, not Cronitor's |
| Custom metrics + assertions | Exit code only | Yes | No | Agent can chart duration first |
| Failure / schedule tolerance | Grace only | Yes | Grace only | Add after you see false-alarm tickets |
| Environments | Tags | First-class `env=` | Tags | Optional |
| Crontab import | No | Yes | Agent does this | Agent does this |
| SDKs | Examples, no first-party SDK | Python, Node, Ruby | curl + 1 shell snippet | Python + Node |
| 2FA TOTP / WebAuthn | Yes | Not in their matrix | TOTP | WebAuthn |
| SAML / SCIM | No | Paid | No | No until a buyer asks |
| Self-host | Yes, BSD-3 | No | Yes, Docker | Yes |
| AI diagnosis | No | Docs for monitoring agents, not diagnosing jobs | **This is the wedge** | Yes |

---

## 5. The agent wedge

People pay ~$20 because wiring a cron to Slack is annoying, and because they do not want to learn grace vs period. An agent removes both.

### 5.1 Jobs the agent does

1. **Install.** User points it at a repo or pastes a crontab. Agent creates one check per job, picks a slug, sets timezone from the host, wraps the command so it pings `/start` and `/<exit>`, and opens a PR. Never commits to the default branch.
2. **Grace from evidence.** After ~20 runs, set grace to something like p95 duration × 1.5, with a floor of 60 seconds. Explain the number in the check description.
3. **Triage.** On `down`, pull the last ping body, the last 5 durations, and the schedule. Post a short incident: what was expected, what arrived, the tail of the log, and one likely cause (job still running, job never started, job started and failed, clock skew, duplicate slug).
4. **Fix proposal.** If the repo is connected, propose a patch (wrong cron, missing `|| curl .../fail`, timezone). Human merges.
5. **Noise control.** If the same check flaps more than N times a day, suggest a longer grace or a pause, and say why. Do not auto-pause production checks.
6. **Provision.** A new worker service starts and pings `create=1`. Agent names it, tags `prod`, attaches the on-call channel, and asks once.
7. **Weekly review.** The email report, but written as "3 checks have never pinged, 1 has been paused for 40 days, backup grace is tighter than its p95."

### 5.2 What the agent must not do

- It must not be on the ping hot path. A slow model call cannot delay `200 OK`.
- It must not send alerts by itself in v1. The deterministic notifier sends. The agent comments.
- It must not invent a ping URL or echo a read-only key into a chat log.
- It must not "fix" a down check by pausing it to clear the alert.
- It must not scrape private IPs through a webhook it configured.

### 5.3 Where this sits in Ghost Chat

If you build the operator UI inside Ghost Chat: Ghost Researcher reads docs and status pages, Ghost Code opens the PR that wraps the cron, a connector (MCP) exposes your management API as tools. The monitoring service itself should be a separate deploy, not a chat plugin. Chat is down more often than a boring ping host should be.

---

## 6. Legal and product boundaries

- **Healthchecks source** is BSD-3. You may ship a fork if you retain the copyright notice and the license disclaimer. You may not call the hosted product Healthchecks.io, use their logo, or copy their marketing text. Their About page and docs are not the code. [13] [15] [18]
- **Cronitor** is closed. Clone the behavior you can see from docs (ping in, alert out). Do not clone their dashboard, copy, or API responses line for line.
- **Ping URLs are secrets.** Treat them like API keys. Read-only API must not return them. Badges get a separate unguessable id. [1] [25]
- **Webhooks are an SSRF footgun.** Block link-local and private ranges unless the operator opts in. Healthchecks does this. [40]
- **Email-in on port 25** is an open relay waiting to happen if you are careless. Skip it in v1.
- **SMS and phone** need a vendor (they use Twilio) and a hard monthly cap, or one script will drain the account overnight. [40] [3]
- Do not promise 99.9% unless you have a second person and an automatic failover. They don't promise it, and they have run this for 11 years. [15]

---

## 7. What is uncertain

- Healthchecks' integration comparison was last updated **2026-09-22**. A channel may have been added since. The self-hosted env var list is the better inventory of what the code can do. [41] [40]
- The 2026-11-01 usage stats are manually updated. They can be weeks stale on any given day. [15]
- Cronitor's per-monitor math is clear; volume discounts are mentioned and not published as a table. [7]
- I did not read their Django models. Table names `api_ping`, `api_flip`, `api_notification` come from their own cleanup docs, so they are real, but a fork should not assume column layout without reading the migrations. [14]
- "Exact site" visually: I did not pull CSS, fonts, or layout. Copying the visual design is the part most likely to be a trademark/trade-dress problem, and it is also the part users do not pay for. Match the information architecture in `build.md`, use your own type and color.

---

## Sources

1. Healthchecks.io Documentation — https://healthchecks.io/docs/ — read 2026-11-18
2. Configuring Checks — https://healthchecks.io/docs/configuring_checks/
3. Plans and Pricing — https://healthchecks.io/pricing/
4. About (stats as of 2026-11-01) — https://healthchecks.io/about/
5. Self-hosted overview — https://healthchecks.io/docs/self_hosted/
6. Server configuration — https://healthchecks.io/docs/self_hosted_configuration/
7. Pinging API — https://healthchecks.io/docs/http_api/
8. Management API v3 — https://healthchecks.io/docs/api/
9. Configuring notifications — https://healthchecks.io/docs/configuring_notifications/
10. Projects and teams — https://healthchecks.io/docs/projects_teams/
11. Prometheus — https://healthchecks.io/docs/configuring_prometheus/
12. Healthchecks vs Cronitor (page says last updated 2026-09-22) — https://healthchecks.io/docs/healthchecks_cronitor_comparison/
13. Healthchecks vs Sentry (page says last updated 2026-09-21) — https://healthchecks.io/docs/healthchecks_sentry_comparison/
14. GitHub: healthchecks/healthchecks (BSD-3, Python & Django) — https://github.com/healthchecks/healthchecks
15. Cronitor pricing — https://cronitor.io/pricing — read 2026-11-18
16. Cronitor heartbeat docs — https://cronitor.io/docs/heartbeat-monitoring
17. Cronitor product home — https://cronitor.io/
18. Secondary pricing roundup (do not cite the dollars without rechecking vendor pages) — https://www.aitoolbox.hk/articles/best-cron-monitoring-tools-2026-cronitor-vs-healthchecks-vs-dead-mans-snitch-vs-better-stack/
```

```markdown build.md
# Heartbeat — Complete Build Spec

Build a cron / heartbeat monitor that behaves like Healthchecks.io, plus an AI-agent layer Cronitor and Healthchecks do not have.

Read `readme.md` first. This file is the thing an agent (or a human) builds from. It is opinionated on purpose. Where it says "copy this behavior," it means the state machine and the API contract, not their code, name, or visuals.

**Product name in this spec:** Heartbeat. Rename before launch. Do not ship as Healthchecks, Cronitor, or hc-ping.

**License of *your* code:** your choice. If you fork `healthchecks/healthchecks`, you must keep their BSD-3 notice. This spec assumes a clean-room build.

---

## 0. Outcome

A user can:

1. Sign up with an email magic link.
2. Create a check (period, or cron + timezone, plus grace).
3. Get a secret ping URL.
4. Curl it from a cron job.
5. See the check go `up`.
6. Wait past period + grace, or hit `/fail`, and get exactly one alert on the way down and one on the way back up.
7. Invite a teammate as read-only.
8. Put a badge in a README.
9. Ask the agent "wire the crons in this repo" and get a PR, not a lecture.

Non-goals for v1: uptime probes, status pages, RUM, SMS, phone, WhatsApp, SAML, email-in pings, native mobile apps.

---

## 1. System shape

Three processes. Do not collapse them.

```
                    ┌──────────────┐
  cron / agent ───► │ ping-api     │  write-optimized, no auth session
                    │ ping.example │  always answers before it does heavy work
                    └──────┬───────┘
                           │ append event
                    ┌──────▼───────┐
                    │ Postgres     │
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────▼─────┐  ┌──────▼─────┐  ┌──────▼─────┐
    │ scheduler  │  │ notifier   │  │ web + api  │
    │ flips late │  │ sends on   │  │ dashboard  │
    │ and down   │  │ transition │  │ management │
    └────────────┘  └─────┬──────┘  └────────────┘
                          │
                    Slack, email, webhook, ntfy
                          │
                    ┌─────▼──────┐
                    │ agent      │  reads flips + bodies, writes comments
                    │ (async)    │  never on the ping path
                    └────────────┘
```

Healthchecks splits this the same way: the site can be up while `sendalerts` is down, and they document that alerts simply do not go out. Your health check must alert *you* if the notifier or scheduler has not heartbeat itself in 60 seconds. [14]

### 1.1 Suggested stack

Boring on purpose. Their production choice is Django + Postgres on bare metal. You do not have to copy the framework. You do have to copy the split.

| Piece | Choice | Why |
|---|---|---|
| Ping API | Go or Rust, one binary | 134M pings/day is ~1,550/s average on *their* fleet, bursty at :00. You will not see that on day one. Still, the ping handler should be alloc-light and not import an ORM. |
| Web + management API | TypeScript (Hono/Fastify) or Django | CRUD, sessions, billing. Latency does not matter. |
| Scheduler + notifier | Same language as the web, separate processes | Restart independently. |
| DB | Postgres 16 | Checks, flips, channels. Not SQLite once you have two processes. |
| Ping bodies | Postgres `text` for ≤16 KB, S3-compatible for the rest | They already moved bodies to object storage. [14] |
| Queue | Postgres `SKIP LOCKED` | Do not add Redis until the notifier actually falls behind. |
| Cron math | A real cron library with timezone support, plus tests against `cronsim` cases | Wrong "next run" is a product bug, not a polish bug. |
| Email | Any transactional SMTP (magic links + alerts) | |
| Agent | Your model gateway, tool calls only, 30 s timeout | |

One VPS is enough until you pass a few million pings a day. Put ping-api behind its own hostname (`ping.example.com`) so a dashboard deploy does not drop pings. Healthchecks uses `hc-ping.com` for this. [32]

### 1.2 Repos

```
heartbeat/
  ping-api/          # hot path
  web/               # dashboard, management API, badges, metrics
  worker/            # scheduler, notifier, retention
  agent/             # triage, installer, weekly review
  sdks/shell/        # the only client you ship in v1
  deploy/            # docker-compose, Caddyfile, migrations
  docs/
```

---

##
