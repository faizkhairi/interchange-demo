# Alerting

Last updated: 2026-11-17

This document covers how Interchange raises alerts, when it looks for them, what a notification is, and how an alert can be posted to a webhook (P7b). Where a statement depends on a source file, the file and line are cited so you can check the behaviour rather than trust this page. The reasoning behind the controls is in [security.md](security.md), the procedures are in the operations runbook (in the source repository), and the endpoint table is in [architecture.md](architecture.md).

## The eight rules

One pure module, `core/pure/alert-rules.ts`, holds the rules, thresholds and messages. The server evaluator (`core/jobs/alerts.ts`) and the in-browser demo engine both call it, so the two cannot disagree about when an alert opens, escalates or resolves (`core/pure/alert-rules.ts:3-9`). The ids are listed at `core/pure/alert-rules.ts:17-26`, the levels at `core/pure/alert-rules.ts:69-121`.

| Rule | Entity | Fires when | Warning | Critical |
|---|---|---|---|---|
| `dlq_open` | system | live `PENDING_REVIEW` dead letters on a system | 1 or more | 10 or more |
| `job_failure_rate` | system | failed (`FAILED` plus `DLQ`) share of terminal jobs in the last 60 minutes, at least 10 terminal jobs | 25% | 50% |
| `inbound_error_rate` | system | 5xx share of inbound calls in the last 15 minutes, at least 20 calls | 20% | 50% |
| `cert_expiry` | certificate | a non-revoked certificate of a non-deleted system ends within 30 days, or has ended | within 30 days | within 7 days, or expired |
| `system_down` | system | an `ACTIVE`, non-deleted system whose stored health is `DOWN` | always | never |
| `tick_stale` | platform | the tick is armed and has not run for 15 minutes (or has not run since it was armed) | always | never |
| `job_stuck` | job | a live `PROCESSING` job locked longer than `JOB_STUCK_MINUTES` (default 10) | always | never |
| `queue_backlog` (P7b) | platform | live jobs are due and the oldest has waited 15 minutes or more | oldest due job waited 15 minutes | oldest due job waited 60 minutes, or 500 or more due jobs |

Each rule keeps at most 100 firing entities per run (`job_stuck`: 50), and one evaluator run writes at most 200 opens plus resolves (`core/pure/alert-rules.ts:40-46`). A rule cut by its cap is not auto-resolved in that run (`core/pure/alert-rules.ts:455-483`).

Each firing has the fingerprint `<rule>:<entityType>:<entityId>` (`core/pure/alert-rules.ts:164-166`). There is at most one OPEN alert per fingerprint. A firing with no open row opens one. A higher severity escalates it. A lower severity or a changed message is a quiet update with no notification. An open row whose fingerprint no longer fires resolves (`core/pure/alert-rules.ts:504-560`).

Alert text is built from counts, codes, labels and dates only. The snapshot the rules read has no payload and no job entity (`core/pure/alert-rules.ts:11-15`).

### `queue_backlog`

The rule is one alert for the whole queue: entity type `platform`, entity id `queue`, so its fingerprint is `queue_backlog:platform:queue` (`core/pure/alert-rules.ts:426-433`).

- **What is counted.** Live jobs in the "due" queue state, and when the oldest of them became due: `min(coalesce(next_attempt_at, created_at))`. The WHERE is the one behind the Jobs list filter `queue=due` (`queueStateWhere("due", ...)` plus the live scope), so the count in the alert equals the total of that list (`core/jobs/alerts.ts:201-210`). Held, waiting and test-scope jobs are not counted (`core/jobs/alerts-backlog.test.ts:136`).
- **Age first.** Nothing fires while the oldest due job has waited less than 15 minutes, however many are due (`core/pure/alert-rules.ts:421`). From 15 minutes the alert is a warning. It is critical at 60 minutes, or when 500 or more jobs are due. The count level still needs the 15 minute age (`core/pure/alert-rules.ts:420-423`, `core/jobs/alerts-backlog.test.ts:128`).
- **Message.** `212 jobs are due and waiting; the oldest has been due for 74 minutes` (`core/pure/alert-rules.ts:211-212`).
- **Resolves** on the first run that finds no due job older than 15 minutes (`core/jobs/alerts-backlog.test.ts:147`).
- **No link from the alert to the Jobs list.** The alert row does not open the filtered list. To see the jobs, open the Jobs page with the filter `queue=due`, or open the Queue page, whose "Due now" card links to the same list (the queue backlog checklist in the operations runbook, in the source repository).
- **In the demo.** The seeded world has two live pending jobs about 22 hours old, so the demo opens one critical `queue_backlog` alert (`src/admin-api/demo/engine-alerts.test.ts:50-52`).

