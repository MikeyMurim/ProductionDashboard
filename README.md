# Flowstate Production Dashboard

One place where everyone on the Flowstate team can see whether Flowstate is working, how it is growing, and what the system has already fixed for itself. The dashboard watches Flowstate's outside services, records what happens in a standard event log, and alerts the team when something needs a person.

> **Status:** for build. Based on *Requirements v3.0* (16 November 2026), which replaces v1 and v2.
> Specific values (thresholds, timings, alert channel, sign-in method) are starting values, to be confirmed with the team during the build.
>
> **Changes since v3:** no UK or EU data residency rule (NFR-4); days run in Sydney time and times show in each viewer's own time zone (FR-11.3); a proposed tech stack section.

## Contents

- [Principles](#principles)
- [Scope](#scope)
- [Glossary](#glossary)
- [Data providers](#data-providers)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Event log: OCEL 2.0](#event-log-ocel-20)
- [Requirement format](#requirement-format)
- [Business requirements](#business-requirements)
- [Functional requirements](#functional-requirements)
- [Non-functional requirements](#non-functional-requirements)
- [Traceability from v1](#traceability-from-v1)

## Principles

| ID | Principle |
|---|---|
| P-1 | **Report what was fixed before what is broken.** Problems the system resolved itself are shown first, as reassurance. Only problems that need a person interrupt anyone. |
| P-2 | **Read-only.** The dashboard observes and never changes the systems it watches. |
| P-3 | **Totals, not people.** Only aggregate numbers about customers are collected or shown. |
| P-4 | **Independent.** The dashboard keeps working when Flowstate is down, because that is when it matters most. |
| P-5 | **Plain words.** Every status and event is readable without technical knowledge. |

## Scope

### In scope

- A main page with status lights, key numbers, a self-healing headline and an events feed.
- Detail pages for Cloudflare, Render, GitHub, Clerk and GA4.
- A synthetic checker running every minute from outside Flowstate's own infrastructure.
- A dashboard-owned store holding the OCEL 2.0 event log and metric history.
- Alerts when a service goes Red and when it recovers, and a weekly summary email.
- Sign-in for team members on an allow-list.

### Out of scope

| Excluded | Why | Where it goes instead |
|---|---|---|
| Fixing or restarting anything from the dashboard | The dashboard is read-only (NFR-3). Write access would make it a security risk in its own right. | Render and GitHub consoles |
| A full CRM (contacts, pipelines, notes) | Needs individual customer records, which NFR-4 forbids. A product decision of its own. | A CRM later, added as a provider (NFR-10) |
| Data about individual customers | Data minimisation under privacy law (NFR-4). Totals answer every question asked. | Clerk dashboard, for those with access |
| In-code performance monitoring (traces, stack traces) | Needs code changes in the app and a different class of tool. | Error tracking or OpenTelemetry later, as a provider |
| Raw log search | High volume and cost. Logs can contain personal data. | Render's log viewer |
| Billing, revenue and payments | No payment provider exists yet. | Added as a provider when one exists (NFR-10) |
| Hosting cost monitoring | Not requested. | Provider billing pages |
| Public status page for customers | Different audience and wording. | Possible later phase, reusing the same checks |
| WhatsApp alerts and a mobile app | WhatsApp needs a Business API account and approved templates. The web page works on phones. | Slack or email (FR-12); responsive page (NFR-8) |
| Data from before go-live | Most providers keep limited history. | History starts when collection starts |

## Glossary

Every requirement uses these terms with exactly these meanings.

| Term | Definition |
|---|---|
| Team member | A person on the Flowstate team allow-list. All team members have the same access. |
| Service | Something with its own status light: Website, App and Accounts (Clerk sign-in). |
| Component | A deployable unit on Render (web service, worker, cron job or database). A service can have several. |
| Provider | An outside system the dashboard reads from: Cloudflare, Render, GitHub, Clerk, Google Analytics 4 (GA4), plus our own synthetic checker. |
| Synthetic check | A scripted visit by the dashboard to a page, recording pass or fail and response time, made the way a visitor would. |
| Status | Green "All good", Amber "Slower than usual", Red "Down", or Grey "Couldn't check right now". Thresholds in FR-1. |
| Incident | A period during which a service is Amber or Red. It has a start, an end and a resolution. |
| Self-healed | An incident that ended without any human action recorded between its start and end (FR-4). |
| Team-fixed | An incident that ended after a human action: a manual deploy, a manual restart or a hotfix merge. |
| Deploy | A Render deploy of a component. Outcome: live, failed, cancelled or rolled back. |
| Uptime | Percentage of synthetic checks passed in the period. Amber counts as up. |
| Visitor | A unique browser on the website as counted by GA4 ("active users"). |
| Sign-up | A completed account creation in Clerk. A started sign-up is a visit to the sign-up page (GA4). |
| Active user | A signed-in person who opened the app in the period. |
| Event | A single timestamped fact stored in the event log, such as "deploy went live". |
| Metric | A number sampled over time, such as CPU % or requests per minute. |
| Period | The window chosen in the period picker: today, last 7 days or last 30 days. Days run in Sydney time (Australia/Sydney), the main operating time zone. |

## Data providers

What each provider can deliver decides how it is collected. Webhooks push events to us within seconds; polling means we ask on a timer.

| Provider | Data used | Method | Frequency | History kept by provider | Constraints |
|---|---|---|---|---|---|
| Synthetic checker (ours) | Up or down, response time, page content correct | We run it | Every 60 s, 2 regions | n/a (we store it) | Must run outside Flowstate's infrastructure |
| Render | Component status, deploys, restarts, CPU, memory, response time | Webhooks for events; API polling for metrics | Seconds (events); 5 min (metrics) | Limited | Webhooks need the Pro plan or higher |
| GitHub | PRs opened and merged, merges to the deploy branch, CI results | Webhooks; API for catch-up | Seconds | Full repository history | Read-only token or GitHub App |
| Clerk | Sign-ups, active users | Webhooks (via Svix); API for counts | Seconds | Users kept; events not | Duplicates possible, so processing must be idempotent. No feed for failed log-ins |
| Cloudflare | Requests, 5xx errors, delivery time, blocked threats | GraphQL Analytics API polling | Every 5 min | At least 31 days | 300 queries per 5 min |
| Google Analytics 4 | Visitors, sources, top pages, sign-up funnel | Data API polling (plus Realtime API) | Hourly and daily | 2 or 14 months | Data is 2 to 48 h late and can change for up to 48 h |

### Webhook support

| Service | Webhooks? | What it would tell us | What it takes |
|---|---|---|---|
| GitHub | Yes, on every plan, including free | Code changes opened and merged, merges to the live branch, test results | Someone with admin rights on each repository switches it on |
| Clerk | Yes | New sign-ups, sign-ins, account changes. Not failed log-ins (FR-9.3). | Switched on in the Clerk dashboard. No code change in the app. |
| Render | Yes, but only on the Pro plan or higher | New versions going live or failing, crashes, restarts, scaling | Pro plan. If we stay on a lower plan, Render is checked every 1 minute instead. |
| Cloudflare | Partly. Only for some alerts, such as attacks, depending on plan. | Attack alerts only. Traffic numbers are not sent. | Traffic is checked every 5 minutes |
| Google Analytics 4 | No | n/a | Checked hourly and daily |

GitHub and Clerk webhooks cost nothing extra. Render webhooks need the Pro plan if we are not already on it.

## Architecture

### Data collection: hybrid

Use **webhooks** wherever a provider offers them (Render, GitHub, Clerk), and **scheduled polling** where it does not (Cloudflare, GA4, Render metrics, synthetic checks). Webhooks work like a doorbell: the provider tells us the moment something happens. Polling is like checking a post box: we ask on a timer.

Events (a deploy, a restart) and levels (CPU, traffic) are different things. Webhooks catch events exactly and instantly; polling suits levels and works with any provider. Neither alone covers all five providers.

**Safety net:** each night the dashboard asks every webhook provider for the day's full event list and adds anything it missed. This gives the speed of push with the completeness of pull.

| | Normal API call (polling) | Webhook |
|---|---|---|
| Who starts it | We ask the service | The service tells us |
| When | On a timer, e.g. every 5 minutes | The moment something happens |
| How fresh | Up to one timer interval late | Within seconds |

Webhook receivers must:

- Live at a permanent, always-on address hosted outside Flowstate (NFR-2), so messages still arrive when Flowstate is down.
- Verify every message's signature and reject any that isn't signed correctly (NFR-3).
- Ignore duplicates (providers sometimes send the same message twice).
- Be backed by the nightly catch-up, in case a message is lost while the dashboard is offline.

### One store, owned by the dashboard

Everything collected goes into the dashboard's own store. The page reads only from that store, so opening it never calls a provider (NFR-7). The store and collectors are hosted outside Flowstate's Render workspace (NFR-2).

```
 Render ─┐ webhooks                         ┌──────────────┐
 GitHub ─┼───────────► Webhook receivers ──►│              │
 Clerk  ─┘            (verify, de-dupe)     │  Dashboard   │       ┌────────────┐
                                            │    store     │──────►│ Dashboard  │
 Cloudflare ─┐ polling                      │ (OCEL events │       │    page    │
 GA4        ─┼───────► Pollers ────────────►│  + metrics)  │       └────────────┘
 Render API ─┘                              │              │──────► Alerts / weekly email
 Synthetic checker (2 regions) ────────────►│              │
                                            └──────────────┘
          Nightly catch-up re-reads each webhook provider's event list
```

## Tech stack

> **Proposed.** To be confirmed with the team before the build starts.

### Approach: match the Flowstate website

The dashboard uses the same languages and framework as the Flowstate website (`flowstate-website`), rather than a separate stack:

- **One language end to end.** The page, webhook receivers, pollers and synthetic checker are all TypeScript on Node. Anyone who works on the website can work on the dashboard, and there is only one set of tooling to maintain.
- **Design system for free (NFR-9).** We can reuse the website's Tailwind config and Design System V2.3 tokens directly, instead of rebuilding them in another framework.
- **Provider SDKs are strongest in TypeScript.** Svix (Clerk webhooks), Octokit (GitHub), the GA4 Data API client and Cloudflare's GraphQL API all have first-class Node support.
- **Nothing in the requirements needs another language.** OCEL 2.0 analysis tools (for example PM4Py, which is Python) read our OCEL JSON export (NFR-11). We don't need to run them inside the dashboard.

We'd only pick a different language if the people building the dashboard are much stronger in it than in TypeScript. Even then, the cost is a second stack to maintain beside the website, and the design system would have to be rebuilt.

### Components

| Layer | Choice | Same as website? | Notes |
|---|---|---|---|
| Language | TypeScript 5 | Yes | Strict mode |
| Web framework | SvelteKit 2 + Svelte 5 (runes) | Yes | Server routes also host the webhook endpoints |
| Styling | Tailwind CSS 3 + Flowstate Design System V2.3 | Yes | Share the website's Tailwind config; light and dark modes follow the device setting |
| Icons | Phosphor (`phosphor-svelte`) | Yes | Every status light gets an icon as well as a word (NFR-12) |
| Charts | A Svelte-friendly chart library (e.g. LayerChart or Chart.js) | New | Sign-ups per day, traffic, CPU and memory |
| Runtime | Node 22 LTS (`adapter-node`) | Website is on Node 20 | Node 20 reached end of life in April 2026; the website should move to 22 as well |
| Store | PostgreSQL | New | OCEL 2.0 tables for events and objects, plus a time-series table for metrics; kept for at least 90 days (NFR-6) |
| Database access | Typed query layer (e.g. Drizzle ORM) | New | Schema kept in TypeScript beside the code |
| Collectors | A Node worker process running the pollers on schedules, plus the nightly catch-up | New | Kept separate from the web process, so a slow page never delays collection |
| Webhook verification | Svix (Clerk), HMAC signature checks (GitHub, Render) | New | Reject invalid signatures and ignore duplicates (NFR-3) |
| Synthetic checker | Small checker deployed in 2 regions, or a hosted check service | New | Must run outside Flowstate's infrastructure (FR-14) |
| Alerts | Slack incoming webhook, with email fallback | New | Channel still to be confirmed (FR-12) |
| Email | Transactional email provider | New | Weekly summary (FR-5) and alert fallback |
| Heartbeat | External heartbeat monitor | New | Alerts if the collector stops for more than 5 minutes (NFR-2) |
| Hosting | A provider other than Flowstate's Render workspace | Website uses Render | Required by NFR-2 |

### Things that must differ from the website

- **Hosting.** The website runs on Render. The dashboard must not run in Flowstate's Render workspace (NFR-2), because it has to stay up when Render is the thing that's down. A different hosting provider is safest.
- **Sign-in.** Flowstate's customers sign in with Clerk, and the dashboard's Accounts light watches that sign-in. If the dashboard used the same Clerk instance for team sign-in, a Clerk outage would lock the team out at the moment they need to look. Team sign-in (FR-13) should use a separate Clerk instance or another method. The sign-in method is still to be confirmed.

## Event log: OCEL 2.0

Every event is stored in [OCEL 2.0](https://www.ocel-standard.org/) (Object-Centric Event Log), the open standard from the process-mining community. Each event is linked to every object it involved, such as a deploy, commit, pull request, component or incident. This lets the log be analysed indefinitely with standard tools, for example "which merges led to incidents?".

Metrics are kept in a separate time-series table. Only threshold crossings, such as "CPU over 90% for 5 minutes", become events.

| Object type | Example events |
|---|---|
| `service` | status changed, incident opened, incident closed |
| `component` | server failed, server restarted, scaled |
| `deploy` | deploy started, build failed, deploy live, rolled back |
| `pull_request` / `commit` | PR opened, PR merged, CI passed, CI failed |
| `check` | check passed, check failed |
| `incident` | opened, self-healed, team-fixed, alert sent |
| `signup_period` | sign-ups counted (aggregate only, no individuals) |

## Requirement format

Each functional requirement has a **user story** (who and why), numbered **EARS** statements (what the system shall do), and a **Gherkin** acceptance scenario (how we prove it). Every statement must be **SMART**: specific, measurable, achievable, relevant and time-bound.

| EARS pattern | Template |
|---|---|
| Ubiquitous | The dashboard shall &lt;response&gt;. |
| Event-driven | When &lt;trigger&gt;, the dashboard shall &lt;response&gt;. |
| State-driven | While &lt;state&gt;, the dashboard shall &lt;response&gt;. |
| Unwanted behaviour | If &lt;unwanted condition&gt;, then the dashboard shall &lt;response&gt;. |
| Optional feature | Where &lt;feature is included&gt;, the dashboard shall &lt;response&gt;. |

## Business requirements

| ID | Business requirement | Measure of success |
|---|---|---|
| BR-1 | One page where any team member can see whether Flowstate is working, without technical knowledge. | A team member can answer "is everything OK?" within 10 seconds of opening the page. |
| BR-2 | Find out when something breaks or an update goes wrong before customers notice. | Every Red status is alerted within 2 minutes. |
| BR-3 | See how Flowstate is growing: visits, sign-ups and app use. | Trends visible for today, 7 and 30 days, compared with the previous period. |
| BR-4 | Early warning of suspicious activity, such as attacks or repeated log-in attempts. | Blocked attacks shown within 15 minutes. Failed log-ins. |
| BR-5 | Track potential and actual customers from sign-up data until a CRM is added. | Sign-up counts and funnel shown; no individual records held. |
| BR-6 | Show what the system fixed by itself, so the team spends effort only where needed. | Every closed incident classified as self-healed or team-fixed. |
| BR-7 | Keep a permanent, analysable record of operational events. | All events stored in OCEL 2.0 and exportable. |

## Functional requirements

### Main page

#### FR-1 Status lights

*As a team member, I want a traffic light for each service, so that I can see at a glance whether Flowstate is working.*

- **FR-1.1** The dashboard shall show one status light for each service: Website, App and Accounts.
- **FR-1.2** When all synthetic checks for a service pass and its response time is within target, the dashboard shall show Green "All good".
- **FR-1.3** When a service's 95th-percentile response time exceeds 2 seconds over 5 minutes, or checks fail from only one region, the dashboard shall show Amber "Slower than usual".
- **FR-1.4** When 3 consecutive checks fail from all regions, the dashboard shall show Red "Down".
- **FR-1.5** The dashboard shall show one plain-English sentence beside each light explaining its status.
- **FR-1.6** If no check result has been received for a service for more than 2 minutes, then the dashboard shall show Grey "Couldn't check right now".

> v1 had a light for "customer tracking". It is not a running service, so it becomes key numbers (FR-2) and the Clerk section (FR-9). The Accounts light covers whether sign-in works.

```gherkin
Feature: Status lights
  Scenario: Website goes down
    Given the Website status is Green
    When 3 consecutive synthetic checks of the Website fail from all regions
    Then the Website light shows Red "Down"
    And the sentence beside it says when the problem started
```

#### FR-2 Key numbers

*As a team member, I want the main numbers in everyday words with a trend, so that I can see whether Flowstate is growing.*

- **FR-2.1** The dashboard shall show: visitors today, new sign-ups this week, active users today, and uptime this month.
- **FR-2.2** The dashboard shall show beside each number its percentage change from the same period last week, with an up or down arrow.
- **FR-2.3** While a figure comes from GA4 data less than 48 hours old, the dashboard shall label it "provisional".

```gherkin
Feature: Key numbers
  Scenario: Sign-ups up on last week
    Given 40 sign-ups were recorded last Monday to Wednesday
    And 50 sign-ups have been recorded this Monday to Wednesday
    When a team member opens the main page on Wednesday
    Then "New sign-ups this week" shows 50 with "up 25% on last week"
```

#### FR-3 Events feed

*As a team member, I want a plain-English list of recent events, so that I know what changed and when.*

- **FR-3.1** The dashboard shall list events newest first, each with time, a plain-English sentence and an outcome tag: Fixed automatically, Fixed by team, Open, or Info.
- **FR-3.2** When an event is recorded, the dashboard shall show it in the feed within 1 minute.
- **FR-3.3** The feed shall include deploys, incidents opened and closed, merges to the deploy branch, and blocked-attack spikes.

```gherkin
Feature: Events feed
  Scenario: New version goes live
    Given PR 123 has been merged to the deploy branch
    When Render reports the deploy as live at 14:02
    Then the feed shows "A new version of the app went live at 14:02 (PR 123)" tagged Info
```

#### FR-4 Self-fixed problems headline

*As a team member, I want to see which problems fixed themselves, so that I know where no action is needed and trust the system.*

- **FR-4.1** When an incident closes, the dashboard shall classify it as Self-healed if no human action (manual deploy, manual restart or hotfix merge) was recorded between its start and end, and otherwise as Team-fixed.
- **FR-4.2** The dashboard shall show at the top of the main page a headline for the selected period, such as "3 problems fixed themselves, 0 need attention".
- **FR-4.3** If the same component self-heals more than 3 times in 24 hours, then the dashboard shall raise an Amber warning for that service.
- **FR-4.4** Self-healing is detected for crash restarts, failed deploys that never went live, autoscaling during traffic spikes, blocked attacks and brief check failures.

```gherkin
Feature: Self-healed incidents
  Scenario: App crashes and restarts itself
    Given the App status is Green
    When Render reports "server failed" at 03:12
    And Render reports "server available" at 03:13
    And no team member deployed or restarted anything in between
    Then the incident is classified as Self-healed
    And the feed shows "The app restarted itself after a crash at 03:12. Back in 1 minute."
    And no alert is sent
```

#### FR-5 Weekly summary email

*As a team member, I want a short weekly email, so that I stay informed without opening the dashboard.*

- **FR-5.1** When it is Monday 09:00 Sydney time, the dashboard shall email the weekly summary to the team distribution list.
- **FR-5.2** The summary shall list, in this order: problems fixed automatically, problems still open, key numbers with trends, and number of deploys.

```gherkin
Feature: Weekly summary
  Scenario: Summary is sent
    Given it is Monday 09:00 Sydney time
    Then each person on the distribution list receives the summary
    And its first section is "Fixed automatically this week"
```

### Detail pages

#### FR-6 Website traffic (Cloudflare)

*As a team member, I want to see traffic, errors, speed and blocked attacks, so that I can judge the website's health.*

- **FR-6.1** The dashboard shall show, for the selected period: requests, percentage of responses that were errors (5xx), median delivery time, and number of threats blocked.
- **FR-6.2** The dashboard shall poll Cloudflare every 5 minutes and stay within 300 queries per 5 minutes.
- **FR-6.3** When blocked threats in a 15-minute window exceed 10 times the 7-day average, the dashboard shall add a feed event tagged Fixed automatically.

```gherkin
Feature: Cloudflare traffic
  Scenario: Attack blocked
    Given the 7-day average is 20 blocked threats per 15 minutes
    When 1,240 threats are blocked in 15 minutes
    Then the feed shows "1,240 suspicious requests blocked. Site unaffected." tagged Fixed automatically
```

#### FR-7 Servers and deploys (Render)

*As a team member, I want to see each component's state, its deploy history and its load, so that I can see whether updates worked.*

- **FR-7.1** The dashboard shall show each Render component with its status, CPU %, memory % and response time.
- **FR-7.2** The dashboard shall show deploy history for the selected period with start time, end time, outcome and linked PR.
- **FR-7.3** When Render reports a deploy event, the dashboard shall record it within 1 minute (webhook), or within 2 minutes where Render is polled.

```gherkin
Feature: Deploy history
  Scenario: Failed deploy
    Given component "app-web" is running version A
    When a deploy of version B ends with outcome "failed"
    Then deploy history shows version B as Failed
    And the App light stays Green while version A keeps running
```

#### FR-8 Code changes (GitHub)

*As a team member, I want each deploy linked to the code change behind it, so that I know what changed when something breaks or improves.*

- **FR-8.1** The dashboard shall show, for the selected period, pull requests opened and merged, and merges to the deploy branch.
- **FR-8.2** When a Render deploy starts, the dashboard shall link it to the merged PR with the same commit SHA.
- **FR-8.3** The feed shall show each linked deploy as one event, such as "PR 123 merged, deployed 14:02, healthy".

```gherkin
Feature: Merge-to-deploy link
  Scenario: Merge triggers deploy
    Given PR 123 is merged to the deploy branch as commit abc123
    When Render starts a deploy of commit abc123
    Then the deploy is linked to PR 123
    And the feed shows one combined event for the merge and the deploy
```

#### FR-9 Accounts (Clerk)

*As a team member, I want sign-up and usage numbers, so that I can track growth and potential customers.*

- **FR-9.1** The dashboard shall show, for the selected period: new sign-ups, active users, and sign-ups per day as a chart.
- **FR-9.2** The dashboard shall store sign-ups as counts per period only, never individual users.
- **FR-9.3** Where a source of failed log-in data is agreed, the dashboard shall show failed log-in attempts per hour and flag more than 20 per hour as Amber.

```gherkin
Feature: Sign-ups
  Scenario: New account created
    Given 12 sign-ups have been recorded today
    When Clerk reports a new user created
    Then "new sign-ups today" shows 13
    And no name or email address is stored
```

#### FR-10 Visitors (GA4)

*As a team member, I want to know where visitors come from and how many finish signing up, so that I can see what drives growth.*

- **FR-10.1** The dashboard shall show, for the selected period: top 5 traffic sources, top 10 pages, and the sign-up funnel (started versus completed, with conversion %).
- **FR-10.2** The dashboard shall poll GA4 hourly for intraday figures and daily for confirmed figures.

```gherkin
Feature: Sign-up funnel
  Scenario: Funnel conversion
    Given GA4 reports 200 visits to the sign-up page in the last 7 days
    And Clerk reports 50 completed sign-ups in the same period
    Then the funnel shows 200 started, 50 completed, 25% conversion
```

### Everywhere

#### FR-11 Period picker

*As a team member, I want to choose the time period, so that I can compare short- and long-term trends.*

- **FR-11.1** The dashboard shall offer three periods: today, last 7 days and last 30 days, defaulting to today.
- **FR-11.2** While a period is selected, every number, chart and list on the page shall use that period.
- **FR-11.3** The dashboard shall show every time in the viewer's own time zone (the team is in Sydney and Bangkok) and label it, for example "14:02 AEDT".

```gherkin
Feature: Period picker
  Scenario: Switch to 30 days
    Given the main page shows today's figures
    When a team member selects "Last 30 days"
    Then all numbers and the feed update to the last 30 days within 1 second
```

#### FR-12 Alerts

*As a team member, I want to be told quickly when a service goes down and when it recovers, so that I can act before customers notice, or stand down if it fixed itself.*

- **FR-12.1** When a service's status changes to Red, the dashboard shall send an alert to the alert channel within 2 minutes of the first failed check.
- **FR-12.2** When a Red service returns to Green, the dashboard shall send a recovery message stating the outage duration and whether it was self-healed.
- **FR-12.3** If an incident self-heals within 5 minutes, then the dashboard shall not alert, and shall record it in the feed and weekly summary.
- **FR-12.4** If the alert channel rejects a message, then the dashboard shall retry for up to 15 minutes, then fall back to email.
- **FR-12.5** While a service remains Red, the dashboard shall send at most one reminder every 30 minutes.

```gherkin
Feature: Alerting
  Scenario: Website down, then recovers by itself
    Given the Website status is Green
    When 3 consecutive synthetic checks of the Website fail
    Then an alert is sent to the alert channel within 2 minutes of the first failure
    When the next check passes and no team member acted
    Then a recovery message says "Recovered by itself after 4 minutes"
```

#### FR-13 Access

*As the team, we want only team members to see the dashboard, so that operational data stays internal.*

- **FR-13.1** The dashboard shall allow sign-in only to people on the team allow-list.
- **FR-13.2** If someone not on the allow-list tries to sign in, then the dashboard shall deny access and record the attempt.
- **FR-13.3** The dashboard shall give every signed-in team member the same read-only access.

```gherkin
Feature: Access
  Scenario: Outsider blocked
    Given "someone@example.com" is not on the allow-list
    When they try to sign in
    Then access is denied
    And the attempt is recorded
```

#### FR-14 Synthetic checks

*As a team member, I want the site and app checked every minute the way a visitor would, so that problems are caught even when no one is using Flowstate.*

- **FR-14.1** The dashboard shall check each monitored URL every 60 seconds from at least 2 regions.
- **FR-14.2** A check shall pass only if the page returns success within 10 seconds and contains its expected content.
- **FR-14.3** The dashboard shall record each check's result and response time as an event.

```gherkin
Feature: Synthetic checks
  Scenario: Page loads but is broken
    Given the expected content for the home page is "Flowstate"
    When the home page returns HTTP 200 without the expected content
    Then the check is recorded as failed
```

## Non-functional requirements

| ID | Quality | Requirement (EARS) | How it is measured |
|---|---|---|---|
| NFR-1 | Up to date | The dashboard shall show status lights no more than 60 s old and other numbers no more than 15 min old, except GA4 figures (FR-2.3). Each section shall show "Last updated hh:mm". | Age of newest data per section, logged every minute |
| NFR-2 | Independent | The dashboard shall run on hosting separate from Flowstate's Render workspace. If the dashboard's collector stops for more than 5 minutes, then an external heartbeat monitor shall alert the team. | Test: stop Flowstate staging; dashboard stays up and shows Red |
| NFR-3 | Secure | The dashboard shall use read-only provider credentials, keep them server-side only, use HTTPS, and reject any webhook with an invalid signature. | Security review; no secret appears in browser traffic |
| NFR-4 | Private (UK GDPR; Australian Privacy Act where it applies) | The dashboard shall store and show aggregate customer data only. It shall store no customer names, emails or IP addresses. The only personal data held is the team allow-list and refused sign-in attempts (FR-13.2). Because the business is registered in the UK, storing these outside the UK is an international transfer and needs the hosting provider's data processing agreement in place. | Data inventory review before go-live |
| NFR-5 | Copes with partial failure | If a provider cannot be reached, then only its section shall show "Couldn't check right now" with the time of the last good data, and all other sections shall keep working. | Test: revoke one provider token |
| NFR-6 | Keeps history | The dashboard shall keep at least 90 days of events and metrics. | Oldest record in store |
| NFR-7 | Light on providers | Opening the page shall make no calls to providers. Polling shall stay within each provider's published rate limits. | Provider API usage dashboards |
| NFR-8 | Fast on phones | The page shall load (largest content painted) in under 2 seconds at the 75th percentile, on a mid-range phone over 4G and on desktop broadband. | Lighthouse and real-user timing |
| NFR-9 | Looks like Flowstate | The dashboard shall follow Flowstate Design System V2.3, and offer light and dark modes following the device setting. | Design review against the design system |
| NFR-10 | Easy to grow | Where a new provider is added, it shall need only a new adapter that writes events and metrics to the store, with no change to existing pages. | Adding a test provider in 1 day or less |
| NFR-11 | Standard event log | The dashboard shall store every event in OCEL 2.0 and shall export the log in OCEL JSON on request. | Export opens in a standard OCEL tool |
| NFR-12 | Accessible | The dashboard shall meet WCAG 2.2 level AA. Status shall never be shown by colour alone: each light shall also carry its word ("All good", "Down") and an icon. | Automated accessibility check; screen-reader test |
| NFR-13 | Reliable | The dashboard and its alerting shall be available at least 99.5% of each calendar month. The dashboard shall send a test alert on the first working day of each month to confirm alerts are delivered. | Heartbeat monitor uptime report; monthly test alert received |

## Traceability from v1

| v1 | Final (v3) |
|---|---|
| BR-1 to BR-5 | BR-1 to BR-5, reworded with measures of success |
| (new) | BR-6 self-fixed problems; BR-7 permanent event record |
| FR-1, FR-2, FR-3 | FR-1, FR-2, FR-3 |
| (new) | FR-4 self-fixed problems headline |
| FR-4 weekly email | FR-5 |
| FR-5 Cloudflare, FR-6 Render | FR-6, FR-7 |
| (new) | FR-8 GitHub |
| FR-7 Clerk, FR-8 Google Analytics | FR-9, FR-10 |
| FR-9 period, FR-10 alerts, FR-11 access, FR-12 checks | FR-11, FR-12, FR-13, FR-14 |
| NFR-1 to NFR-10 | NFR-1 to NFR-10, made measurable |
| (new) | NFR-11 OCEL event log; NFR-12 accessible; NFR-13 reliable |
