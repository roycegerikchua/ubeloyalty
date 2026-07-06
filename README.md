# UbeLoyalty — Eng Bee Tin Loyalty Rewards System

A Node-RED-based loyalty rewards system powering the **Ube Card** program for Eng Bee Tin.
Manages member registration, point earning/redeeming, push notifications via Firebase Cloud Messaging (FCM), and a cashier-facing terminal.

---

## 🧩 Architecture

```
┌──────────────────────┐     ┌──────────────────────┐
│   Member PWA (/)     │     │  Cashier Terminal    │
│  (mobile-first web)  │     │   (/cashier)         │
└────────┬─────────────┘     └────────┬──────────────┘
         │                            │
         ▼                            ▼
┌──────────────────────────────────────────────┐
│         Node-RED (VIPLoyaltyRed)             │
│         192.168.111.112:32778                │
│                                              │
│  ┌─ EBT Ube Loyalty API ──────────────────┐ │
│  │  Member CRUD, earn/redeem, OTP, auth   │ │
│  ├─ UI ───────────────────────────────────┤ │
│  │  Member-facing PWA, cashier terminal   │ │
│  ├─ FCM Device Registration ──────────────┤ │
│  │  Push notification token management    │ │
│  ├─ FCM Birthday ─────────────────────────┤ │
│  │  Auto-grant 50pts on birthday          │ │
│  └─ FCMBroadcaster ───────────────────────┤ │
│     Test/broadcast admin tools            │ │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│         MSSQL (192.168.27.27)                │
│  ┌─ UbeLoyalty ────────────────────────────┐ │
│  │  members, transactions, tiers, devices  │ │
│  ├─ VIP_HO ────────────────────────────────┤ │
│  │  Staff, passwords, groups (CTA cipher)  │ │
│  └─ PosConfig ─────────────────────────────┤ │
│     Branch config and database mapping     │ │
└──────────────────────────────────────────────┘
```

---

## 📋 Flow Tabs

| Tab | Nodes | Description |
|-----|-------|-------------|
| **EBT Ube Loyalty API** | 110 | Core REST API — member CRUD, OTP, earn/redeem points, staff auth |
| **UI** | 51 | Member-facing PWA (`/`) + Cashier terminal (`/cashier`) |
| **FCM Device Registration** | 24 | Push notification token registration/unregistration |
| **FCM Birthday (per-device)** | 10 | Auto-grant 50 birthday points via cron |
| **FCMBroadcaster** | 45 | Admin broadcast and test tools |

---

## 🌐 API Endpoints

### Authentication
| Method | Route | Description |
|--------|-------|-------------|
| `POST` | `/login` | Staff login — authenticates against `VIP_HO.dbo.Staff` (CTA cipher password) |
| `POST` | `/loyalty/cashier/login` | Log cashier session to `loyalty_cashier_sessions` |
| `POST` | `/loyalty/cashier/logout` | End cashier session |

### Member Management
| Method | Route | Description |
|--------|-------|-------------|
| `POST` | `/loyalty/otp/send` | Send OTP via SMS (Movider) or email |
| `POST` | `/loyalty/otp/verify` | Verify OTP & register member |
| `POST` | `/loyalty/register` | Register new member (generates `EBT-XXXXX` ID) |
| `GET` | `/loyalty/member/:id` | Fetch member profile (points, tier, metadata) |
| `PATCH` | `/loyalty/member/:id/metadata` | Update JSON metadata |
| `PATCH` | `/loyalty/member/:id/contact` | Update contact info |

### Points
| Method | Route | Description |
|--------|-------|-------------|
| `POST` | `/loyalty/earn` | Earn points (₱100 = 1pt) — triggers tier upgrade check |
| `POST` | `/loyalty/redeem` | Redeem points (1pt = ₱1) |
| `GET` | `/loyalty/transactions/:id` | Paginated transaction history |

### Push Notifications (FCM)
| Method | Route | Description |
|--------|-------|-------------|
| `POST` | `/loyalty/device/register` | Register device for push (accepts `token` or `fcm_token`) |
| `POST` | `/loyalty/device/unregister` | Remove device from push notifications |
| `POST` | `/loyalty/device/notify` | Send notification to a member's devices |
| `GET` | `/api/devices/:id` | List registered devices with hint and last seen |

### Supporting
| Method | Route | Description |
|--------|-------|-------------|
| `GET` | `/api/branches` | List branches from `PosConfig` |
| `GET` | `/api/rewards` | Rewards catalog |
| `GET` | `/` | Member PWA |
| `GET` | `/cashier` | Cashier terminal |

---

## 🗄️ Database (UbeLoyalty)

### Tables

| Table | Purpose |
|-------|---------|
| `loyalty_members` | Member profiles — name, mobile, email, tier, points, metadata |
| `loyalty_transactions` | Earn/redeem records — linked to member + branch |
| `loyalty_tiers` | Tier definitions — Silver (0pts), Gold (2k), Platinum (5k) |
| `loyalty_cashiers` | Cashier accounts with password hashes |
| `loyalty_cashier_sessions` | Cashier login audit log |
| `loyalty_otp` | One-time passwords for member verification |
| `loyalty_member_devices` | FCM push tokens (key notification table) |

### Key Columns

```sql
-- FCM device tokens
loyalty_member_devices.fcm_token   -- Firebase Cloud Messaging token
loyalty_member_devices.is_active   -- BIT: 1=active, 0=deactivated

-- Always filter by is_active = 1 when querying devices
```

---

## 🔔 Push Notifications

Notifications use **Firebase Cloud Messaging v1 API** with the `ebt-ubecard` project.

**Flow:**
1. Earn/redeem transaction completes
2. `Build earn response` function sends `msgFCM` to `toFCM` link call
3. `Notify FCM` function sets title/body and triggers token fetch
4. `Validate & fetch tokens SQL` queries `loyalty_member_devices` for active tokens
5. `Fan out one FCM msg per token` sends each token to Firebase FCM v1 API

**Settings page:**
Members can register/unregister devices under **Profile → Settings → Push Notifications**.

---

## 🔐 Authentication

- **Staff login:** Uses `VIP_HO.dbo.Staff` with CTA cipher password decryption
- **Cashier session:** Logged to `loyalty_cashier_sessions` after staff authentication
- **Member login:** Via OTP sent to mobile/email

---

## 📦 Dependencies

- Node-RED v4.1.11+
- MSSQL database server
- Firebase project (`ebt-ubecard`) with service account JSON at `/data/firebase-service-account.json`
- (Optional) Movider SMS API for OTP delivery
- (Optional) AWS SES for email notifications
