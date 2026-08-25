Tikto — Ticket Booking System
A production-shaped ticket booking platform for movies and concerts. Customers pick seats from a live visual seat map; seats are held with a configurable TTL and auto-release on abandoned checkout; sold-out shows run a FIFO waitlist that assigns freed seats automatically with a time-limited claim link; every confirmed booking emails a QR e-ticket.

Built as one Next.js 14 app: React frontend, route-handler API, Postgres via Prisma.


1. Table of contents
Stack
Quick start
Environment variables
Seed data — real catalogue
Demo walkthrough
Seat hold & TTL mechanism
Concurrency protection
Waitlist auto-assignment & time-limited offers
QR tickets & email
Database schema
API reference
Roles & permissions
Testing the concurrency guarantee
Deployment
Project layout


2. Stack
Layer
Choice
Why
Framework
Next.js 14 (App Router, route handlers)
One deployable, one type system across API and UI
Language
TypeScript (strict)
Seat-state bugs are the expensive kind — catch them at compile time
Database
PostgreSQL
Row-level locking is the whole concurrency strategy
ORM
Prisma 5
Typed models, plus $executeRaw where the raw conditional UPDATE matters
Auth
JWT (jose) + bcryptjs, httpOnly cookie
Stateless, works on serverless; Authorization: Bearer also accepted for API clients
Validation
Zod
Every request body parsed at the boundary
Styling
Tailwind CSS
Seat grid is pure CSS grid — no canvas, no chart library
QR
qrcode
PNG data URL, embedded in the email and the bookings page
Email
Resend free tier, console fallback
Zero-signup local development
Scheduling
Vercel Cron → /api/cron/sweep, or npm run sweep worker
Two options so any host works



3. Quick start
# 1. install

npm install

# 2. configure

cp .env.example .env         # then set DATABASE_URL and JWT_SECRET

# 3. create the schema

npx prisma migrate dev --name init      # or: npm run db:push

# 4. load real catalogue + venues + shows

npm run db:seed

# 5. run

npm run dev                  # http://localhost:3000

# 6. (separate terminal) run the TTL sweeper

npm run sweep

Need a database in one command?

docker run --name tikto-pg -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=tikto -p 5432:5432 -d postgres:16

# DATABASE_URL="postgresql://postgres:postgres@localhost:5432/tikto?schema=public"

Seeded logins — password Password123! for all:

Role
Email
Admin
admin@tikto.dev
Organiser
organiser@tikto.dev
Customer
customer@tikto.dev, rahul@tikto.dev, sana@tikto.dev, dev@tikto.dev



4. Environment variables
Variable
Required
Default
Purpose
DATABASE_URL
✅
—
Postgres connection string (Neon/Supabase/local)
JWT_SECRET
✅
—
Session signing key. openssl rand -base64 32
NEXT_PUBLIC_APP_URL
✅
http://localhost:3000
Used in QR payloads and email links
SEAT_HOLD_TTL_SECONDS


600
Checkout hold lifetime
WAITLIST_OFFER_TTL_SECONDS


900
How long a waitlisted customer has to claim
RESEND_API_KEY


(blank)
Blank ⇒ emails print to the server console
EMAIL_FROM


Tikto <onboarding@resend.dev>
Sender identity
TMDB_API_KEY


(blank)
Blank ⇒ seeder uses the bundled real-title catalogue
CRON_SECRET


dev-cron-secret
Guards /api/cron/sweep in production



5. Seed data — real catalogue
Movies. With TMDB_API_KEY set, npm run db:seed calls TMDB's public API — /genre/movie/list, /movie/now_playing?region=IN, and /movie/{id} for runtimes — and creates events from whatever is genuinely in cinemas, including real posters served from image.tmdb.org. Get a free key at https://www.themoviedb.org/settings/api.

Without a key the seeder falls back to a bundled catalogue of real titles (prisma/tmdb.ts) carrying the same TMDB ids, so nothing about the flow changes and setup needs no signup.

