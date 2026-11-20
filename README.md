# 🚪 GateKeep

**A hostel gate management app for receiving, securing and tracking deliveries.**

Gate staff log every incoming order, the app issues a one-time password (OTP) for collection and notifies the resident, and every handover is time-stamped, so parcels stop going missing or reaching the wrong hands.

---

## 📌 Table of Contents

- [The Problem](#-the-problem)
- [Features](#-features)
- [How It Works](#-how-it-works)
- [Order Lifecycle](#-order-lifecycle)
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Getting Started](#-getting-started)
- [API Reference](#-api-reference)
- [Data Model](#-data-model)
- [Security](#-security)
- [Roadmap](#-roadmap)
- [Contributing](#-contributing)
- [Author](#-author)

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

Clients never touch the database. The API checks the caller's role, writes the change, then asks the notification service to alert the resident.

## 🛠 Tech Stack

> Update this section to match what you actually used.

| Layer | Technology |
|---|---|
| Frontend | _e.g. React / mobile-first web app_ |
| Backend | _e.g. Node.js + Express_ |
| Database | _e.g. PostgreSQL / MongoDB_ |
| Auth | JWT, bcrypt-hashed passwords |
| Notifications | _e.g. Firebase Cloud Messaging / SMS gateway_ |

## 🚀 Getting Started

> Commands below are a template. Adjust them to your project.

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

```env
PORT=5000
DATABASE_URL=your_database_url
JWT_SECRET=your_jwt_secret
OTP_EXPIRY_MINUTES=10
```

## 📡 API Reference

All endpoints are under `/api`. Everything except register and login needs a valid token (`Authorization: Bearer <jwt>`).

| Method | Endpoint | Purpose | Access |
|---|---|---|---|
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
| GET | `/users/:id/orders` | A user's order history | Staff, Admin, or the user |

### Example: register

```http
POST /api/auth/register
```

```json
{
  "name": "Student Name",
  "email": "student@example.com",
  "password": "********",
  "role": "resident",
  "hostel": "Block A",
  "room_no": "214",
  "phone": "9XXXXXXXXX"
}
```

### Example: login

```http
POST /api/auth/login
```

```json
{ "email": "student@example.com", "password": "********" }
```

```json
{ "token": "<jwt>", "user": { "id": "u_101", "name": "Student Name", "role": "resident" } }
```

## 🗄 Data Model

| Table | Key fields |
|---|---|
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

- [ ] QR code in place of manual OTP entry
- [ ] Photo capture of parcels at arrival
- [ ] Reminders for unclaimed orders
- [ ] Analytics on peak hours and pending parcels
- [ ] Multi-gate and multi-hostel support
- [ ] Integration with hostel ID cards

## 🤝 Contributing

Contributions are welcome.

1. Fork the repository
2. Create a branch: `git checkout -b feature/your-feature`
3. Commit your changes: `git commit -m "Add your feature"`
4. Push the branch: `git push origin feature/your-feature`
5. Open a Pull Request

## 👤 Author

**Vivek Yadav** ([@Naivete07](https://github.com/Naivete07))

## 📄 License

Distributed under the MIT License. See `LICENSE` for details.