## When the rules are evaluated

An evaluation reads eight bounded snapshots, turns them into firings, and reconciles the `alert` table inside one transaction guarded by a transaction advisory lock. A run that does not get the lock writes nothing (`core/jobs/alerts.ts:37-47`, `core/jobs/alerts.ts:265-278`). The database is the second layer: every open is an upsert on the partial unique index, so a run without the lock still cannot write a second OPEN row (`core/jobs/alerts.ts:305-319`).

What starts an evaluation depends on whether the scheduler is dormant or armed.

| State | What runs the evaluation | Window |
|---|---|---|
| **Dormant** (the QStash tick is not armed; the production default) | The throttled after-response run. It is started after the response of `notifications.list`, `alerts.list` and `systemHealth.get` (`core/http/admin-handlers.ts:269-275`), and after an inbound call finishes (`src/app/api/inbound/[systemCode]/[endpoint]/route.ts:242`) | At most once per 300 seconds across instances (`core/pure/alert-rules.ts:49`, `core/jobs/alerts.ts:516-553`) |
| **Armed** (QStash configured and the tick armed) | The tick stage `alerts`, which runs after the stage `healthChecks` (`src/app/api/jobs/tick/route.ts:75-76`). The after-response run still happens as well | 240 seconds for the tick, so a tick that lands a few seconds early does not skip (`core/jobs/alerts.ts:50-52`) |
| **Either** | The admin action "Evaluate now" (`alerts.evaluate`, `settings:write`). It ignores the window, still takes the advisory lock and stamps the window (`core/jobs/alerts.ts:555-573`) | None |

Consequences with the tick dormant:

- Alerts are only as fresh as the traffic that triggers them. With no console use and no inbound calls, nothing evaluates.
- `tick_stale` stays quiet while the tick is not armed (`core/pure/alert-rules.ts:365`).
- `system_down` needs a health result. The scheduled health check (below) is what produces one without a human click.

The after-response triggers sit behind `alertTriggerEnabled()`, a module flag that is on in production and off under test (`core/jobs/alert-trigger.ts:13-21`). Every run is recorded as an `alerts` row in `process_log`. The summary of that row carries the transition counts and, since P7b, `delivered`, `deliveryFailed` and `deliveryThrottled` (`core/jobs/alerts.ts:495-509`).

## Scheduled health checks (A10)

Before P7b a health URL was probed only when a person pressed "Check now", so `system_down` could only fire after a click. The scheduled check runs the same probe, `probeHealthUrl` (`core/services/systems.ts:326-357`): `safeFetch` with a 5 second timeout and no redirects, the result stored as `UP` or `DOWN` plus a fixed reason, and one conditional write that never overwrites a health URL edited meanwhile.

| | Dormant (tick off) | Armed (tick on) |
|---|---|---|
| Where it runs | After the response of the System health read (`systemHealth.get`), in one deferred job: the health check first, then the alert evaluation, so `system_down` sees the fresh result (`core/http/admin-handlers.ts:124-143`, `core/http/admin-handlers.ts:275`) | The tick stage `healthChecks`, before the stage `alerts` (`src/app/api/jobs/tick/route.ts:74-76`). The after-response path still runs too |
| Window | At most once per 15 minutes (`core/jobs/health-checks.ts:34`) | At most once per 240 seconds (`core/jobs/health-checks.ts:35`) |
| Systems per run | 3 (`core/jobs/health-checks.ts:37`) | 5 (`core/jobs/health-checks.ts:38`) |

