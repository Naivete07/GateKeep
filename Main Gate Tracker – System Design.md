# Main Gate Tracker – System Design

Nov 20, 2026 · @Naivete

Main Gate Tracker is a three-tier web system: staff and resident clients call one REST API, which owns all order, OTP and user state in a single relational database and hands notifications to a separate messaging service.

## Architecture

&#91;embedded content: architecture · 3 clients, 4 API modules, database, notifications\]

Clients never touch the database: the API checks the caller's role, writes the change, then asks the notification service to alert the resident.

## Components

Each component has one job and one owner of its data, so a change to OTP rules never touches user accounts.

| Component | Responsibility | Owns |
| --- | --- | --- |
| Gate staff app | Log deliveries, verify OTPs, create collective pickups | Nothing (stateless client) |
| Resident app | Show active OTPs, order status and history | Nothing (stateless client) |
| Admin dashboard | Review records, manage users, look up disputes | Nothing (stateless client) |
| Auth and roles | Register, login, issue and check JWTs, enforce role rules | Credentials, sessions |
| Orders and OTP | Create orders, generate and verify OTPs, move status | Orders, OTP hashes, pickup times |
| Pickup groups | Bundle orders, track per-order collection inside a group | Pickup groups and membership |
| Users | Profiles, lookup by id, listing | User records |
| Notification service | Send the OTP and status alerts to residents | Delivery attempts and retry state |
| Relational database | Single source of truth, with constraints on every link | All persistent data |

## Order lifecycle

&#91;embedded content: order lifecycle · 3 states, 2 routes to Collected\]

Every move forward is time-stamped. An order inside a pickup group still needs its own OTP, so one resident's code never releases another's parcel.

## Data model

Five tables cover everything; an order points to one resident, one staff member and optionally one pickup group.

| Table | Key fields | Links |
| --- | --- | --- |
| users | id, name, email (unique), password\_hash, role, hostel, room\_no, phone, active | Role is resident, staff or admin |
| orders | id, reference\_no, courier, description, item\_count, status, arrival\_time, collected\_time, otp\_hash, otp\_expiry, otp\_attempts | resident\_id → users, handled\_by → users, pickup\_group\_id → pickup\_groups (nullable) |
| pickup\_groups | id, status, created\_at | created\_by → users |
| notifications | id, order\_id, channel, status, attempts, last\_attempt\_at | order\_id → orders |
| audit\_log | id, order\_id, action, timestamp | actor\_id → users |

Indexes: `orders(resident_id, status)` for the resident's open orders, `orders(status, arrival_time)` for the gate's pending list, `users(email)` for login. Rows are never hard-deleted; users are deactivated and orders keep their audit trail.

## Key flows

Three flows carry the product; each runs inside one database transaction so a crash never leaves a half-recorded order.

### Log a delivery

1. Staff submit resident, hostel or room, delivery details; the server stamps the arrival time.
2. The API checks the staff token and role, then finds the resident.
3. It generates an OTP, stores only its hash and expiry, and saves the order as Received.
4. It writes an audit entry and queues a notification with the OTP for the resident.

### Verify an OTP and hand over

1. Staff open the order and enter the code the resident gives.
2. The API compares it against the stored hash, checks expiry and the attempt counter.
3. On a match it marks the order Collected, stores the pickup time and the staff member, and invalidates the OTP.
4. On a miss it increments the attempt counter and locks the order after the limit.

### Collective pickup

1. Staff select several Received orders and create a pickup group.
2. Each order moves to In collective pickup and each resident is notified.
3. At the gate, each resident verifies their own OTP; each order is then Collected on its own time stamp.
4. The group closes when its last order is collected.

## Security

The OTP is the only thing that releases a parcel, so the design protects it at every step.

| Risk | Control |
| --- | --- |
| OTP leaked from the database | Store a hash only, never the code |
| OTP guessed at the gate | Attempt limit per order, lock and flag after it |
| OTP reused or stale | Single use, invalid after collection, expires after a set period |
| Stolen or forged login | Passwords hashed with bcrypt, short-lived JWT, HTTPS everywhere |
| Resident reads another's orders | Role checks on every route; residents see only their own ids |
| Staff hands over without proof | Handover only through the verify route, which records who and when |
| Disputed pickup | Append-only audit log of create, notify, verify and collect |
| Public register abused | Resident sign-up checked against a hostel list, or approved by an admin |

## Notifications and reliability

A failed alert must never block or undo a delivery entry, so notifications run after the order is saved, not inside the same request.

- **Queue first:** the API saves the order, then writes a notification row marked pending; a worker sends it.
- **Retries:** a failed send is retried with growing delays up to a fixed number of attempts, then marked failed and shown to staff.
- **Fallback:** if push fails, the worker tries SMS or email; staff can always read the OTP request from the resident at the gate and use resend.
- **Idempotency:** each notification row has one id, so a retry never sends two different codes.
- **Database safety:** regular backups and a restore test; the audit log is append-only.
- **Peak load:** meal-time rushes mean many writes in minutes; indexes on status and arrival time keep the gate's pending list fast.

## Deployment and scaling

A hostel gate sees tens to a few hundred orders a day, so one small server and one database are enough at launch; the design only has to leave room to grow.

| Concern | Launch setup | Growth path |
| --- | --- | --- |
| Hosting | One API server behind HTTPS, one managed database | Run several API copies behind a load balancer |
| Clients | Mobile-friendly web app for staff and residents | Installable app with push alerts |
| Notifications | Worker process beside the API | Separate worker pool and queue service |
| Environments | Development, staging, production, each with its own database | Same, plus automated deploys |
| Config and secrets | Environment variables, never in code | A secrets manager |
| Monitoring | Error logs and a health route | Alerts on failed notifications and slow verify calls |
| Multiple gates | One gate | Add a gate id to orders and staff |

## Design decisions

| Decision | Why | Cost |
| --- | --- | --- |
| One REST API, not microservices | Small team, small load, one place to enforce rules | Must split later if load grows |
| Relational database | Orders, residents and groups are linked; constraints stop orphan records | Less flexible schema than a document store |
| OTP hashed, not stored | A database leak does not release parcels | Staff cannot read a code back; resend issues a new one |
| Per-order OTP inside groups | One resident's code cannot release another's parcel | Each resident verifies separately at the gate |
| Notifications sent by a worker | Alert failures never block the gate | Slight delay and one more process to run |
| Append-only audit log | Clear answer to who collected what and when | Extra storage |
| Soft delete for users and orders | Records stay available for disputes | Needs filters on every query |
