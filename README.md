# Real-Time-Cloud-Based-Event-Planning-RSVP-Tracker
Real-time event planning and RSVP tracker built for Google Colab. Python + SQLite backend with role-based access, capacity-safe transactions, FIFO waitlist, announcements, notifications and audit logs, plus a live dashboard with analytics charts, simulated traffic and an automated test suite including a race-condition test.
# Real-Time Cloud-Based Event Planning & RSVP Tracker

![Python](https://img.shields.io/badge/Python-3.10%2B-blue) ![SQLite](https://img.shields.io/badge/Database-SQLite-lightgrey) ![Platform](https://img.shields.io/badge/Runs%20on-Google%20Colab-orange) ![Status](https://img.shields.io/badge/Data-Synthetic-green)

A real-time event planning and RSVP platform that runs entirely inside a single Google Colab cell. It combines a transactional Python backend with a live, interactive dashboard, and demonstrates the core ideas behind cloud-hosted event systems: role-based access control, real-time updates, capacity enforcement, concurrency safety and audit logging.

---

## Table of Contents
1. [Problem Statement](#problem-statement)
2. [Features](#features)
3. [Architecture](#architecture)
4. [Tech Stack](#tech-stack)
5. [Quick Start](#quick-start)
6. [Using the Dashboard](#using-the-dashboard)
7. [Database Design](#database-design)
8. [Backend Actions (API)](#backend-actions-api)
9. [Concurrency Handling](#concurrency-handling)
10. [Security](#security)
11. [Testing](#testing)
12. [Mapping to Cloud Concepts](#mapping-to-cloud-concepts)
13. [Scalability Notes](#scalability-notes)
14. [Limitations](#limitations)
15. [Future Improvements](#future-improvements)
16. [Screenshots](#screenshots)
17. [Author](#author)

---

## Problem Statement
Spreadsheets and chat-group lists break down for event management: duplicate entries, no live headcount, overbooking, and last-minute confusion. This project centralizes events and RSVPs in one database, enforces one response per person, protects capacity under simultaneous requests, and gives organizers a live view of attendance.

## Features

**Organizer**
- Create events with name, venue, date, capacity and registration deadline
- Live dashboard: Going / Maybe / Not Going / Waitlist counts
- Analytics: response rate, RSVP conversion, capacity utilization, seats left, RSVP growth chart
- Publish announcements (attendees who RSVPed are notified automatically)
- Cancel events (all responders are notified)
- View the attendee list (visible only to the event's own organizer)

**Attendee**
- RSVP as Going, Maybe or Not Going, and change the response at any time
- Automatic waitlist when an event is full, with FIFO promotion when a seat opens
- In-app notifications for RSVP confirmation, waitlist promotion, announcements and cancellation
- View event details and announcements

**Platform**
- Event status lifecycle: `PUBLISHED`, `FULL`, `CANCELLED`
- Role-based authorization enforced in the backend
- Audit log of key actions
- Built-in automated test suite, including a multi-thread race-condition test
- Traffic simulator to generate random RSVPs for demos

## Architecture

```
 Browser (Colab output)                     Python kernel (backend)
┌──────────────────────────┐   invokeFunction   ┌───────────────────────────────┐
│ Dashboard UI (HTML/JS)   │ ─────────────────► │ api() dispatcher              │
│  - polls every 2 seconds │                    │  ├─ RBAC checks               │
│  - renders charts/tables │ ◄───────────────── │  ├─ validation                │
└──────────────────────────┘      JSON          │  └─ rsvp / events / announce  │
                                                └──────────────┬────────────────┘
                                                               │ BEGIN IMMEDIATE
                                                        ┌──────▼──────┐
                                                        │  SQLite DB  │
                                                        └─────────────┘
```

**RSVP flow:** attendee clicks a button → JS calls the backend → backend opens a write transaction, validates the request, checks capacity, writes the RSVP, creates notifications, updates event status, logs the action, commits → the next dashboard poll shows the new counts.

## Tech Stack
| Layer | Technology |
|---|---|
| Backend | Python 3, `sqlite3`, `threading` |
| Database | SQLite (WAL mode) |
| Frontend | HTML, CSS, vanilla JavaScript, inline SVG charts |
| Bridge | `google.colab.output.register_callback` |
| Real-time | Short polling (2 s) |

No external packages or API keys are required.

## Quick Start
1. Open [Google Colab](https://colab.research.google.com) and create a new notebook.
2. Paste the full contents of `rsvp_tracker_colab.py` into a single cell.
3. Run the cell. The dashboard appears directly below it.

> The dashboard relies on Colab's Python-to-JavaScript callback, so it needs to run in Google Colab (not a plain Jupyter notebook or a local Python script).

Each run resets the database and seeds fresh dummy data: 2 organizers, 20 attendees and one sample event ("Cloud Computing Workshop", capacity 10).

## Using the Dashboard
Use the **Acting as** dropdown to switch users and test roles without real logins.

| Tab | Purpose |
|---|---|
| **Dashboard** | KPI cards, RSVP distribution donut, capacity bar, growth line, attendee table |
| **RSVP / Attendee** | Submit or change an RSVP, read announcements and notifications |
| **Manage** | Create events, publish announcements, cancel events, simulate random RSVPs |
| **Tests & Logs** | Run the automated test suite and view the audit log |

**Suggested demo**
1. Stay as *Olivia Organizer* and open **Dashboard**.
2. Go to **Manage** and click **+5 random RSVPs**. Switch back and watch the counters update without a refresh.
3. Click **+12 RSVPs** to fill the event; later Going requests are waitlisted and the status becomes `FULL`.
4. Switch to an attendee and change an RSVP in the **RSVP** tab; check their notifications.
5. Switch back to the organizer and publish an announcement.
6. Open **Tests & Logs** and run the test suite.

## Database Design

```
users (id PK, name, role)
   │ 1
   │ creates            ┌────────────────────────────┐
   ▼ N                  │ announcements (event_id)   │
events (id PK, organizer_id FK, name, description, venue,
        event_date, capacity CHECK>0, deadline, status, created_at)
   │ 1                  └────────────────────────────┘
   │ has many
   ▼ N
rsvps (id PK, event_id FK, user_id FK, status, wseq,
       responded_at, updated_at, UNIQUE(event_id, user_id))

notifications (id PK, user_id, event_id, type, message, created_at)
audit_logs    (id PK, user_id, action, detail, created_at)
```

- **Unique constraint:** `UNIQUE(event_id, user_id)` guarantees a single RSVP row per user per event at the database level.
- **Check constraints:** role and RSVP status values are restricted; capacity must be greater than zero.
- **Index:** `idx_rsvp_event_status` on `(event_id, status)` speeds up count queries.
- **Waitlist ordering:** the `wseq` timestamp defines FIFO order; it is preserved if a waitlisted user re-submits.

## Backend Actions (API)
All calls go through one dispatcher, `api()`, which receives JSON from the dashboard.

| Action | Who | Description |
|---|---|---|
| `state` | Any user | Returns events, stats, timeline, announcements, notifications and (for owners) the attendee list |
| `rsvp` | Attendee | Create or update an RSVP (`GOING`, `MAYBE`, `NOT_GOING`) |
| `cancel_rsvp` | Attendee | Withdraw a response (recorded as `NOT_GOING`) |
| `create_event` | Organizer | Validates dates, capacity and deadline, then creates an event |
| `announce` | Event owner | Publishes an announcement and notifies responders |
| `cancel_event` | Event owner | Cancels the event and notifies responders |
| `simulate` | Event owner | Generates random RSVPs for demos |
| `tests` | Any user | Runs the automated test suite |

Errors are returned as HTTP-style messages such as `400 Invalid status`, `403 Forbidden`, `404 Event not found` and `409 Registration deadline passed`.

## Concurrency Handling
A naive flow like "read current Going count, then insert RSVP if below capacity" has a race condition: with 99 of 100 seats taken, two simultaneous requests can both read 99 and both be accepted, giving 101.

This project avoids it by doing the check and the write inside a single transaction:

```python
c.execute("BEGIN IMMEDIATE")   # acquire the write lock before reading
going = c.execute("SELECT COUNT(*) FROM rsvps WHERE event_id=? AND status='GOING'", (eid,)).fetchone()[0]
if going >= capacity:
    final = "WAITLISTED"
# ... upsert RSVP, create notifications, update event status ...
c.execute("COMMIT")
```

Writers are serialized, so the second request sees the updated count and is waitlisted. On any error the transaction is rolled back. In a managed cloud database the same guarantee comes from a transaction, a conditional write or row-level locking (for example Firestore transactions or `SELECT ... FOR UPDATE` in PostgreSQL).

## Security
- **RBAC:** organizers manage only events they own; attendees cannot create events, post announcements or view other people's responses.
- **Server-side source of truth:** the browser only sends requests; counts and capacity are computed in the backend.
- **Input validation:** status values, dates, capacity and required fields are validated before writing.
- **Parameterized SQL:** every query uses placeholders, which prevents SQL injection.
- **Output escaping:** the dashboard escapes all user-supplied text before rendering, which mitigates XSS.
- **Audit trail:** RSVPs, event creation, announcements and cancellations are logged.

> The "Acting as" selector is a demo stand-in for real authentication. A production deployment would use a managed identity service (for example Firebase Auth, Cognito or Supabase Auth) with verified tokens.

## Testing
Open the **Tests & Logs** tab and click **Run test suite**. The suite currently checks:

| Test | Expected result |
|---|---|
| 15 users request the last seat at the same time | Exactly 1 accepted, the rest waitlisted |
| Submit the same RSVP twice | Only 1 database row exists |
| Update MAYBE → GOING | Update succeeds |
| Attendee tries to create an event | `403` rejected |
| Different organizer tries to cancel the event | `403` rejected |
| Attendee tries to post an announcement | `403` rejected |
| Going attendee cancels while a waitlist exists | First waitlisted user is promoted to Going |
| Create an event with capacity 0 | `400` rejected |

The race test creates a temporary event named "Race Test (1 seat)", which remains in the event list afterward. Restart the cell to reset all data.

## Mapping to Cloud Concepts
| Concept | How it appears here | Cloud equivalent |
|---|---|---|
| Managed database | SQLite with constraints and indexes | Firestore, Cloud SQL, RDS, DynamoDB |
| Real-time updates | 2-second polling | Firestore listeners, WebSockets, SSE |
| Authorization | Role and ownership checks | Cognito / Firebase rules / IAM |
| Transactions | `BEGIN IMMEDIATE` | Firestore transactions, conditional writes |
| Event-driven notifications | Notifications created on RSVP, announcement and cancellation | Cloud Functions, SNS, FCM |
| Audit and monitoring | `audit_logs` table | CloudWatch, Cloud Logging |

## Scalability Notes
This demo runs on a single node. For very large events (for example 100,000 RSVPs in a few minutes), a production design would add a load balancer with autoscaling or serverless functions, a managed database, a cache for event details, a queue for notifications and analytics, WebSocket or push infrastructure instead of polling, a CDN for static assets and rate limiting. Hot spots on a single event counter can be reduced with sharded counters or by reserving seats through a queue.

## Limitations
- Runs only inside Google Colab; data is stored on the temporary Colab disk and lost when the runtime resets.
- Real-time updates use polling, not server push.
- User switching is simulated; there is no password-based login.
- No email, SMS or push delivery; notifications are in-app only.
- The test suite leaves its temporary race-test event in the database.

## Future Improvements
- Migrate to React + FastAPI with Firebase/Supabase for real authentication and listeners
- Unique invite links and QR codes with on-site check-in
- Email, SMS and push notifications
- Scheduled reminders and calendar (`.ics`) attachments
- Attendance prediction (for example logistic regression) to target reminders
- CI/CD, monitoring and infrastructure as code

## Screenshots
Add your screenshots to a `screenshots/` folder and link them here.

| Screenshot | What it shows |
|---|---|
| `01_organizer_dashboard.png` | KPIs, charts and live counts |
| `02_attendee_rsvp.png` | RSVP and notifications |
| `03_full_event_waitlist.png` | `FULL` status and waitlisted users |
| `04_unauthorized_rejected.png` | `403` protection |
| `05_test_results.png` | Passing test suite, including the race test |

## Author
**Debankita Panja** 
GitHub: https://github.com/dp2005-lang · LinkedIn: https://www.linkedin.com/in/debankita-8482a2403/

---

*All users, events and RSVPs in this project are synthetic and created for demonstration purposes only.*
