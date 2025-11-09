# 🧩 TaskNShop Technical Blueprint

A concise technical plan covering frontend, backend, and database architecture; communication flows; feasibility.

---

## 1️⃣ Recommended Stack (MVP-First)

### **Frontend (Mobile)**
- **Framework:** React Native (Expo) – single codebase for iOS & Android, rapid iteration.  
- **Real-time:** Socket.IO client *(or Firebase SDK as alternative for faster setup)*.  
- **Maps & Geolocation:** Google Maps SDK / Mapbox SDK.  
- **Storage Uploads:** Pre-signed S3 uploads from the app.  
- **Push Notifications:** Firebase Cloud Messaging (FCM) & Apple Push Notification Service (APNS).

### **Backend**
- **Language / Framework:** Node.js with Express or NestJS (modular, battle-tested).  
- **Real-time Gateway:** Socket.IO server (hosted with the API) or Firebase Realtime/Firestore.  
- **Worker & Queue:** BullMQ (Redis-backed) for matching, retries, and background jobs.  
- **File Proxy / Signed URLs:** Backend generates pre-signed S3 URLs.  
- **Auth & Security:** JWT access + refresh tokens; OAuth/social logins optional.

### **Database & Cache**
- **Primary DB:** PostgreSQL (relational consistency for orders, users, ratings).  
- **Cache / Fast Lookups:** Redis (online shopper presence, geohash indexes, rate limiting).  
- **Search / Location Indexing:** PostGIS for Postgres or Redis Geo for lightweight proximity queries.

### **Storage & 3rd-Party**
- **Object Storage:** AWS S3 (or DigitalOcean Spaces) for photos & receipts.  
- **Payments:** Local providers (Paystack, Flutterwave) + Stripe fallback.  
- **Maps & Routing:** Google Maps Platform or Mapbox.  
- **Hosting / Infra (MVP):** Render / Railway / Fly / Heroku for fast deployment; AWS/GCP for scale.

---

## 2️⃣ Component Communication (Detailed Sequences)

### **Core Communication Channels**
- **HTTPS REST:** Standard CRUD, authentication, order creation, account management.  
- **WebSockets (Socket.IO):** Real-time chat, shopper status, live updates.  
- **Push Notifications (FCM/APNS):** Notify offline users/shoppers about assignments or urgent events.  
- **Background Queue (Redis + BullMQ):** Matching jobs, notifications, payment reconciliation.  
- **Webhooks:** Payment provider callbacks and third-party events.

---

### **Sequence: Create Order → Match → Purchase → Deliver (Happy Path)**

1. **Order Creation:**  
   App sends `POST /orders` with JWT + item data (photo ref or uploaded image).  
   Backend stores order in Postgres (`status: searching`).

2. **Enqueue Match Job:**  
   Backend pushes a job to BullMQ to find available shoppers.

3. **Match Worker:**  
   Queries Redis for nearby shoppers (geo constraints), validates verification, assigns one.

4. **Notify Shopper:**  
   Worker sends push notification + Socket.IO event to shopper.

5. **Shopper Accepts:**  
   Shopper emits `accept_order` → Backend validates and transitions order to `in_progress`.

6. **Live Shopping Updates:**  
   Shopper uploads photos via pre-signed S3 URL → emits `photo_uploaded` → Backend updates buyer via Socket.IO.

7. **Payment:**  
   Buyer pays via in-app provider → webhook confirms → backend updates DB escrow/payment state.

8. **Completion & Rating:**  
   After delivery, order → `completed`; both rate each other; funds released per escrow rules.

---

### **Failure / Retries / Dispute Flows**
- If no shopper accepts → fallback logic (expand radius, retry).  
- Webhook signature verification prevents tampering.  
- Disputes trigger admin workflow; background jobs freeze funds until resolved.

---

## 3️⃣ Data Model (High-Level)

| Table | Key Fields |
|-------|-------------|
| **Users** | id, name, phone, email, role (buyer/shopper/admin), KYC status, rating |
| **Shoppers** | user_id, verified_at, service_areas (geo polygons), active_status, accepted_orders_count |
| **Orders** | id, buyer_id, shopper_id, items (json), photos (urls), status, price_subtotal, fees, escrow_state |
| **Payments** | id, order_id, provider, reference, status, amount, timestamp |
| **Ratings** | id, order_id, rater_id, ratee_id, score, comment |

---

## 4️ Security & Trust Mechanisms
- **KYC:** Shopper ID upload + manual/automated verification.  
- **Escrow Payments:** Buyer funds held until delivery confirmation.  
- **Transport Security:** HTTPS, HSTS, TLS 1.2+.  
- **API Protection:** Rate limiting, auth checks, role-based access control.  
- **Data Protection:** Encrypt sensitive fields; rotate secrets via secrets manager.

---

## 5️⃣Why This Approach Is Technically Feasible
-  **Mature Tools:** React Native, Node.js, Postgres, Redis, S3, Socket.IO are stable and well-supported.  
- **Incremental Real-Time:** Start with polling + push notifications before adding Socket.IO.  
- **Third-Party Integrations:** Payment and mapping SDKs reduce complexity.  
- **Single-City Launch:** Simplifies matching logic and logistics for MVP.  
- **Scalable Path:** Stateless API servers, Redis shared state, Postgres replicas for scaling.

---

## 6️⃣ Steps to Complete (Ordered Checklist)

### **Phase 0: Discovery & Planning**
- Select pilot city; research shopper density & payment methods.  
- Define fees, incentives, trust & safety policies.

### **Phase 1: Scaffolding (1–2 weeks)**
- Initialize monorepo: `/mobile`, `/api`, `/workers`, `/admin`.  
- Setup CI/CD, linters, README.  
- Provision dev infra: Postgres, Redis, S3 dev bucket.

### **Phase 2: Core Backend & DB (2–3 weeks)**
- Build JWT auth, user & shopper models, basic order schema.  
- Integrate payment sandbox + webhook handler.  
- Setup BullMQ worker & matching job skeleton.

### **Phase 3: Mobile MVP (2–4 weeks)**
- Build order creation UI + photo upload (pre-signed URL).  
- Shopper UI: accept orders, send updates.  
- Implement polling or simple WebSocket for live updates.

### **Phase 4: QA & Pilot (2–4 weeks)**
- Internal testing (buyers & shoppers).  
- Fix matching reliability, payment flow, upload issues.  
- Launch closed pilot in target city.

### **Phase 5: Iterate & Harden**
- Add KYC, escrow, admin dashboard.  
- Add monitoring (Sentry, logs, alerts).  
- Optimize matching algorithm (rating, distance, acceptance rate).

### **Phase 6: Scale**
- Deploy to scalable cloud infra (auto-scaling, managed DB).  
- Add analytics, A/B testing, multi-region support, new payment providers.

---

## 7️⃣ Implementation Notes & Trade-Offs

| Decision | Trade-Off |
|-----------|------------|
| **Socket.IO vs Firebase** | Socket.IO = more control, cheaper; Firebase = faster dev, vendor lock-in. |
| **Postgres + PostGIS vs Redis Geo** | PostGIS = complex spatial accuracy; Redis Geo = fast proximity search. |
| **Escrow Logic** | Full escrow adds complexity; start with conditional payouts for MVP. |

---

** Summary:**  
This blueprint enables a lean, scalable MVP for TaskNShop — balancing fast delivery with solid real-time capabilities, secure payments, and a clear upgrade path toward production-level scale.