- **Dormant means nobody opens the page, nobody is probed.** With the tick off, only the after-response path runs, and it starts only when someone reads System health. The other two alert-trigger reads (`notifications.list`, `alerts.list`) and inbound calls start the alert evaluation only, not a probe (`core/http/admin-handlers.ts:128-129`).
- **Both paths share one stamp.** The window is a compare-and-set on the `app_setting` row `health_checks_last_run`, so concurrent or double triggers run once per window. A run by one path also blocks the other until its own window has passed (`core/jobs/health-checks.ts:103-128`).
- **Which systems.** `ACTIVE`, non-deleted systems that have a health URL and whose last check is older than 15 minutes. Never-checked systems come first, then the oldest check, then the id (`ORDER BY health_checked_at ASC NULLS FIRST, id`) (`core/jobs/health-checks.ts:58-78`). Each system is therefore probed about every 15 minutes at the most.
- **Record.** One process row of kind `health` with the counts `checked`, `up`, `down` and `changed`. There is no actor, so no audit row (`core/jobs/health-checks.ts:130-136`, `core/jobs/process-log.ts:24`). Each probe writes an outbound activity row for the system, marked as a test run (`core/services/systems.ts:339`).

## Notifications

Notifications are written only by alert transitions: `opened` at the alert's severity, `escalated` at the new severity, and `resolved` as `info` (`core/jobs/alerts.ts:387-399`). Read state is per user (`notification_read`). The list keeps the newest 1,000 (`pruneNotificationsToCap`). Resolved alerts and notifications older than `jobDays` are pruned inside the retention kill switch; an OPEN alert is never pruned ([security.md](security.md#alerts-notifications-and-system-health-p3c)).

A notification is the in-app record. It does not depend on delivery: if a webhook POST fails, the notification is still there.

## Delivery to a webhook (G4, P7b)

An operator can have the transitions of an evaluation posted to one webhook URL. It is off by default.

### The flow

