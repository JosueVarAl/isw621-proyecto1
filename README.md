# Cabina — Cultural Center Management API

**Course:** Programación Web II (`ISW-621`) — Proyecto #1  
**Repository:** `isw621-proyecto1`

## 1. Project Overview

**Cabina** is a RESTful backend API built with **Express.js** and **MongoDB (Mongoose)** to manage the daily operations of a cultural center's rehearsal spaces.

At the cultural center, musical **bands** (composed of **musicians**) reserve **rehearsal rooms** on specific dates and hours. Alongside a room booking, a band can request **equipment** from the center's inventory (such as microphones, amplifiers, or support drum kits). When a booking is created, the requested equipment is atomically loaned out and deducted from the available inventory. If the booking is later canceled, the room slot is released and all borrowed equipment is returned to the inventory within a single database transaction.

### Key Technical Constraints
- **Stack:** Node.js, Express, and MongoDB via Mongoose.
- **Format:** Backend-only API; every response (success or error) is formatted as JSON (`{ "error": "mensaje claro" }` on errors).
- **Environment:** Database connection string is stored in `.env` (with a credential-free `.env.example` committed to the repository).
- **Testing:** Accompanied by an exported **Postman Collection (`Cabina`)** and **Postman Environment**.

---

## 2. Team Members (Integrantes)

- **Josué Vargas**
- *(Add teammate 2)*
- *(Add teammate 3)*

---

## 3. Domain Models

All six models include automatic `timestamps` (`createdAt`, `updatedAt`):

| Model | Key Fields & Validations |
| :--- | :--- |
| **`Room`** | `name` (unique, required, 3–40 chars), `capacity` (required, 1–30), `floor` (required), `hasDrumKit` (boolean, default `false`), `status` (`disponible` \| `mantenimiento`, default `disponible`). |
| **`Band`** | `name` (unique, required, 2–50 chars), `genre` (`rock` \| `jazz` \| `coral` \| `otro`, default `otro`), `active` (boolean, default `true`), `contactName` (required). |
| **`Musician`** | `name` (required), `instrument` (required), `email` (unique, required), `band` (required `ObjectId` $\rightarrow$ `Band`), `active` (boolean, default `true`). |
| **`Booking`** | `room` (required `ObjectId` $\rightarrow$ `Room`), `band` (required `ObjectId` $\rightarrow$ `Band`), `date` (required), `startHour` (int 8–21), `endHour` (int 9–22), `status` (`apartada` \| `confirmada` \| `cancelada`, default `apartada`), `notes` (optional, max 200 chars). Includes virtual `durationHours` (`endHour - startHour`) serialized in JSON. |
| **`Equipment`** | `name` (unique, required), `category` (`audio` \| `cuerdas` \| `percusion`), `totalUnits` (int `> 0`), `availableUnits` (int `0..totalUnits`, initialized equal to `totalUnits`). |
| **`Loan`** | `booking` (required `ObjectId` $\rightarrow$ `Booking`), `equipment` (required `ObjectId` $\rightarrow$ `Equipment`), `units` (int `> 0`), `status` (`prestado` \| `devuelto`). Created during booking creation and returned upon booking cancellation. |

---

## 4. Índices (Database Indexes)

All required indexes are declared directly in the Mongoose schemas:

