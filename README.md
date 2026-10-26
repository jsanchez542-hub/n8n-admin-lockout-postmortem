# Postmortem: n8n admin lockout after an owner account reset

A report on a production incident. An n8n instance on a private server became unreachable from the browser after its owner account was rebuilt, while the API, the database and every scheduled workflow kept running normally.

Published because the failure mode is misleading (it presents as an authentication problem but is a referential integrity problem) and the error that finally identified it is not documented anywhere I could find.

**Incident date:** 15 August 2026
**Time to resolution:** approximately 1 hour 20 minutes
**Data loss:** none. All 24 workflows, their credentials and their scheduled executions kept running throughout.

**Environment:** n8n 2.33.x on a private server, Docker Swarm, Traefik reverse proxy, SQLite backend.

---

## 1. Summary

What started as a forgotten password escalated into a frontend that would authenticate successfully and then bounce straight back to the login screen, producing no authentication errors of any kind on the server.

The root cause: **n8n resolves a user's personal project through the `project.creatorId` column, not through the `project_relation` table.** After the owner account was rebuilt, `project_relation` was repaired but `creatorId` still pointed at the deleted user. The endpoint `GET /rest/projects/personal` returned 404, the editor's initialisation threw inside a `Promise.all`, and the router redirected back to the login page.

Because authentication itself never failed, the server logs showed nothing useful. That single misleading signal accounted for most of the resolution time.

---

## 2. Timeline

| Time (UTC) | Event |
|---|---|
| 11:45 | Login rejected with "Wrong username or password" |
| 11:50 | Unrelated real bug found and fixed: `N8N_PROXY_HOPS` was unset, breaking the login rate limiter behind the proxy |
| 12:00 | No SMTP configured, so email password recovery is unavailable. CLI reset is the only path |
| 12:06 | `n8n user-management:reset` runs but only partially succeeds. It leaves a residual user row with 2FA still enabled, so the setup wizard never appears |
| 12:10 | A manual repair that went too far: the entire user row is deleted. This breaks the project to user link |
| 12:20 | A second reset fails with `Could not find any entity of type "Project" matching: { "type": "personal", "creatorId": ... }`. This message described the root cause and was not followed up at the time |
| 12:25 | `project_relation.userId` repaired. Setup wizard appears, new owner account created |
| 12:29 | Account created successfully, but the editor will not load. Login succeeds, then returns to the login screen |
| 12:29 to 13:20 | Hypothesis testing, see section 4 |
| 13:25 | Browser console reveals `GET /rest/projects/personal 404` and `Could not find a personal project for this user` |
| 13:30 | `project.creatorId` corrected along with 725 further orphaned references. Resolved |

---

## 3. Diagnostic method

The working rule was to avoid changing production on a guess. Every hypothesis had to be supported by observable evidence before any action was taken.

- Container logs read with timestamps and correlated against the exact moments of real login attempts
- SQLite queries that only read the database (`sqlite3.connect('file:...?mode=ro', uri=True)`) to inspect state without risk to a live database
- Reading the deployed source itself (`auth.service.js` inside the container) rather than relying on documentation or forum posts
- Reproducing the full login cycle with `curl`, isolating one variable at a time
- Temporary debug logging to capture one specific user attempt end to end
- The browser console, which turned out to be the only blind spot the server could not cover

---

## 4. Hypotheses ruled out

The dead ends are recorded too, each with the evidence that ruled it out.

| Hypothesis | How it was ruled out |
|---|---|
| Password was simply wrong | A `curl` request with a deliberately wrong password returned an identical message, proving the message is generic and carries no diagnostic value |
| Rate limiter broken by proxy misconfiguration | A genuine bug, fixed with `N8N_PROXY_HOPS=1`. The errors stopped and the login still failed |
| Cookie rejected due to HTTPS or proxy handling | `/rest/settings` already reported `authCookie.secure: true`. `N8N_PROTOCOL=https` was added as good practice with no effect on the symptom |
| `browserId` validation failing | The `browserId check failed` warning appeared only in my own isolated curl test, never during a real user attempt |
| Multiple instances splitting session state | Verified as a single replica, one running container |
| Instance level 2FA enforcement rejecting the new account | No MFA setting in the `settings` table, no MFA environment variables |
| Browser tracker blocking | Plausible, since it failed in two different browsers, but disproved by the actual console error |
| Incomplete onboarding hanging the survey modal | `settings` and `personalizationAnswers` were indeed NULL and were populated as a precaution, but this was not the cause |

