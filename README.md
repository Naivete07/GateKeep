# 🚪 GateKeep

**A hostel gate management app for receiving, securing and tracking deliveries.**

Gate staff log every incoming order, the app issues a one-time password (OTP) for collection and notifies the resident, and every handover is time-stamped, so parcels stop going missing or reaching the wrong hands.

---

## 📌 Table of Contents

- The Problem
- Features
- How It Works
- Order Lifecycle
- Architecture
- Tech Stack
- Getting Started
- API Reference
- Data Model
- Security
- Roadmap
- Contributing
- Author

---

## ❓ The Problem

In most hostels, deliveries are handled with notebooks, phone calls and informal handovers. That leads to:

- Parcels handed to the wrong person
- Lost orders with no proof of who collected what, or when
- Residents not knowing something has arrived
- Long queues when many orders arrive together
- No audit trail when disputes happen

## ✨ Features

- **Delivery logging**: record resident, hostel or room, delivery details and arrival time
- **OTP-secured collection**: a one-time password is generated for every order
- **Resident notifications**: residents are alerted the moment their order is logged
- **Collective pickups**: group several orders so residents verify their OTPs and collect together
- **Status tracking**: every order moves from *Received* to *Collected*
- **Pickup records**: arrival, verification and collection times are stored for every order
- **Role-based access**: separate views for gate staff, residents and admins
- **Audit trail**: a searchable history to settle any dispute

## 🔄 How It Works

### Single delivery

1. A courier arrives at the gate.
2. Staff create a delivery entry (resident, hostel or room, details). Arrival time is captured automatically.
3. The app generates an OTP and notifies the resident. Status: **Received**.
4. The resident shares the OTP at the gate.
5. Staff verify it; the order becomes **Collected** and the pickup time is saved.

### Collective pickup

1. Several orders are logged individually.
2. Staff select them and create one collective pickup.
3. Residents are notified that their orders are grouped.
4. Each resident verifies their own OTP at the gate.
5. Each order is marked **Collected** with its own time stamp.

## 📦 Order Lifecycle

```
Received ──► In collective pickup (optional) ──► Collected
    │                                                ▲
    └──────────── OTP verified (single pickup) ──────┘
```

An order is only marked **Collected** after its own OTP is verified, even inside a group.

## 🏗 Architecture

```
 Gate staff app    Resident app    Admin dashboard
        │                │                │
        └────────── HTTPS + JWT ──────────┘
                         │
              ┌──────────▼──────────┐
              │   REST API server   │
              │ Auth · Orders/OTP · │
              │ Pickup groups · Users│
              └───────┬───────┬─────┘
                      │       │
         ┌────────────▼┐     ┌▼───────────────────┐
         │  Database   │     │ Notification service│
         └─────────────┘     │ push / SMS / email  │
                             └─────────────────────┘
```

Clients never touch the database. The API checks the caller’s role, writes the change, then asks the notification service to alert the resident.

## 🛠 Tech Stack

> Update this section to match what you actually used.
> 

| Layer | Technology |
| --- | --- |
| Frontend | *e.g. React / mobile-first web app* |
| Backend | *e.g. Node.js + Express* |
| Database | *e.g. PostgreSQL / MongoDB* |
| Auth | JWT, bcrypt-hashed passwords |
| Notifications | *e.g. Firebase Cloud Messaging / SMS gateway* |

## 🚀 Getting Started

> Commands below are a template. Adjust them to your project.
> 

### Prerequisites

- Node.js 18+ (or your runtime)
- A running database instance

### Installation

```bash
# clone the repository
git clone https://github.com/<your-username>/main-gate-tracker.git
cd main-gate-tracker

# install dependencies
npm install

# set up environment variables
cp .env.example .env

# start the development server
npm run dev
```

### Environment variables

```
PORT=5000
DATABASE_URL=your_database_url
JWT_SECRET=your_jwt_secret
OTP_EXPIRY_MINUTES=10
```

## 📡 API Reference

All endpoints are under `/api`. Everything except register and login needs a valid token (`Authorization: Bearer <jwt>`).

| Method | Endpoint | Purpose | Access |
| --- | --- | --- | --- |
| POST | `/auth/register` | Create a new account | Public |
| POST | `/auth/login` | Log in and receive a JWT | Public |
| GET | `/users` | List all users | Staff, Admin |
| GET | `/users/:id` | Get one user | Staff, Admin, or the user |
| PUT | `/users/:id` | Update a user | Admin, or the user |
| DELETE | `/users/:id` | Deactivate a user | Admin |
| POST | `/orders` | Log a delivery and generate OTP | Staff |
| GET | `/orders` | List and filter orders | Staff, Admin |
| GET | `/orders/:id` | View one order | Staff, Admin, or the resident |
| POST | `/orders/:id/verify` | Verify OTP and mark collected | Staff |
| POST | `/orders/:id/resend-otp` | Regenerate and resend OTP | Staff |
| POST | `/pickup-groups` | Create a collective pickup | Staff |
| GET | `/pickup-groups/:id` | View group and per-order status | Staff, Admin, or a resident in the group |
| GET | `/users/:id/orders` | A user’s order history | Staff, Admin, or the user |

### Conventions

- **Base URL:** `http://localhost:5000/api`
- **Format:** JSON in and out, so send `Content-Type: application/json`
- **Auth:** protected routes need `Authorization: Bearer <jwt>`
- **Passwords and OTPs:** never returned in any response
- **Errors:** every error uses the same shape

```json
{ "error": { "code": "EMAIL_EXISTS", "message": "An account with this email already exists" } }
```