| Model | Index Definition | Query It Serves |
| :--- | :--- | :--- |
| **`Room`** | `{ name: 1 }` (`unique: true`) | Enforces unique room names and speeds up lookups/searches by room name (`POST /api/rooms`, `PUT /api/rooms/:id`, `GET /api/rooms?search=...`). |
| **`Band`** | `{ name: 1 }` (`unique: true`) | Enforces unique band names and accelerates sorting/searching bands by name (`POST /api/bands`, `GET /api/bands`). |
| **`Musician`** | `{ email: 1 }` (`unique: true`) | Guarantees no two musicians share an email and speeds up email lookups/searches (`POST /api/musicians`, `GET /api/musicians?search=...`). |
| **`Musician`** | `{ band: 1 }` | Optimizes finding all musicians belonging to a specific band (`GET /api/bands/:id/musicians`, `GET /api/musicians?band=...`, virtual populate on `Band`, and counting active musicians during booking validation). |
| **`Equipment`** | `{ name: 1 }` (`unique: true`) | Enforces unique equipment names and speeds up default sorting and searching by name (`GET /api/equipment`). |
| **`Booking`** | `{ room: 1, date: 1, startHour: 1 }` (Compound) | Optimizes the daily room schedule endpoint (`GET /api/rooms/:id/schedule?date=...`) and the schedule overlap verification during booking creation/updates. |
| **`Loan`** | `{ booking: 1 }` | Speeds up retrieving all equipment loans associated with a booking (`GET /api/bookings/:id`, `GET /api/loans?booking=...`, and finding loans to return when canceling a booking). |
| **`Loan`** | `{ equipment: 1, status: 1 }` (Compound) | Optimizes filtering loans by equipment item and loan status (`GET /api/loans?equipment=...&status=...`) and checking active loans for an equipment item. |

---

## 5. Business Rules & Validations

### Booking Creation & Updates
1. **Operating Hours & Duration:** `startHour` (8–21), `endHour` (9–22), `endHour > startHour`, and maximum duration of **3 hours** (`endHour - startHour <= 3`).
2. **Future or Current Date:** The booking `date` must be today or a future date.
3. **Room Availability:** The target room must exist and have `status === 'disponible'`.
4. **Band & Capacity Validation:** The target band must exist, have `active === true`, have **at least 1 active musician**, and its total number of active musicians must not exceed `room.capacity`.
5. **No Schedule Overlap:** Cannot overlap with any other `apartada` or `confirmada` booking for the same room and date:
   $$\text{startHour} < \text{other.endHour} \quad \land \quad \text{endHour} > \text{other.startHour}$$
   Canceled bookings (`cancelada`) do not block the room. When editing a booking, it does not conflict with itself.
6. **Valid State Transitions:**
   - `apartada` $\rightarrow$ `confirmada` or `cancelada`
   - `confirmada` $\rightarrow$ `cancelada`
   - `cancelada` $\rightarrow$ *(final state; no further changes allowed)*
7. **Equipment Requests:** Sent in `POST /api/bookings` as `equipment: [{ equipmentId, units }]` (optional/empty array allowed). Cannot contain duplicate `equipmentId` items, and each `units` value must fit within `availableUnits`. Modifying equipment on an already created booking is out of scope.

### Protected Deletions & Updates (`409 Conflict`)
- **Delete Room:** Returns `409` if the room has any `apartada` or `confirmada` bookings.
- **Delete Band:** Returns `409` if the band has any `apartada` or `confirmada` bookings, or if it still has musicians registered to it.
- **Delete Equipment:** Returns `409` if `availableUnits < totalUnits` (units are currently on loan).
- **Update Equipment (`PUT /api/equipment/:id`):** Only `name`, `category`, and `totalUnits` can be updated directly (`availableUnits` only changes via booking transactions). If the new `totalUnits` is less than the currently loaned units (`totalUnits - availableUnits`), it returns `409`.

---

## 6. Transacción (Multi-Collection Transactions)

Both **booking creation (`POST /api/bookings`)** and **booking cancellation (`PATCH /api/bookings/:id/cancel` / status transition to `cancelada`)** execute inside a MongoDB multi-document transaction using `mongoose.startSession()` and `session.withTransaction()`:

1. **Booking Creation Transaction (`createBooking`):**
   - **Collections touched:** `rooms`, `bands`, `musicians`, `bookings`, `equipment`, `loans`.
   - **Operations:** Inside `withTransaction`, verifies room availability, active band/musicians vs. capacity, and schedule overlap; validates equipment availability and decrements `Equipment.availableUnits` atomically via `$inc: { availableUnits: -units }`; creates the `Booking` document in status `apartada`; and inserts the `Loan` documents in status `prestado`.
   - **Atomicity guarantee:** If any validation fails or any equipment item lacks sufficient `availableUnits`, the transaction aborts and rolls back completely—preventing orphan bookings without equipment or deducted inventory without a booking.

