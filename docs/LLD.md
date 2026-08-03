# Low-Level Design (LLD)
## QuickShow — Online Movie Ticket Booking Platform

**Version:** 1.0 (reverse-engineered from codebase)

---

## 1. Database Schemas (MongoDB / Mongoose)

### 1.1 `User`
```js
{
  _id: String,        // Clerk user id (reused as primary key)
  name: String,        // required
  email: String,       // required
  image: String        // required, profile picture URL
}
```
No `timestamps`. Populated/kept in sync exclusively by the Inngest
functions listening to Clerk webhooks — never written directly by
user-facing controllers.

### 1.2 `Movie`
```js
{
  _id: String,               // TMDB movie id (reused as primary key)
  title: String,              // required
  overview: String,           // required
  poster_path: String,        // required
  backdrop_path: String,      // required
  release_date: String,       // required
  original_language: String,
  tagline: String,
  genres: Array,               // required, TMDB genre objects
  casts: Array,                 // required, TMDB credits.cast
  vote_average: Number,        // required
  runtime: Number,             // required (minutes)
  createdAt / updatedAt         // timestamps: true
}
```

### 1.3 `Show`
```js
{
  _id: ObjectId,
  movie: String,                 // ref 'Movie', required (TMDB id)
  showDateTime: Date,             // required
  showPrice: Number,              // required
  occupiedSeats: Object            // default {}, e.g. { "A1": "<userId>", "C4": "<userId>" }
}
// schema option: { minimize: false }  -> keeps occupiedSeats: {} even when empty
```
`occupiedSeats` is a map keyed by seat label (e.g. `"A1"`) → the booking
user's Clerk id. This doubles as (a) the seat-availability source of truth
and (b) the lookup used by the reminder job to find who to email.

### 1.4 `Booking`
```js
{
  _id: ObjectId,
  user: String,           // ref 'User', required (Clerk user id)
  show: String,             // ref 'Show', required (ObjectId as string)
  amount: Number,           // required, showPrice * seats.length
  bookedSeats: Array,        // required, e.g. ["A1", "A2"]
  isPaid: Boolean,           // default false
  paymentLink: String,       // Stripe Checkout Session URL; cleared on payment
  createdAt / updatedAt       // timestamps: true
}
```

**Entity relationship:**
```
User (1) ──< Booking (N) >── (1) Show (N) >── (1) Movie
```

---

## 2. API Contract

Base path: `/api`. All routes return JSON of shape
`{ success: boolean, ...payload }` or `{ success: false, message }`.

### 2.1 `showRoutes` (`/api/show`)
| Method | Path | Auth | Handler | Description |
|---|---|---|---|---|
| GET | `/now-playing` | admin | `getNowPlayingMovies` | Proxies TMDB `movie/now_playing` |
| POST | `/add` | admin | `addShow` | Body: `{ movieId, showsInput: [{date, time:[]}], showPrice }`. Creates `Movie` (if new, via TMDB) + one `Show` per date/time. Fires `app/show.added`. |
| GET | `/all` | public | `getShows` | Upcoming shows, deduped by movie, populated with movie data |
| GET | `/:movieId` | public | `getShow` | Movie details + upcoming showtimes grouped by date: `{ movie, dateTime: { "YYYY-MM-DD": [{time, showId}] } }` |

### 2.2 `bookingRoutes` (`/api/booking`)
| Method | Path | Auth | Handler | Description |
|---|---|---|---|---|
| POST | `/create` | signed-in user | `createBooking` | Body: `{ showId, selectedSeats }`. Returns `{ success, url }` (Stripe Checkout URL) |
| GET | `/seats/:showId` | public | `getOccupiedSeats` | Returns `{ success, occupiedSeats: [seatId,...] }` |

### 2.3 `adminRoutes` (`/api/admin`) — all behind `protectAdmin`
| Method | Path | Handler | Description |
|---|---|---|---|
| GET | `/is-admin` | `isAdmin` | Returns `{ success:true, isAdmin:true }` (reachable only if middleware passed) |
| GET | `/dashboard` | `getDashboardData` | `{ totalBookings, totalRevenue, activeShows, totalUser }` |
| GET | `/all-shows` | `getAllShows` | All upcoming shows, populated with movie, sorted by date |
| GET | `/all-bookings` | `getAllBookings` | All bookings, populated with user + show.movie, newest first |

### 2.4 `userRoutes` (`/api/user`)
| Method | Path | Handler | Description |
|---|---|---|---|
| GET | `/bookings` | `getUserBookings` | Current user's bookings, populated with show + movie |
| POST | `/update-favorite` | `updateFavorite` | Body: `{ movieId }`; toggles in `privateMetadata.favorites` |
| GET | `/favorites` | `getFavorites` | Movies matching the user's favorite ids |

> Note: `userRoutes` are mounted without an explicit auth middleware in
> the router itself; `req.auth().userId` is read directly inside each
> controller, relying on `clerkMiddleware()` mounted globally in
> `server.js`. There is no explicit "require signed-in" guard beyond that —
> an unauthenticated request will fail when `req.auth()` has no `userId`.

