# Product Requirements Document (PRD)
## QuickShow — Online Movie Ticket Booking Platform

**Version:** 1.0 (reverse-engineered from codebase)
**Date:** August 2026

---

## 1. Overview

QuickShow is a web application that lets moviegoers browse currently-playing
movies, view showtimes, select seats on a visual seat map, pay online, and
receive booking confirmations and reminders by email. A separate admin
console lets theater staff add new shows (pulling movie metadata from TMDB),
monitor revenue/bookings, and manage the show catalog.

The product has two user-facing surfaces:
- **Customer app** — browse, book, pay, manage bookings & favorites.
- **Admin dashboard** — add shows, view all bookings, view revenue metrics.

---

## 2. Problem Statement

Independent cinemas and small theater chains need a lightweight booking
system that:
- Avoids maintaining their own movie metadata (posters, cast, synopsis).
- Prevents double-booking of seats.
- Automatically frees up seats reserved by users who never complete payment.
- Notifies customers about upcoming shows and confirms paid bookings.
- Gives staff a simple way to schedule new shows without manual data entry.

## 3. Goals

| # | Goal | Success signal |
|---|------|-----------------|
| G1 | Let a user book a seat in under 2 minutes | Time from landing on a movie page to Stripe redirect |
| G2 | Guarantee no seat is double-booked | Zero overlapping `occupiedSeats` entries per show |
| G3 | Auto-release unpaid holds | Seats held >10 min without payment are freed automatically |
| G4 | Give admins visibility into revenue & bookings | Dashboard reflects `isPaid` bookings only |
| G5 | Keep movie data fresh without manual entry | Movie is pulled from TMDB on first use and cached locally |

## 4. Non-Goals

- Multi-theater / multi-location seat maps (current model is one seat map per show, not per venue).
- Refunds / cancellations initiated by the customer.
- Native mobile apps (web-only, responsive via Tailwind).
- Multi-currency support (Stripe checkout is hardcoded to USD).
- Partial payments, group discounts, or promo codes.

---

## 5. User Roles

### 5.1 Guest / Anonymous visitor
- Can browse movies, view showtimes, view trailers.
- Cannot book — prompted to sign in when attempting to book.

### 5.2 Registered User (Clerk-authenticated)
- Everything a guest can do, plus:
  - Select seats and book a show.
  - Pay via Stripe Checkout.
  - View own booking history ("My Bookings").
  - Mark/unmark movies as favorites.
  - Receive booking-confirmation and show-reminder emails.

### 5.3 Admin
- A Clerk user whose `privateMetadata.role === "admin"`.
- Everything a registered user can do, plus access to `/admin/*`:
  - Add a new show (movie + one or more dates/times + price).
  - View dashboard (total bookings, total revenue, active shows, total users).
  - View all shows (list-shows).
  - View all bookings across all users (list-bookings).

---

## 6. Functional Requirements

### 6.1 Authentication & User Sync
- FR1: Users authenticate via Clerk (hosted sign-in/sign-up).
- FR2: On `clerk/user.created`, `clerk/user.updated`, `clerk/user.deleted`
  webhook events, the local `User` collection is kept in sync (name, email,
  profile image) via background functions.
- FR3: Admin-only API routes are protected by a middleware that checks the
  Clerk user's `privateMetadata.role`.

### 6.2 Movie Catalog
- FR4: Admin can fetch "now playing" movies from TMDB to choose from when
  scheduling a show.
- FR5: When a show is added for a movie not yet in the local database, the
  system fetches movie details + cast credits from TMDB and persists them
  locally (poster, backdrop, overview, genres, cast, rating, runtime,
  release date, tagline).
- FR6: Public users can browse all movies that currently have upcoming shows.
- FR7: A movie detail page shows synopsis, cast, trailer, and available
  dates/times grouped by date.

### 6.3 Show Scheduling (Admin)
- FR8: Admin selects a movie, one or more dates, one or more times per date,
  and a ticket price; the system creates one `Show` document per
  date+time combination.
- FR9: Adding a show triggers a "new show" notification email to all
  registered users.

### 6.4 Seat Selection & Booking
- FR10: Seat map is fixed at 9 seats × 10 rows (A–J), grouped visually into
  pairs of rows.
- FR11: A user may select at most 5 seats per booking.
- FR12: Already-occupied seats (from other confirmed or pending bookings)
  are shown as unavailable and cannot be selected.
- FR13: On checkout, the server re-validates seat availability before
  creating a booking (race-condition guard).
- FR14: Booking amount = show price × number of selected seats.
- FR15: A Stripe Checkout Session is created for the booking amount in USD;
  the session expires after 30 minutes.
- FR16: Selected seats are marked occupied (tagged with the booking user's
  ID) at booking-creation time, before payment completes — this holds the
  seats during checkout.
- FR17: If payment is not completed within 10 minutes, a scheduled job
  releases the held seats and deletes the unpaid booking.
- FR18: On successful Stripe payment (`payment_intent.succeeded` webhook),
  the booking is marked paid and a confirmation email is sent.

### 6.5 My Bookings
- FR19: A signed-in user can view all of their past and pending bookings,
  including movie, showtime, seats, amount, and payment status.

### 6.6 Favorites
- FR20: A signed-in user can toggle a movie as a favorite; favorites are
  stored in the user's Clerk `privateMetadata` (not the local database).
- FR21: A "Favorites" page lists the user's favorited movies.

### 6.7 Notifications (Email)
- FR22: Booking confirmation email sent after successful payment.
- FR23: Reminder email sent to all ticket holders ~8 hours before showtime
  (via a recurring scheduled job).
- FR24: New-show announcement email sent to all users when a show is added.

### 6.8 Admin Dashboard
- FR25: Dashboard shows total paid bookings, total revenue (sum of paid
  booking amounts), count of active (upcoming) shows, and total registered
  users.
- FR26: Admin can view a list of all upcoming shows and all bookings
  (with user and movie details) across the platform.

---

## 7. Non-Functional Requirements

- **NFR1 — Data consistency:** Seat holds must prevent two users from
  booking the same seat for the same show.
- **NFR2 — Security:** Payment webhook requests must be signature-verified;
  admin routes must be inaccessible to non-admin users.
- **NFR3 — Reliability of async work:** Seat-release, reminders, and email
  sends must survive server restarts (durable background execution).
- **NFR4 — Performance:** Movie/show browsing pages should load without
  blocking on external TMDB calls at request time except when data is not
  yet cached locally.
- **NFR5 — Availability:** Client and server are deployed independently
  (Vercel) and can scale/redeploy without shared state beyond MongoDB and
  Clerk.

---

## 8. Key Entities (Product View)

- **User** — id, name, email, avatar (synced from Clerk).
- **Movie** — cached TMDB movie metadata, keyed by TMDB id.
- **Show** — a screening: movie + date/time + price + map of occupied seats.
- **Booking** — a user's reservation for a show: seats, amount, payment
  status, Stripe payment link.

## 9. Out of Scope / Known Limitations (as built)

- No cancellation/refund flow for paid bookings.
- No per-theater/screen distinction — one global seat layout shape per show.
- Only one payment provider (Stripe) and one currency (USD).
- Admin role is a boolean flag in Clerk metadata; no granular permission
  levels (e.g., manager vs. staff).

---

## 10. Open Questions for Future Iterations

1. Should seat layout support variable theater sizes/screen types?
2. Should there be self-service cancellation with partial refunds?
3. Should pricing support tiers (e.g., premium seats, matinee pricing)?
4. Is multi-currency / regional pricing needed?