Venues. Four real venues with layouts modelled on their actual seating shape — lettered rows, continuous seat numbering, centre and side aisles rendered as gaps in the grid (src/lib/layout.ts):

Venue
City
Categories
Seats
PVR ICON — Phoenix Palladium
Mumbai
Recliner / Premium / Standard
132
INOX — Nariman Point
Mumbai
Premium / Standard
90
Cinepolis — Nexus Koramangala
Bengaluru
Premium / Standard
86
Jawaharlal Nehru Stadium — Arena Floor
New Delhi
Golden Circle / Silver / General
142


Pricing uses realistic Indian multiplex and arena bands (₹220 standard → ₹650 recliner; ₹3,500 → ₹12,000 for arena concerts). Amounts are stored in paise as integers — never floats.

The seeder also builds one deliberately sold-out show with four people already waitlisted, so the auto-assignment flow is demonstrable the moment the app boots.


6. Demo walkthrough (5 minutes)
A. Hold + auto-release

Sign in as customer@tikto.dev, open any show, select two seats, Hold seats & continue.
In a private window as rahul@tikto.dev, open the same show — those seats now render orange (held).
Close the checkout tab and wait out the timer (or set SEAT_HOLD_TTL_SECONDS=30). The seat map turns them green again on its next poll, with no sweeper needed.

B. Concurrency

npm run test:concurrency     # 20 parallel holds on the same seats → exactly 1 wins

C. Waitlist auto-assignment

As customer@tikto.dev, open My bookings and cancel TKT-DEMO-0001.
The seats free, and each is offered instantly to the head of the Premium queue. The offer email (console or Resend) carries /offer/<token> with a live countdown.
Open that link as the offered customer and claim — a booking and QR ticket are issued.
Let a different offer lapse instead: the sweeper marks it expired and re-offers the seat to the next person in the queue automatically.

D. QR verification — open any confirmed booking, hit Verify ticket, or scan the QR: it resolves to /verify/<reference>, which reports valid/cancelled live.


7. Seat hold & TTL mechanism
A ShowSeat row exists for every (show, seat) pair, created when the show is created. Its lifecycle:

AVAILABLE ──hold──▶ HELD ──commit──▶ BOOKED ──cancel──▶ AVAILABLE

     ▲                │                                     │

     │                └──release / TTL lapse────────────────┘

     │

     └──offer──▶ OFFERED ──claim──▶ BOOKED

                     └──offer TTL lapse──▶ AVAILABLE (re-offered to next in queue)

Holding writes three columns together: state='HELD', holdId (an opaque UUID identifying the checkout session) and holdExpiresAt = now() + TTL.

Expiry is lazy, and the sweeper is only housekeeping. Both the read path and the write path treat a lapsed lock as free:

getSeatMap() reports a HELD/OFFERED seat whose holdExpiresAt has passed as AVAILABLE.
The hold statement's WHERE matches state='AVAILABLE' OR (state IN ('HELD','OFFERED') AND holdExpiresAt < NOW()).

So a late or dead sweeper can never wrongly block a sale — the worst case is a seat looking taken for a few extra seconds on a screen that hasn't polled yet. /api/cron/sweep (Vercel Cron, every minute) or npm run sweep then physically normalises the rows and drives waitlist re-offers.

Time is always the database's NOW(), never a Node clock, so multiple serverless instances with skewed clocks stay consistent. The checkout countdown is cosmetic and re-syncs with the server every 15 seconds.


8. Concurrency protection
Every transition is one conditional SQL UPDATE whose WHERE clause encodes its own precondition (src/lib/seats.ts):

UPDATE show_seats

   SET state='HELD', "holdId"=$1, "holdExpiresAt"=$2, version=version+1, "updatedAt"=NOW()

 WHERE "showId"=$3

   AND "seatId" IN ($4…)

   AND (state='AVAILABLE' OR (state IN ('HELD','OFFERED') AND "holdExpiresAt" < NOW()));