2. **Booking Cancellation Transaction (`cancelBooking`):**
   - **Collections touched:** `bookings`, `loans`, `equipment`.
   - **Operations:** Inside `withTransaction`, updates `Booking.status` to `cancelada`, transitions all associated `Loan` records to `devuelto`, and restores inventory on each `Equipment` document using `$inc: { availableUnits: units }`.
   - **Note on Confirmation:** Confirming a booking (`apartada` $\rightarrow$ `confirmada`) does not modify inventory because units were already deducted when the booking was initially reserved.

3. **How `409` Rollbacks Are Tested:**
   - Attempting to create a booking requesting more equipment units than `availableUnits` (or with an overlapping schedule) returns `409 Conflict`, and subsequent `GET /api/equipment` / `GET /api/bookings` requests verify that neither `availableUnits` nor `Booking`/`Loan` records were altered.
   - Attempting to cancel an already canceled booking returns `409 Conflict` without double-incrementing `availableUnits`.

---

## 7. API Endpoints Summary

### Pagination, Sorting & Search (All 6 List Endpoints)
All collection list endpoints (`GET /api/rooms`, `/api/bands`, `/api/musicians`, `/api/bookings`, `/api/equipment`, `/api/loans`) accept `page` (default `1`), `limit` (default `10`, max `50`), `sort` (`field` or `-field`), and case-insensitive `search`, returning:
```json
{
  "data": [],
  "page": 1,
  "limit": 10,
  "total": 0,
  "pages": 0
}
```

| Resource | Endpoints | Allowed `sort` (Default) | `search` Fields & Query Filters |
| :--- | :--- | :--- | :--- |
| **Rooms** | `POST`, `GET`, `GET /:id`, `PUT /:id`, `DELETE /:id`, `GET /:id/schedule?date=...`, `GET /stats` | `name`, `capacity`, `floor`, `createdAt` (`name`) | Search: `name`. |
| **Bands** | `POST`, `GET`, `GET /:id`, `PUT /:id`, `DELETE /:id`, `GET /:id/musicians`, `GET /:id/bookings` | `name`, `createdAt` (`name`) | Search: `name`. |
| **Musicians** | `POST`, `GET`, `GET /:id`, `PUT /:id`, `DELETE /:id` | `name`, `instrument`, `createdAt` (`name`) | Search: `name`, `email`. Filters: `band`, `active`. |
| **Equipment** | `POST`, `GET`, `GET /:id`, `PUT /:id`, `DELETE /:id`, `GET /stats` | `name`, `availableUnits`, `createdAt` (`name`) | Search: `name`. Filters: `category`, `available=true`. |
| **Bookings** | `POST`, `GET`, `GET /:id`, `PUT /:id`, `PATCH /:id/cancel`, `GET /stats` | `date`, `startHour`, `createdAt` (`date`, then `startHour`) | Search: `notes`. Filters: `status`, `room`, `band`, `date`, `from`, `to`. |
| **Loans** | `GET`, `GET /:id` | `units`, `createdAt` (`createdAt`) | Search: `status`. Filters: `status`, `booking`, `equipment`. |

### Aggregation Summary Endpoints
- `GET /api/bookings/stats`: Counts bookings grouped by `status`.
- `GET /api/rooms/stats`: Returns per room how many active bookings (`apartada` or `confirmada`) it has.
- `GET /api/equipment/stats`: Returns per equipment `category` the sum of `totalUnits` and `availableUnits`.

---

## 8. Installation & Running the Project

### Prerequisites
- **Node.js** v18+
- **MongoDB Replica Set** or **MongoDB Atlas** cluster (required for multi-collection transactions)

### Setup Steps
1. Clone the repository and install dependencies:
   ```bash
   npm install
   ```
2. Copy the example environment file and configure your MongoDB connection string:
   ```bash
   cp .env.example .env
   ```
   Edit `.env`:
   ```env
   PORT=3000
   MONGODB_URI=mongodb+srv://<username>:<password>@<cluster>.mongodb.net/cabina?retryWrites=true&w=majority
   ```
3. Start the development server:
   ```bash
   npm run dev
   ```
