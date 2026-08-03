# High-Level Design (HLD)
## QuickShow — Online Movie Ticket Booking Platform

**Version:** 1.0 (reverse-engineered from codebase)

---

## 1. Architecture Style

QuickShow is a **client-server web application** with an **event-driven
background-processing layer**:

- **Frontend:** Single-page application (React 19 + Vite + Tailwind CSS 4 +
  React Router 7), deployed on Vercel.
- **Backend:** REST API (Node.js + Express 5), deployed on Vercel
  (serverless functions).
- **Database:** MongoDB (Mongoose ODM), single database `quickshow`.
- **Auth provider:** Clerk (hosted auth, session/JWT verification via
  `@clerk/express` middleware and `@clerk/clerk-react` on the frontend).
- **Payments:** Stripe Checkout (hosted payment page) + Stripe webhooks.
- **Background jobs / event bus:** Inngest — durable, event-driven
  functions with built-in retries, scheduled sleeps, and cron triggers.
- **Email:** Nodemailer over Brevo SMTP relay.
- **External data source:** TMDB (The Movie Database) public API, for
  movie metadata and cast.
- **Media (declared dependency):** Cloudinary (available for image
  handling; not exercised by the flows reviewed here beyond being a
  server dependency).

---

## 2. System Context Diagram

```
                     ┌──────────────────────────┐
                     │        End User          │
                     │   (browser, customer)     │
                     └────────────┬──────────────┘
                                  │ HTTPS
                                  ▼
                     ┌──────────────────────────┐
                     │   React SPA (Vite)        │
                     │   client/ (Vercel)         │
                     └────────────┬──────────────┘
                                  │ REST (Axios) + Clerk JWT
                                  ▼
                     ┌──────────────────────────┐
        TMDB API◄────┤   Express API server      ├────► Clerk (Auth API)
                     │   server/ (Vercel)          │
        Brevo SMTP◄──┤                            ├────► Stripe (Checkout + Webhooks)
                     └────────────┬───────────────┘
                                  │
                     ┌────────────┴───────────────┐
                     │        MongoDB             │
                     │  (Users, Movies, Shows,     │
                     │   Bookings)                 │
                     └────────────┬───────────────┘
                                  │
                     ┌────────────┴───────────────┐
                     │   Inngest (event bus /      │
                     │   durable functions,        │
                     │   cron scheduler)           │
                     └────────────────────────────┘
```

---

## 3. Major Components

### 3.1 Client (React SPA)
- **Public pages:** Home, Movies list, Movie Details, Seat Layout, My
  Bookings, Favorites.
- **Admin pages** (`/admin/*`, gated by Clerk sign-in + `isAdmin` check):
  Dashboard, Add Shows, List Shows, List Bookings.
- **AppContext** — global state: current user (from Clerk), auth token
  getter, `isAdmin` flag, cached list of shows, favorite movies. Fetches
  `is-admin` and `favorites` whenever a Clerk user session is present.
- Axios base URL points at the deployed API (`VITE_BASE_URL`).

### 3.2 API Server (Express)
Organized by route domain, each backed by a controller:

| Route prefix | Controller | Responsibility |
|---|---|---|
| `/api/show` | `showController` | TMDB lookups, show creation, listing shows, per-movie showtimes |
| `/api/booking` | `bookingController` | Seat availability check, booking + Stripe session creation, occupied-seats lookup |
| `/api/admin` | `adminController` | Admin-only: dashboard metrics, all shows, all bookings, is-admin check |
| `/api/user` | `userController` | User's bookings, favorites (stored in Clerk metadata) |
| `/api/stripe` | `stripeWebhooks` | Stripe webhook receiver (raw body, signature verified) |
| `/api/inngest` | Inngest handler | Serves Inngest functions for the Inngest dev/cloud runner to invoke |

Global middleware: `express.json()`, `cors()`, `clerkMiddleware()` (attaches
`req.auth()` used across controllers). The Stripe webhook route is mounted
**before** `express.json()` so it can consume the raw body required for
signature verification.

### 3.3 Data Layer (MongoDB via Mongoose)
Four collections: `User`, `Movie`, `Show`, `Booking` (see LLD for schemas).
Notable design choice: `Movie._id` and `User._id` are **strings** — they
reuse the external ID (TMDB movie id, Clerk user id) as the primary key,
avoiding a separate mapping table.