Postgres takes a row lock before evaluating the predicate. Two requests for seat H7 serialise: the first flips the row; the second waits, re-reads the row under READ COMMITTED, no longer matches the predicate, and updates 0 rows.

The application then asserts rowsUpdated === seatIds.length. A partial match means somebody won at least one seat, so it throws and the enclosing transaction rolls back — a multi-seat hold is all-or-nothing, never a half-hold that strands seats.

Why this and not the alternatives:

Approach
Why not
SELECT … FOR UPDATE then UPDATE
An extra round trip per seat; the single statement already locks
Read-then-write with a version check
Lost-update window between read and write; needs a retry loop
Redis lock / in-process mutex
A second source of truth that can disagree with the DB, and it breaks across serverless instances
SERIALIZABLE isolation
Correct but pays retry cost on every unrelated booking


Deadlocks (two overlapping seat sets locked in opposite scan order) are the only genuine retry case, and withRetry() retries only on SQLSTATE 40P01/40001 — never on a business rejection, so a customer never silently steals a seat on retry. A @@unique([showId, seatId]) index keeps the table itself honest, and the bookings ← show_seats foreign key means a confirmed seat can only point at a real booking.


9. Waitlist auto-assignment & time-limited offers
Joining. WaitlistEntry is a FIFO queue per (show, seatCategory). position is allocated inside a transaction, and @@unique([showId, categoryId, userId]) stops the same customer queueing twice.

Freeing. On cancellation (or when an offer lapses) offerFreedSeats(showId, seatIds) runs per seat:

Read the seat's category, take the lowest-position WAITING entry for it.
Conditionally flip the seat to OFFERED with holdId = offerToken (a UUID) and holdExpiresAt = now() + WAITLIST_OFFER_TTL_SECONDS. If that update touches 0 rows a walk-in customer beat us to it — fine, we simply move on and the entry stays queued.
Mark the entry OFFERED, then send the claim email outside the transaction, so a mail-provider outage can never un-reserve a seat that is already reserved.

Claiming. /offer/<token> shows a countdown; POST /api/offers/:token/claim verifies the token belongs to the signed-in customer and then calls the ordinary booking commit. The offer token is the hold id, so claiming reuses commitHold() — one code path, one set of guarantees, no parallel booking logic that could drift.

Lapsing. expireLapsedOffers() marks entries EXPIRED, releases their seats, and immediately calls offerFreedSeats() again, so the seat walks down the queue on its own until somebody takes it or the queue empties. offerCount records how many times an entry was offered, for auditing.

Every transition writes an AuditLog row, which is what makes the machinery reviewable after the fact.


10. QR tickets & email
The QR encodes ${NEXT_PUBLIC_APP_URL}/verify/<reference> rather than a bare string, so scanning it with any phone camera opens a page that checks the booking live — a cancelled ticket reads as invalid immediately, which a static QR payload could never do.

References look like TKT-A7KP-3M9T, using a 32-char alphabet with I, O, 0 and 1 removed so they survive being read aloud at a counter.

The PNG data URL is stored on the booking, embedded inline in the confirmation email, attached as ticket-qr.png, and re-rendered on the bookings page. With RESEND_API_KEY blank, emails print to the server console — the whole flow stays demonstrable with no third-party account.

Emails sent: booking confirmed (with QR), waitlist joined (with position), waitlist offer (with claim link and deadline), booking cancelled (with refund amount).


11. Database schema
11 models. prisma/schema.prisma is the source of truth; this is the shape:

User (CUSTOMER|ORGANISER|ADMIN)

 ├─< Event ──< Show ──< ShowPrice >── SeatCategory

 │                │                        │

 │                ├─< ShowSeat >── Seat ───┘        ← the concurrency table

 │                ├─< Booking ──< ShowSeat

 │                └─< WaitlistEntry

 └─< Booking / WaitlistEntry

Venue ──< SeatCategory, Seat, Show