### 2.5 Webhooks
| Method | Path | Description |
|---|---|---|
| POST | `/api/stripe` | Verifies `stripe-signature` header against `STRIPE_WEBHOOK_SECRET`. On `payment_intent.succeeded`, looks up the Checkout Session by payment intent, reads `metadata.bookingId`, sets `Booking.isPaid = true`, clears `paymentLink`, fires `app/show.booked`. |
| ALL | `/api/inngest` | Served by `inngest/express`'s `serve()` — the endpoint Inngest's runner calls to execute registered functions. |

---

## 3. Core Sequence Flows

### 3.1 Booking + Payment (happy path)
```
User (SeatLayout page)
  │ 1. GET /api/booking/seats/:showId
  ▼
Server → Show.findById → return occupiedSeats keys
  │
User selects ≤5 free seats
  │ 2. POST /api/booking/create { showId, selectedSeats }  [Bearer Clerk JWT]
  ▼
Server:
  a. checkSeatsAvailability(showId, selectedSeats)
     - Show.findById(showId)
     - none of selectedSeats present in occupiedSeats  → proceed
  b. Show.findById(showId).populate('movie')             (for price + title)
  c. Booking.create({ user, show, amount, bookedSeats })
  d. mutate showData.occupiedSeats[seat] = userId for each seat
     → markModified('occupiedSeats') → showData.save()
  e. stripe.checkout.sessions.create({
       line_items: [{ price_data: { currency:'usd', product_data:{name: movie.title},
                       unit_amount: amount*100 }, quantity: 1 }],
       mode: 'payment',
       success_url: `${origin}/loading/my-bookings`,
       cancel_url: `${origin}/my-bookings`,
       metadata: { bookingId },
       expires_at: now + 30 min
     })
  f. booking.paymentLink = session.url; booking.save()
  g. inngest.send({ name: "app/checkpayment", data: { bookingId } })
  h. respond { success:true, url: session.url }
  ▼
Client redirects browser to Stripe-hosted checkout (session.url)
  │
User completes payment on Stripe
  ▼
Stripe → POST /api/stripe  (payment_intent.succeeded)
  Server: find session by payment_intent → bookingId from metadata
          → Booking.findByIdAndUpdate(bookingId, { isPaid:true, paymentLink:"" })
          → inngest.send({ name: "app/show.booked", data:{ bookingId } })
  ▼
Inngest function `send-booking-confirmation-email`
  → Booking.findById(bookingId).populate(show→movie).populate(user)
  → sendEmail(...) via Brevo SMTP
```

### 3.2 Unpaid Booking Auto-Release
```
At booking-creation time: inngest.send("app/checkpayment", {bookingId})
  ▼
Inngest function `release-seats-delete-booking`:
  step.sleepUntil(now + 10 minutes)
  step.run("check-payment-status"):
     booking = Booking.findById(bookingId)
     if (!booking.isPaid):
        show = Show.findById(booking.show)
        for seat in booking.bookedSeats: delete show.occupiedSeats[seat]
        show.markModified('occupiedSeats'); show.save()
        Booking.findByIdAndDelete(booking._id)
```
This is the mechanism that frees seats if a user abandons Stripe
Checkout or the session expires (Stripe session itself is also
configured to expire at 30 minutes).

### 3.3 Adding a Show (Admin)
```
Admin UI: pick movie from TMDB "now playing" results, choose dates/times, price
  │ POST /api/show/add { movieId, showsInput:[{date,time:[...]}], showPrice }  [admin]
  ▼
Server:
  Movie.findById(movieId)
  if not found:
    parallel TMDB calls: GET /movie/:id , GET /movie/:id/credits
    build movieDetails, Movie.create(movieDetails)
  build one Show doc per (date × time) combination
  Show.insertMany(showsToCreate)
  inngest.send("app/show.added", { movieTitle })
  ▼
Inngest function `send-new-show-notifications`
  → User.find({}) → sendEmail to every user
```

### 3.4 Clerk User Lifecycle Sync
```
Clerk (identity provider)
  │ webhook: clerk/user.created | .updated | .deleted
  ▼
Inngest functions:
  syncUserCreation   → User.create({_id, name, email, image})
  syncUserUpdation   → User.findByIdAndUpdate(id, {...})
  syncUserDeletion   → User.findByIdAndDelete(id)
```

### 3.5 Show Reminder Cron
```
Cron: "0 */8 * * *" (every 8 hours)
  ▼
send-show-reminders:
  windowStart = now + 8h - 10min ; windowEnd = now + 8h
  step.run("prepare-reminder-tasks"):
     shows = Show.find({ showTime: { $gte: windowStart, $lte: windowEnd } }).populate('movie')
     for each show: userIds = unique(Object.values(show.occupiedSeats))
                    users = User.find({_id:{$in:userIds}})
                    push {userEmail, userName, movieTitle, showTime} per user
  step.run("send-all-reminders"):
     Promise.allSettled(sendEmail(...) for each task)
  return { sent, failed }
```
**Implementation note:** the query filters on `showTime`, but the `Show`
schema field is named `showDateTime`. As written, this query will not
match any documents — a bug to fix if reminders are expected to fire (see
§6 Known Issues).