### 3.4 Background Processing (Inngest)
Seven durable functions, triggered by either Clerk webhooks, internal app
events, or a cron schedule:

| Function | Trigger | Purpose |
|---|---|---|
| `sync-user-from-clerk` | `clerk/user.created` | Create local User on signup |
| `update-user-from-clerk` | `clerk/user.updated` | Keep local User in sync |
| `delete-user-with-clerk` | `clerk/user.deleted` | Remove local User on account deletion |
| `release-seats-delete-booking` | `app/checkpayment` (sent at booking time) | Sleep until +10 min, then release seats & delete booking if still unpaid |
| `send-booking-confirmation-email` | `app/show.booked` (sent by Stripe webhook) | Email the user their confirmed booking |
| `send-show-reminders` | cron `0 */8 * * *` | Email users with shows starting in ~8 hours |
| `send-new-show-notifications` | `app/show.added` (sent by admin add-show) | Email all users about a newly added show |

Using Inngest rather than `setTimeout`/naive cron in-process makes the
10-minute seat-hold expiry and the 8-hour reminder job durable across
serverless cold starts/restarts.

### 3.5 External Integrations
- **Clerk** — identity provider; also used as a lightweight key-value store
  for per-user data that doesn't need relational querying (favorites,
  admin role) via `privateMetadata`.
- **TMDB** — source of truth for movie catalog data; QuickShow caches a
  denormalized snapshot into `Movie` the first time a movie is scheduled.
- **Stripe** — Checkout Sessions for payment; webhook confirms payment
  asynchronously (decoupled from the booking-creation request).
- **Brevo (via Nodemailer/SMTP)** — transactional email delivery.

---

## 4. Key Design Decisions & Trade-offs

1. **Seats held optimistically at booking-creation, not at payment
   confirmation.** This simplifies the seat map (one source of truth: `Show.occupiedSeats`)
   but requires the 10-minute auto-release job to avoid seats being stuck
   "held" forever by abandoned checkouts.
2. **Movie catalog is lazily cached from TMDB.** Reduces up-front data
   entry, but means the first admin to schedule a given movie incurs the
   TMDB round trip; subsequent shows for that movie reuse the cached
   `Movie` document.
3. **Favorites live in Clerk metadata, not MongoDB.** Avoids a join table
   and keeps user-profile-adjacent data next to the identity provider, at
   the cost of an extra Clerk API call per favorites read/write and a
   dependency on Clerk's metadata size limits.
4. **Stripe confirmation is webhook-driven, not response-driven.** The
   booking API responds with a redirect URL immediately; the authoritative
   "paid" state only flips once Stripe calls back, which is more resilient
   to users closing the browser mid-payment.
5. **Admin authorization is a metadata flag, not a separate role table.**
   Simple to implement, but not extensible to multiple permission tiers
   without a broader refactor.
6. **Serverless deployment (Vercel) for both tiers.** Client and server
   scale independently; the trade-off is that long-lived in-memory state
   (e.g., a naive setInterval-based scheduler) would not survive across
   invocations — which is precisely why Inngest is used for anything
   time-delayed.

---

## 5. Cross-Cutting Concerns

- **AuthN/AuthZ:** Clerk-issued JWT verified on every request via
  `clerkMiddleware()`; `protectAdmin` middleware additionally checks
  `privateMetadata.role`.
- **Idempotency / race conditions:** Seat availability is re-checked
  server-side at booking time (not trusted from client state) to reduce
  double-booking risk, though true atomicity depends on the underlying
  `findById` + in-memory mutate + `save()` pattern (see LLD § "Known
  Concurrency Risk").
- **Error handling:** Controllers uniformly return `{ success: false,
  message }` JSON on failure rather than relying solely on HTTP status
  codes.
- **Observability:** Console logging only; no external APM/logging service
  wired in.

---

## 6. Deployment View

- `client/vercel.json` and `server/vercel.json` indicate independent
  Vercel projects/deployments for frontend and backend.
- Environment configuration via `.env` files: MongoDB URI, Clerk keys,
  Stripe keys/webhook secret, TMDB API key, SMTP credentials, Cloudinary
  credentials, and the frontend's API base URL / TMDB image base URL.