AuditLog (append-only)

Key decisions:

ShowSeat is materialised up front, one row per seat per show. Deriving availability from bookings instead would make an atomic per-seat lock impossible.
Seat is immutable venue geometry; per-show state lives only on ShowSeat. Re-pricing or blocking a seat for one show never touches the layout.
Prices live on ShowPrice, per (show, category) — matinee and weekend pricing need no schema change.
Money is integer paise. No floats anywhere in the money path.
OFFERED reuses holdId/holdExpiresAt, so the sweeper, the seat map and the commit path handle waitlist offers with zero special-casing.
Indexes that matter: @@unique([showId, seatId]), @@index([showId, state]) (seat map), @@index([state, holdExpiresAt]) (sweeper), @@index([showId, categoryId, status, position]) (queue head).


12. API reference
All responses are JSON. Auth is the tikto_session httpOnly cookie, or Authorization: Bearer <jwt>. Errors: { "error": "message" } with 400 validation · 401 unauthenticated · 403 wrong role · 404 missing · 409 conflict (seat taken) · 410 hold/offer expired · 422 schema violation.
Auth
Method
Path
Auth
Body / notes
POST
/api/auth/register
—
{name, email, password, phone?, role?} → sets cookie, returns {user, token}
POST
/api/auth/login
—
{email, password}
POST
/api/auth/logout
—
Clears the cookie
GET
/api/auth/me
—
{user} or {user:null}

Catalogue
Method
Path
Auth
Notes
GET
/api/events
—
Filters: q, type=MOVIE|CONCERT, city, language, date=YYYY-MM-DD
GET
/api/events/:id
—
Event + upcoming shows with availableSeats / soldOut
POST
/api/events
organiser
{type, title, description, posterUrl?, language, genre, durationMin, certificate}
GET
/api/shows?eventId=
—
Shows with venue and prices
POST
/api/shows
organiser
{eventId, venueId, startsAt, screen, prices:[{categoryId, priceCents}]} — also materialises every ShowSeat
GET
/api/venues
—
Venues with categories and seat counts
POST
/api/venues
admin
{name, city, address, categories[], rows:[{rowLabel, seats, category, aisleAfter[]}]}

Seats, holds, bookings
Method
Path
Auth
Notes
GET
/api/shows/:id/seatmap
—
Full grid + per-category availability. no-store; polled every 4s
POST
/api/shows/:id/hold
customer
{seatIds[]} (max 10) → {holdId, expiresAt, totalCents}. 409 if any seat is taken
GET
/api/holds/:holdId
—
{active, secondsLeft, seats, totalCents, isWaitlistOffer}
DELETE
/api/holds/:holdId
customer
Abandon checkout, release immediately
POST
/api/bookings
customer
{showId, holdId, contactName, contactEmail, contactPhone?} → booking + QR + email. 410 if the hold lapsed
GET
/api/bookings
customer
Booking history with QR data URLs
GET
/api/bookings/:id
owner/admin
Single booking
POST
/api/bookings/:id/cancel
owner/admin
Frees seats, fires waitlist offers → {reference, seatsReleased, offersSent}
GET
/api/verify/:reference
—
Gate scanner endpoint → {valid, reason, holder, seats, …}

Waitlist
Method
Path
Auth
Notes
POST
/api/shows/:id/waitlist
customer
{categoryId, quantity} → {entry, position}
GET
/api/shows/:id/waitlist
customer
Queue depth per category + the caller's own place
GET
/api/waitlist
customer
Every queue the customer is in, with live positions
DELETE
/api/waitlist/:id
owner/admin
Leave the queue (409 while an offer is live)
GET
/api/offers/:token
—
Offer detail + secondsLeft
POST
/api/offers/:token/claim
offered customer
Converts the offer into a booking