1. The evaluator commits its transaction and returns the transitions it wrote (`opened`, `escalated`, `resolved`) (`core/jobs/alerts.ts:259-265`).
2. Delivery runs, then notification pruning. The order is deliberate: a transition is one-shot, so a failing prune must not cost the webhook its only chance to see it (`core/jobs/alerts.ts:484-493`).
3. `deliverAlerts` reads the stored setting. Disabled, or no URL: nothing happens. It filters the transitions by the minimum severity and by `includeResolved` (a resolve is judged by the alert's own severity) (`core/jobs/alert-delivery.ts:187-195`, `core/pure/alert-delivery.ts:77-82`).
4. It opens the sealed URL in memory. If the URL will not open (the key it was sealed under is not the one in use) the run records `URL_UNREADABLE` and sends nothing (`core/jobs/alert-delivery.ts:197-201`).
5. It claims the keys that no recent send covers (the throttle, next section).
6. It sends one POST for everything it claimed and records the outcome (`core/jobs/alert-delivery.ts:203-211`).

The evaluator's transaction is never open during the POST (`core/jobs/alerts.ts:484-485`). `deliverAlerts` never throws: a failure is logged with safe fields only and counted as a failed delivery (`core/jobs/alert-delivery.ts:212-215`). Delivery runs for every evaluation, including the "Evaluate now" action.

### The 30 minute throttle and the claim

The same alert transition is sent again only after 30 minutes (`ALERT_DELIVERY_THROTTLE_MS`, `core/pure/alert-delivery.ts:13-14`). The throttle key is:

```
<rule>:<entityType>:<entityId>:<kind>:<severity>
```

for example `queue_backlog:platform:queue:opened:warning` (`core/pure/alert-delivery.ts:84-86`). Because the kind and the severity are part of the key:

- An escalation from warning to critical is a different key from the open, so it is sent at once.
- An alert that resolves and opens again within 30 minutes has the same `opened` key, so the second open is not sent.
- Two notes with one key inside a single evaluation are sent once (`core/pure/alert-delivery.ts:100-120`).

The state is the `app_setting` row `alert_delivery_state`: `{ sent: { <key>: epochMs }, last }`. A stamp older than 30 minutes, or more than 60 seconds ahead of now (clock skew), is dropped on the next claim, so the map stays bounded (`core/pure/alert-delivery.ts:88-98`).

The claim is the concurrency control. It runs in one short transaction: the state row is inserted first with `ON CONFLICT DO NOTHING`, so the lock has a row to hold, then read with `SELECT ... FOR UPDATE`, and the keys are stamped before the POST. Two runs therefore cannot send the same key (`core/jobs/alert-delivery.ts:100-125`).

### The POST

- **One POST per evaluation**, however many transitions it holds (`core/jobs/alert-delivery.ts:207`).
- **At most 20 items**, plus a `more` count of the rest (`core/pure/alert-delivery.ts:15-16`, `core/pure/alert-delivery.ts:137-157`).
- **Through `safeFetch`**: the SSRF guard (private and reserved addresses refused, https only in production), no redirects, a 5 second timeout (`core/jobs/alert-delivery.ts:47`, `core/jobs/alert-delivery.ts:147-170`).
- **Headers**: `content-type: application/json` and `X-Interchange-Event: alerts` (`core/jobs/alert-delivery.ts:157`).
- **Activity row**: the outbound call is logged with the endpoint label `alert-delivery` and no system code, so a webhook path, which is a secret, is never stored (`core/jobs/alert-delivery.ts:162`).
- **The answer is never read.** The response body is cancelled unread. Success is any 2xx status. The outcome is stored as `HTTP_<status>` or the guard's fixed code (`BLOCKED`, `TIMEOUT`, `CONNECT`, `DNS`, `TLS` and so on) (`core/jobs/alert-delivery.ts:141-170`).

### The payload

```json
{
  "text": "Interchange alerts: 1 opened, 1 resolved (1 critical, 1 warning)",
  "source": "interchange",
  "test": false,
  "items": [
    {
      "kind": "opened",
      "rule": "queue_backlog",
      "title": "Queue backlog",
      "severity": "critical",
      "message": "212 jobs are due and waiting; the oldest has been due for 74 minutes",
      "systemCode": null,
      "alertId": "5b0f6f0e-3c1a-4a52-9f4e-0d6c2b8e7a11",
      "at": "2026-11-17T09:30:00.000Z"
    },
    {
      "kind": "resolved",
      "rule": "dlq_open",
      "title": "Dead letters waiting",
      "severity": "warning",
      "message": "3 dead letters waiting for review on EXAMPLE-SYSTEM",
      "systemCode": "EXAMPLE-SYSTEM",
      "alertId": "c2a9d3b4-7e65-4c1b-8a20-91f3d5e4b6c7",
      "at": "2026-11-17T09:30:00.000Z"
    }
  ],
  "more": 0
}
```

`buildDeliveryBody` copies named fields only, never a spread, so nothing but the alert's own title and message (counts, codes, labels, dates) can reach the wire (`core/pure/alert-delivery.ts:7-10`, `core/pure/alert-delivery.ts:138-157`). Notes:

- `text` is a one-line summary: the counts by kind (opened, escalated, resolved), then by severity (`core/pure/alert-delivery.ts:124-135`).
- `at` is the time of the evaluation, the same on every item.
- `systemCode` is `null` for the platform-level rules (`tick_stale`, `queue_backlog`).
- A resolved item carries the alert's own severity, not `info` (the in-app notification for a resolve is `info`).
- `more` is the number of notes beyond the 20 items.

### Decisions to know before you rely on it

These are deliberate. Each one costs something, so each is stated here.

1. **A failed POST keeps its claim and is not retried.** A non-2xx answer, a guard refusal or a timeout is recorded as the last result, and the keys stay stamped. This avoids hammering a receiver that is down. A transition is final per key: it is not re-sent when the receiver comes back. The in-app notification remains (`core/jobs/alert-delivery.ts:32-37`, `core/jobs/alert-delivery.ts:203-215`). If the URL cannot be opened (`URL_UNREADABLE`) nothing is claimed, but the transitions of that run are lost to the webhook for the same reason: a note is one-shot.
2. **Notes beyond 20 in one evaluation are counted in `more`, not sent later.** All claimed keys are stamped, and a note exists only for its own transition, so a note cut by the cap is not re-sent by a later run. The receiver learns that something was cut from `more`, and the console shows the rest.
3. **There is no signature header.** The receiver cannot verify that a request came from Interchange. The only protection is that the URL is secret, so use a long, unguessable path or token in it and treat it as a password. Signing the body is a later item.
4. **The Test button sends a fixed fictional body.** It is `deliveryTestBody`: rule `test`, system code `EXAMPLE-SYSTEM`, an all-zero alert id, `test: true` (`core/pure/alert-delivery.ts:159-179`). It is never built from stored data. The button is disabled while the form has unsaved changes, because the test goes to the saved URL (`src/features/health/AlertDeliveryPanel.tsx:221-224`). Without a saved URL the test answers `NOT_CONFIGURED`. With a URL that will not open (the master key was changed without the re-key) it answers `URL_UNREADABLE` (`core/services/alert-delivery.ts:112-124`).
5. **The URL is moved by `scripts/rekey.ts` on a master key rotation.** Saving this setting while a re-key runs is a read-modify-write race: the save may write back the old blob. Keep the console quiet during a re-key (the master key rotation in the operations runbook, in the source repository).
6. **No link from a `queue_backlog` alert to the Jobs list filter.** See above.

### The setting and the secret

The setting is the `app_setting` row `alert_delivery`: `{ enabled, minSeverity, includeResolved, url, urlHost }`. The URL is sealed with the vault under the purpose `alert-webhook` and the AAD `app_setting:alert_delivery`. Only the host is ever shown (`core/jobs/alert-delivery.ts:22-31`, `core/jobs/alert-delivery.ts:43-47`). No response, audit row, log line or error message carries the URL or the receiver's answer (`core/services/alert-delivery.ts:25-29`). See [security.md](security.md#alert-delivery-webhook-p7b).

### Admin endpoints

| Endpoint | Method and path | Permission | Audit |
|---|---|---|---|
| `alertDelivery.get` | `GET /settings/alert-delivery` | `alerts:read` | none |
| `alertDelivery.save` | `POST /settings/alert-delivery` | `settings:write` | `alert_delivery.save` |
| `alertDelivery.test` | `POST /settings/alert-delivery/test` | `settings:write` | `alert_delivery.test` |

(`src/admin-api/endpoints/alerts.ts:94-133`.) `get` answers `{ enabled, minSeverity, includeResolved, urlSet, urlHost, last }`, never the URL.

`save` takes `enabled`, `minSeverity`, `includeResolved` and an optional `url`. Omitting `url` keeps the stored one and `null` clears it. A new URL must parse, start with `https://`, be at most 2,048 characters and pass the same SSRF check as an endpoint URL. Otherwise the answer is `VALIDATION` with the issue on the path `url`: a URL over 2,048 characters is refused by the input schema (`src/admin-api/endpoints/alerts.ts:116`), and the service then reports `not_https`, `invalid` or the blocked-address message (it also checks `too_long`, for callers that skip the schema). Turning delivery on with no URL is `VALIDATION` too (issue code `required`) (`core/services/alert-delivery.ts:61-79`). The audit row records the flags and whether a URL is `set` or `cleared`, never the URL (`core/services/alert-delivery.ts:53-59`).

`test` answers `{ delivered, code }`. It is limited to 5 a minute per user, then `RATE_LIMITED` with `Retry-After` ([rate-limiting.md](rate-limiting.md)). It is not subject to the 30 minute throttle and it claims no key. Its result is stored as the last result with `test: true` (`core/services/alert-delivery.ts:102-134`).

## Set it up

Delivery is configured on the System health page, in the "Alert delivery" panel below the health summary (`src/features/health/SystemHealthPage.tsx:238`). Anyone with `alerts:read` sees the state. Changing it and sending a test need `settings:write` (admin only) (`src/features/health/AlertDeliveryPanel.tsx:41-46`).

1. Create an incoming webhook at the receiver (a chat tool or any HTTPS endpoint that accepts a JSON POST and answers 2xx). Treat its URL as a secret (see decision 3).
2. Open System health, find "Alert delivery", and turn on "Post alerts to the webhook".
3. Choose the minimum severity (`info`, `warning` or `critical`). Tick "Include resolved alerts" if you want the all-clear too.
4. Paste the URL into "Webhook URL" (https only) and press Save. The panel now reads `Set, host <host>`. The URL is never shown again. To change it press Replace; to remove it press Clear then Save; leaving the box empty keeps the stored URL.
5. Press "Send test". Expect `Delivered (HTTP_200)` (or the status your receiver answers).

The panel also shows the last send: when, whether it was delivered, the code, and whether it was a test or an alert (`src/features/health/AlertDeliveryPanel.tsx:37-39`).

## Testing it

- **The test send.** Step 5 above. A `Not delivered: BLOCKED` means the guard refused the address (a host that resolves to a private or reserved range). `TIMEOUT` means the receiver did not answer in 5 seconds. `HTTP_4xx` or `HTTP_5xx` is the receiver's own refusal. `URL_UNREADABLE` means the key the URL was sealed under is not the one in use: save the URL again, or finish the re-key ("Alert not delivered" in the operations runbook, in the source repository).
- **A real alert.** Set the minimum severity to `info`, then use "Evaluate now" on a condition you can create safely in a test deployment (for example a health URL that answers 500, which opens `system_down` after a check). A second evaluation inside 30 minutes sends nothing for the same key.
- **The automated suites.** Delivery, the throttle, the claim and the no-URL-leak checks are covered by the unit suite (the P7b tests in the testing guide, in the source repository).

## Verified live

One run against the production deployment on 2026-11-17 (commit `72f5ae1`), with fictional probe systems and a public request-inspection bin as the receiver. The probe rows were removed afterwards and the create path was run once more, clean.

| Check | Result |
|---|---|
| Receiver | A public request-inspection bin; its URL was cleared from the setting at the end of the run |
| Settings read back | `urlSet: true`, `urlHost` the bin's host, no URL field in the answer |
| Test send | `{ delivered: true, code: "HTTP_200" }`; the bin received one request with `test: true` |
| Real alerts opened and posted | `queue_backlog` (warning, once a retrying job had been due for 17 minutes), `system_down` (warning, from the scheduled health check, no Check now) and `dlq_open` (warning); each reached the bin as an `opened` item |
| Resolves posted | With "Include resolved" on, the three resolves reached the bin: `queue_backlog` when the job moved to dead letters, the other two after clean-up |
| Last result shown | `ok`, `HTTP_200`, `test: false` (the last real POST) |
| URL absent | The audit rows of the save and the Test call, and the alert run summaries, hold neither the bin id nor a URL path |
| Not run live | The 30 minute throttle, an escalation and a failing receiver. They are covered by the unit suite: `core/jobs/alert-delivery.test.ts:226` (sends, throttles at 29 minutes, sends again at 31), `:280` (a `500` is recorded and not retried in the window) and `core/jobs/alerts.test.ts:225` (escalation) |

## Related

- The operations runbook (in the source repository): alert delivery and health checks in operation, the queue backlog checklist and the re-key procedure.
- [security.md](security.md): the sealed webhook URL and the SSRF guard.
- [architecture.md](architecture.md): the endpoint registry and the P7b summary.
- [rate-limiting.md](rate-limiting.md): the Test limiter.
- The testing guide (in the source repository): the P7b tests and the load test.