---

## 4. Middleware & Auth Details

### `clerkMiddleware()` (global, `server.js`)
Attaches `req.auth()` (function) to every request based on the Clerk
session JWT sent via `Authorization: Bearer <token>` (or Clerk's
cookie-based session on same-origin requests).

### `protectAdmin` (`middleware/auth.js`)
```js
const { userId } = req.auth();
const user = await clerkClient.users.getUser(userId);
if (user.privateMetadata.role !== 'admin') return res.json({success:false, message:"not authorized"});
next();
```
Applied to all `adminRoutes` and to the admin-only `showRoutes`
(`/now-playing`, `/add`).

---

## 5. Frontend Module Design (client/src)

- **`context/AppContext.jsx`** — single global context:
  - State: `isAdmin`, `shows`, `favoriteMovies`.
  - Derived: `user`, `getToken` (from Clerk hooks), `navigate`, `location`.
  - Effects: fetch shows on mount; fetch `isAdmin` + `favorites` whenever
    Clerk `user` becomes available.
  - Axios instance configured with `baseURL = VITE_BASE_URL`.
- **Routing (`App.jsx`)** — React Router v7 route tree; `/admin/*` is
  gated inline (`user ? <Layout/> : <SignIn/>`), with nested routes for
  Dashboard / AddShows / ListShows / ListBookings.
- **`pages/SeatLayout.jsx`** — seat-map UI:
  - Rows `A–J`, grouped as pairs `[[A,B],[C,D],[E,F],[G,H],[I,J]]`, 9 seats
    per row.
  - Client-side rules mirrored (and re-validated server-side): must pick a
    time before seats; max 5 seats; cannot pick an occupied seat.
  - On "proceed to checkout": POST `/api/booking/create`, then
    `window.location.href = data.url` to hand off to Stripe.
- **`pages/MyBookings.jsx` / `Favorite.jsx`** — read-only lists from
  `/api/user/bookings` and `/api/user/favorites`.
- **`pages/admin/*`** — AddShows (calls `/api/show/now-playing` +
  `/api/show/add`), Dashboard (`/api/admin/dashboard`), ListShows
  (`/api/admin/all-shows`), ListBookings (`/api/admin/all-bookings`).

---

## 6. Known Issues / Concurrency Risks (as observed in code)

1. **Seat-hold race condition:** `checkSeatsAvailability` reads
   `Show.occupiedSeats`, and — in a *separate* later step — the booking
   handler re-fetches the show, mutates `occupiedSeats` in memory, and
   calls `.save()`. Between the availability check and the save, two
   concurrent requests for the same seat could both pass the check before
   either persists its write (classic check-then-act race). A stricter
   implementation would use an atomic MongoDB update (e.g.
   `findOneAndUpdate` with a query asserting the seats are still free) to
   close this window.
2. **`send-show-reminders` field name mismatch:** queries
   `Show.find({ showTime: ... })`, but the schema field is
   `showDateTime`. This function will currently match zero shows.
3. **No explicit "signed-in" middleware on `userRoutes` /
   `bookingRoutes.create`:** authorization relies on `req.auth().userId`
   being present; if Clerk middleware fails open (no token), downstream
   code will throw/behave unpredictably rather than returning a clean 401.
4. **Booking amount trusts server-side price only at creation time** —
   correct — but there is no `bookedSeats` uniqueness constraint at the
   DB level; the only true guard is the occupiedSeats check in (1) above.

---

## 7. Environment Configuration

**Server (`server/.env`):** `MONGODB_URI`, Clerk secret key, `STRIPE_SECRET_KEY`,
`STRIPE_WEBHOOK_SECRET`, `TMDB_API_KEY`, `SMTP_USER`, `SMTP_PASS`,
`SENDER_EMAIL`, Cloudinary credentials.

**Client (`client/.env`):** `VITE_BASE_URL` (API origin), Clerk publishable
key, `VITE_TMDB_IMAGE_BASE_URL`.

---

## 8. Suggested Hardening (not implemented today)

- Replace read-then-write seat booking with an atomic
  `findOneAndUpdate({ _id: showId, $nor: selectedSeats.map(s => ({[`occupiedSeats.${s}`]: {$exists:true}})) }, {$set:{...}})`.
- Fix the `showTime` → `showDateTime` field mismatch in the reminder cron.
- Add rate limiting / input validation (e.g. zod/celebrate) on all POST bodies.
- Add an explicit `requireAuth()` guard on `/api/user/*` and
  `/api/booking/create` instead of relying on implicit `req.auth()` checks.