Organiser & operations
Method
Path
Auth
Notes
GET
/api/organiser/events
organiser
Catalogue with seats sold and revenue per show
GET
/api/organiser/shows/:id/summary
organiser
Occupancy, gross/refunded revenue, per-category held/offered/free, waitlist depth and conversions
GET/POST
/api/cron/sweep
CRON_SECRET
Expires offers, releases holds, re-offers freed seats → counts


Example

curl -X POST localhost:3000/api/shows/$SHOW/hold \

  -H 'Content-Type: application/json' -H "Authorization: Bearer $JWT" \

  -d '{"seatIds":["seat_a","seat_b"]}'

# → {"holdId":"5f0…","expiresAt":"…","seatIds":[…],"totalCents":44000,"ttlSeconds":600}


13. Roles & permissions
Capability
Customer
Organiser
Admin
Browse, seat map, hold, book, cancel own
✅
✅
✅
Join / leave waitlist, claim offers
✅
✅
✅
Create events and schedule shows
—
✅
✅
Booking summary & revenue
—
own events
all
Create venues and seat layouts
—
—
✅
Cancel anyone's booking
—
—
✅


Enforced server-side by requireUser(req, roles); the nav only hides links.


14. Testing the concurrency guarantee
npm run test:concurrency

Against your live database it asserts:

20 simultaneous holds on identical seats → exactly one succeeds.
All contested seats carry one holdId — no split hold.
A request overlapping a held seat is rejected entirely, leaving its other seat still available.
A lapsed hold reads as AVAILABLE before the sweeper runs (lazy expiry).
Another customer can immediately re-hold those seats.
An explicit release returns every seat to sale.


15. Deployment
Vercel + Neon (recommended)

Create a Neon Postgres database, copy the pooled connection string.
Import the repo into Vercel; set every variable from .env.example (NEXT_PUBLIC_APP_URL = your deployed URL).
Build command npm run build — it runs prisma generate first.
Apply the schema once: npx prisma migrate deploy locally against the production DATABASE_URL, then npm run db:seed if you want the demo catalogue.
vercel.json already registers the cron; Vercel sends Authorization: Bearer $CRON_SECRET automatically when that variable is set.

Render / Railway — deploy the web service the same way, then add a second background worker running npm run sweep instead of the cron (both do the same work).

Notes: use a pooled connection string on serverless; NEXT_PUBLIC_APP_URL must be the real URL or QR codes and offer links will point at localhost; verify a domain in Resend to send to arbitrary addresses (the free sandbox sender only delivers to your own).


16. Project layout
prisma/

  schema.prisma        11 models — ShowSeat is the concurrency table

  seed.ts              venues, TMDB catalogue, shows, demo sold-out show + waitlist

  tmdb.ts              live TMDB ingestion + bundled fallback catalogue

src/lib/

  seats.ts             hold / release / commit / sweep — the conditional-UPDATE engine

  waitlist.ts          FIFO queue, auto-assignment, time-limited offers

  bookings.ts          checkout commit, QR issue, cancellation + re-offer

  auth.ts              JWT sessions, bcrypt, requireUser(roles)

  email.ts             Resend + console fallback, all four templates

  qr.ts                QR payload, booking references

  layout.ts            venue layouts → seat grid coordinates

  env.ts http.ts       config, JSON helpers, error mapping

src/app/               pages (browse, event, seat map, checkout, bookings, waitlist, offer,

                       organiser, admin, verify) and /api route handlers

scripts/

  sweep.ts             standalone TTL worker

  concurrency-test.ts  the assertions in §14

docs/system-design.md  800-word design write-up


17. Known limits
Payment is simulated — POST /api/bookings issues the booking directly; a real gateway would hold the seats through an intent/webhook round trip, which the HELD → BOOKED transition is already shaped for. The seat map polls every 4 seconds rather than using WebSockets (polling survives serverless cold starts and costs nothing); the state machine is transport-agnostic, so swapping in Postgres LISTEN/NOTIFY or Pusher would touch only the read path. Waitlist offers are per-seat, so a party of three waiting together receives three separate offers rather than one grouped offer.