| Status | Meaning |
| --- | --- |
| 200 / 201 | Success / created |
| 400 | Invalid or missing fields |
| 401 | Missing, expired or invalid token, or wrong login |
| 403 | Role not allowed for this action |
| 404 | User not found |
| 409 | Email already registered |
| 429 | Too many login attempts |

---

### 1. Register

`POST /api/auth/register` · **Access:** public

Creates a new account. Residents are created by default; `staff` and `admin` roles can only be assigned by an admin.

**Request body**

| Field | Type | Required | Rules |
| --- | --- | --- | --- |
| `name` | string | Yes | 2 to 60 characters |
| `email` | string | Yes | Valid email, unique |
| `password` | string | Yes | At least 8 characters, stored hashed |
| `role` | string | No | `resident` (default), `staff`, `admin` |
| `hostel` | string | Residents | Required when role is `resident` |
| `room_no` | string | Residents | Required when role is `resident` |
| `phone` | string | No | 10 digits |

```bash
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Student Name",
    "email": "student@example.com",
    "password": "StrongPass123",
    "role": "resident",
    "hostel": "Block A",
    "room_no": "214",
    "phone": "9XXXXXXXXX"
  }'
```

**Response `201 Created`**

```json
{
  "user": {
    "id": "u_101",
    "name": "Student Name",
    "email": "student@example.com",
    "role": "resident",
    "hostel": "Block A",
    "room_no": "214",
    "phone": "9XXXXXXXXX",
    "created_at": "2026-11-20T09:30:00Z"
  }
}
```

**Errors:** `400` validation failed · `409` email already registered · `403` role other than `resident` requested without an admin token

---

### 2. Login

`POST /api/auth/login` · **Access:** public

Checks the credentials and returns a JWT to use on every later request.

**Request body**

| Field | Type | Required |
| --- | --- | --- |
| `email` | string | Yes |
| `password` | string | Yes |

```bash
curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{ "email": "student@example.com", "password": "StrongPass123" }'
```

**Response `200 OK`**

```json
{
  "token": "<jwt>",
  "expiresIn": 3600,
  "user": { "id": "u_101", "name": "Student Name", "role": "resident" }
}
```

**Errors:** `400` missing fields · `401` wrong email or password (same message for both, so accounts cannot be guessed) · `403` account deactivated · `429` too many failed attempts, try again later

---

### 3. Get all users

`GET /api/users` · **Access:** staff, admin

Returns a paginated list of users. Filters are optional and can be combined.

**Query parameters**

| Param | Type | Default | Purpose |
| --- | --- | --- | --- |
| `role` | string | all | Filter by `resident`, `staff` or `admin` |
| `hostel` | string | all | Filter by hostel |
| `search` | string | none | Match on name, email or room number |
| `page` | integer | 1 | Page number |
| `limit` | integer | 20 | Page size, maximum 100 |

```bash
curl "http://localhost:5000/api/users?role=resident&hostel=Block%20A&page=1&limit=20" \
  -H "Authorization: Bearer <jwt>"
```

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": "u_101",
      "name": "Student Name",
      "email": "student@example.com",
      "role": "resident",
      "hostel": "Block A",
      "room_no": "214",
      "phone": "9XXXXXXXXX"
    }
  ],
  "page": 1,
  "limit": 20,
  "total": 1
}
```

**Errors:** `401` missing or invalid token · `403` caller is a resident

---

### 4. Get user by id

`GET /api/users/:id` · **Access:** staff, admin, or the user themself

Returns one user’s profile.

```bash
curl http://localhost:5000/api/users/u_101 \
  -H "Authorization: Bearer <jwt>"
```

**Response `200 OK`**

```json
{
  "user": {
    "id": "u_101",
    "name": "Student Name",
    "email": "student@example.com",
    "role": "resident",
    "hostel": "Block A",
    "room_no": "214",
    "phone": "9XXXXXXXXX",
    "created_at": "2026-11-20T09:30:00Z"
  }
}
```

**Errors:** `401` missing or invalid token · `403` a resident asked for another user’s id · `404` no user with that id

---

## 🗄 Data Model

| Table | Key fields |
| --- | --- |
| `users` | id, name, email, password_hash, role, hostel, room_no, phone, active |
| `orders` | id, reference_no, courier, description, item_count, status, arrival_time, collected_time, otp_hash, otp_expiry, resident_id, handled_by, pickup_group_id |
| `pickup_groups` | id, status, created_at, created_by |
| `notifications` | id, order_id, channel, status, attempts |
| `audit_log` | id, order_id, action, actor_id, timestamp |

## 🔐 Security

- OTPs are stored **hashed**, are single-use and expire
- Attempt limits lock an order after repeated wrong codes
- Passwords are hashed; sessions use short-lived JWTs over HTTPS
- Role checks on every route; residents only see their own orders
- Append-only audit log of create, notify, verify and collect

## 🗺 Roadmap

- [ ]  QR code in place of manual OTP entry
- [ ]  Photo capture of parcels at arrival
- [ ]  Reminders for unclaimed orders
- [ ]  Analytics on peak hours and pending parcels
- [ ]  Multi-gate and multi-hostel support
- [ ]  Integration with hostel ID cards

## 🤝 Contributing

Contributions are welcome.

1. Fork the repository
2. Create a branch: `git checkout -b feature/your-feature`
3. Commit your changes: `git commit -m "Add your feature"`
4. Push the branch: `git push origin feature/your-feature`
5. Open a Pull Request

## 📄 License

Distributed under the MIT License. See `LICENSE` for details.