Three of these were reasonable and consistent with every symptom available at the time. None were correct. Progress only came from client side evidence.

---

## 5. Root cause

n8n resolves a personal project with a query equivalent to:

```sql
SELECT * FROM project WHERE type = 'personal' AND creatorId = :userId
```

After the account rebuild, the database state was:

```
project.creatorId        = <deleted user>   causes the 404
project_relation.userId  = <new user>       repaired
shared_workflow          = 24 workflows correctly attached to the project
```

The relation was correct. The `creatorId` was not. The frontend issues that call inside a `Promise.all` during initialisation (`init.ts` into `router.ts`), so the 404 threw, aborted startup, and redirected to the login screen.

The reason this took so long to isolate: the failure presented as an *authentication* problem while actually being a *referential integrity* problem. Authentication worked correctly the entire time, which was confirmed by repeated successful authenticated requests issued from the command line.

---

## 6. Resolution

The fix did not stop at the single column that unblocked the symptom. Every table was scanned programmatically for references to the deleted user id, and each one was reassigned.

| Table and column | Rows corrected |
|---|---|
| `project.creatorId` | 1 (the critical one) |
| `workflow_publish_history.userId` | 722 |
| `user_api_keys.userId` | 2 |
| `chat_hub_sessions.ownerId` | 1 |
| **Total** | **726** |

Correcting `user_api_keys` had a secondary benefit. It restored API keys used both by an external integration and by internal workflows that call the n8n API, all of which had been failing silently for hours.

Verification after the fix:

```
POST /rest/login              200
GET  /rest/projects/personal  200   (previously 404)
GET  /rest/workflows          200   (24 workflows)
GET  /rest/projects           200
GET  /rest/credentials        200
```

---

## 7. Configuration issues found along the way

Unrelated to the root cause, but discovered and corrected during the investigation:

- `N8N_PROXY_HOPS=1`. The rate limiter did not trust the reverse proxy, raising `ERR_ERL_UNEXPECTED_X_FORWARDED_FOR` on login attempts
- `N8N_PROTOCOL=https` and `N8N_HOST`. Neither was defined, only `WEBHOOK_URL` was set
- `N8N_DIAGNOSTICS_ENABLED=false`. Telemetry disabled

---

## 8. If you are hitting this same problem

Symptoms that match this incident:

- Login appears to succeed, then the page returns to the login screen
- Server logs show no authentication errors
- The URL accumulates an encoded redirect such as `signin?redirect=%252Fhome%252Fworkflows`
- Browser console shows `Could not find a personal project for this user`

Checks, in order:

1. Open the browser console. If the failing request is `GET /rest/projects/personal` returning 404, this is the same issue
2. Query the database: `SELECT id, type, creatorId FROM project;`
3. Compare `creatorId` against the id of the account you are actually logging in with: `SELECT id, email FROM user;`
4. If they differ, that is the cause

Before repairing, stop n8n, and scan every table for the stale user id instead of correcting only the one that unblocks the symptom.

If `n8n user-management:reset` leaves you unable to reach the setup wizard, the cause is likely residual 2FA on the existing row. Clear only `mfaEnabled`, `mfaSecret` and `mfaRecoveryCodes`. Do not delete the user row, which is what created the referential damage described here.

---

## 9. Lessons

**An early error message contained the answer.** The failing second reset stated `matching: { "type": "personal", "creatorId": ... }` in plain text. It was read as an obstacle to work around. It was describing the data model. Tool error messages describe internal logic and should be read as documentation.

**Deleting is worse than correcting.** Removing the whole user row was more destructive than the situation required and caused the referential damage. Clearing the specific 2FA fields blocking the official reset would have been sufficient.

**Repair by referential integrity, not by symptom.** A programmatic scan found 726 stale references where manual repair had corrected 1.

**The server has blind spots.** Logs, database and deployed source were all exhausted without finding the cause. One look at the browser console found it in seconds. When the backend is verified healthy, move to the client early.

**Separate verified from assumed.** Statements such as "the workflows are intact" were backed by real counts at every step, which made it possible to keep the blast radius clear while the rest of the diagnosis was still uncertain.

---

A Spanish version of this document is available in [POSTMORTEM.es.md](POSTMORTEM.es.md).

Sensitive values (addresses, tokens, credentials) are deliberately omitted throughout.
