# Cabina Cultural Center — Product Backlog

This document breaks down the **Cabina Cultural Center** project into user stories written in plain, everyday language. Each story follows the required structure:
- **US# - Title**
- **Requirements**
- **Acceptance Criteria**
- **Technical implications**

---

## US1 - Project Setup and Shared Configuration

### Requirements
- As a development team member, I want the project workspace, database connection, and configuration templates set up cleanly so that any team member or instructor can download, configure, and run the system on their computer without exposing private passwords.

### Acceptance Criteria
- The system starts up with a single command and connects to the database.
- Private connection credentials are kept in a local secret file that is never uploaded to the code repository.
- An example configuration file is included showing what settings are needed without revealing any real usernames or passwords.
- All responses from the system are delivered in a consistent, structured format.

### Technical implications
- Initialize Node.js project with **Express** and **Mongoose** (`package.json` with `npm run dev` and `npm start`).
- Store `MONGODB_URI` and `PORT` in `.env` (ignored via `.gitignore`).
- Provide `.env.example` with placeholder values (`mongodb+srv://<user>:<password>@cluster.mongodb.net/cabina` without credentials).
- Configure Express JSON body parser (`express.json()`) with a middleware to catch malformed JSON payloads and return HTTP `400` (`{ "error": "mensaje claro" }`).
- MongoDB connection must target a Replica Set (e.g., MongoDB Atlas) to support multi-collection transactions.

---

## US2 - Register and Edit Rehearsal Rooms

### Requirements
- As a cultural center administrator, I want to register new rehearsal rooms and update their information (such as their name, how many people fit inside, which floor they are on, whether they have a drum kit, and whether they are open or under maintenance) so that bands know what spaces exist at the center.

### Acceptance Criteria
- A new room can be registered by providing a unique name (between 3 and 40 characters), a maximum capacity (between 1 and 30 people), and a floor location.
- If I do not specify whether the room has a drum kit, the system assumes it does not.
- If I do not specify the room's condition, the system marks it as available by default.
- Two rooms cannot share the exact same name.
- I can update an existing room's details at any time, such as switching it to maintenance mode when repairs are needed.

### Technical implications
- **`Room` Mongoose Schema:**
  - `name`: `String`, required, unique, `minlength: 3`, `maxlength: 40`, trimmed.
  - `capacity`: `Number`, required, integer, `min: 1`, `max: 30`.
  - `floor`: required (`Number` or `String`).
  - `hasDrumKit`: `Boolean`, default `false`.
  - `status`: `String`, enum `['disponible', 'mantenimiento']`, default `'disponible'`.
  - `timestamps: true`.
- **Schema Index:** Unique index on `Room.name`.
- **Endpoints:** `POST /api/rooms` (returns `201`), `PUT /api/rooms/:id` (returns `200`, `400` on invalid input, `404` if not found, `409` on duplicate name).

---

## US3 - Browse, Search, and Remove Rehearsal Rooms

### Requirements
- As a user of the cultural center system, I want to look up a specific rehearsal room, browse through the list of all rooms in pages, sort them by different attributes, search for a room by name, and remove rooms that are no longer needed, provided they don't have upcoming active bookings.

### Acceptance Criteria
- I can view the full details of a single room by its identifier.
- When viewing all rooms, the list is broken into pages (10 items per page by default, up to a maximum of 50 per page) and shows the total number of matching rooms and total pages.
- I can sort the room list by name, capacity, floor, or creation date in ascending or descending order (sorted by name by default).
- I can search for rooms by typing part of their name, regardless of uppercase or lowercase letters.
- If I try to delete a room that currently has a reserved or confirmed booking, the system blocks the deletion and explains why.

### Technical implications
- **Endpoints:** `GET /api/rooms`, `GET /api/rooms/:id`, `DELETE /api/rooms/:id`.
- **Pagination & Sorting:** Accept `page` (default `1`), `limit` (default `10`, `1..50`, else `400`), `sort` (allowed: `name`, `capacity`, `floor`, `createdAt` with optional `-` prefix; default `name`; any other field returns `400`), and `search` (case-insensitive regex on `name`).
- **Response envelope:** `{ data, page, limit, total, pages }`.
- **Protected Delete (`DELETE /api/rooms/:id`):** Query `Booking.exists({ room: id, status: { $in: ['apartada', 'confirmada'] } })`. If true, return HTTP `409`.

---

## US4 - Check Daily Room Schedule

### Requirements
- As a cultural center coordinator, I want to pick a rehearsal room and a specific date to see all bookings scheduled for that day ordered from earliest to latest, including which band booked each time slot.

### Acceptance Criteria
- When I request a room's schedule for a given date, I receive the list of bookings for that room on that day sorted from the earliest start hour to the latest.
- Each entry in the daily schedule includes the details of the band that reserved the room.
- If the room does not exist, the system informs me that the room was not found.

### Technical implications
- **Endpoint:** `GET /api/rooms/:id/schedule?date=YYYY-MM-DD`.
- **Query & Index:** Uses the compound index `{ room: 1, date: 1, startHour: 1 }` on `Booking`.
- **Populate:** `.populate('band')` and `.sort({ startHour: 1 })`.
- Validate that room `:id` exists (`404` if not) and `date` query parameter is valid (`400` if missing/invalid).

---

## US5 - Register and Edit Musical Bands

### Requirements
- As a cultural center administrator, I want to register musical bands and update their information (name, musical style, contact person, and whether the band is currently active) so they can be assigned musicians and book rehearsal rooms.

### Acceptance Criteria
- A band can be registered with a unique name (2 to 50 characters) and a mandatory contact person's name.
- The band's musical style must be one of the supported categories (rock, jazz, choir/coral, or other); if not specified, it defaults to "other".
- New bands are marked as active by default.
- Two bands cannot have the same name.
- I can update a band's information or mark a band as inactive at any time.

### Technical implications
- **`Band` Mongoose Schema:**
  - `name`: `String`, required, unique, `minlength: 2`, `maxlength: 50`, trimmed.
  - `genre`: `String`, enum `['rock', 'jazz', 'coral', 'otro']`, default `'otro'`.
  - `active`: `Boolean`, default `true`.
  - `contactName`: `String`, required, trimmed.
  - `timestamps: true`.
- **Schema Index:** Unique index on `Band.name`.
- **Endpoints:** `POST /api/bands` (`201`), `PUT /api/bands/:id` (`200`, `400`, `404`, `409` on duplicate name).

---

## US6 - Browse, Search, and Remove Musical Bands

### Requirements
- As a user of the cultural center system, I want to browse all registered bands by page, search for a band by name, inspect a single band along with its member musicians, and delete a band only if it has no registered musicians and no active bookings.

### Acceptance Criteria
- Viewing a single band's profile automatically shows the list of musicians who belong to that band.
- Viewing the list of bands supports paging, searching by band name (case-insensitive), and sorting by name or registration date (sorted by name by default).
- If I attempt to delete a band that still has musicians registered to it, the system stops me and explains the conflict.
- If I attempt to delete a band that has a reserved or confirmed room booking, the system stops me and explains the conflict.

### Technical implications
- **Endpoints:** `GET /api/bands`, `GET /api/bands/:id`, `DELETE /api/bands/:id`.
- **Virtual Populate on `Band`:** Define virtual `musicians` (`ref: 'Musician'`, `localField: '_id'`, `foreignField: 'band'`) and populate it in `GET /api/bands/:id`.
- **Pagination & Sorting:** Allowed `sort` fields: `name`, `createdAt` (default `name`). `search` filters `name` via case-insensitive regex.
- **Protected Delete (`DELETE /api/bands/:id`):** Check `Musician.exists({ band: id })` and `Booking.exists({ band: id, status: { $in: ['apartada', 'confirmada'] } })`. Return HTTP `409` if either exists.

---

## US7 - View a Band's Musicians and Booking History

### Requirements
- As a cultural center coordinator, I want dedicated views to see all musicians belonging to a specific band and all rehearsal bookings made by that band (including which room each booking uses).

### Acceptance Criteria
- I can request the list of all musicians for a specific band.
- I can request the list of all bookings for a specific band, and each booking shows the information of the room where it takes place.
- If the band does not exist, the system informs me with a clear not-found message.

### Technical implications
- **Endpoints:**
  - `GET /api/bands/:id/musicians` — Queries `Musician.find({ band: id })` (uses `{ band: 1 }` index on `Musician`).
  - `GET /api/bands/:id/bookings` — Queries `Booking.find({ band: id }).populate('room')`.
- Verify `Band.findById(id)` exists first; return `404` if not found.

---

## US8 - Register, Edit, and Remove Musicians

### Requirements
- As a cultural center administrator, I want to add musicians to a band, update their personal details, instrument, band membership, or active status, and remove musicians when necessary.

### Acceptance Criteria
- Registering a musician requires their name, instrument, a unique email address, and the band they belong to.
- No two musicians can share the same email address.
- New musicians are marked as active by default.
- A musician cannot be assigned to a band that does not exist.
- I can update a musician's details (or mark them inactive) and delete a musician record at any time.

### Technical implications
- **`Musician` Mongoose Schema:**
  - `name`: `String`, required, trimmed.
  - `instrument`: `String`, required, trimmed.
  - `email`: `String`, required, unique, trimmed, lowercase.
  - `band`: `ObjectId` (`ref: 'Band'`), required.
  - `active`: `Boolean`, default `true`.
  - `timestamps: true`.
- **Schema Indexes:** Unique index on `Musician.email`; single-field index on `Musician.band` (`{ band: 1 }`).
- **Endpoints:** `POST /api/musicians` (`201`), `PUT /api/musicians/:id` (`200`), `DELETE /api/musicians/:id` (`200`). Verify referenced `band` exists (`404`/`400` if not) and handle duplicate `email` with `409`.

---

## US9 - Browse, Filter, and Search Musicians

### Requirements
- As a user of the cultural center system, I want to browse the directory of musicians in pages, filter them by their band or whether they are active, search for them by name or email, sort them, and view an individual musician's profile showing their band.

### Acceptance Criteria
- Viewing a single musician shows their personal details along with their band's information.
- When browsing all musicians, I can filter by a specific band, filter by active/inactive status, or combine both filters.
- Typing a search term matches musicians whose name OR email contains that text (ignoring uppercase/lowercase).
- I can sort musicians by name, instrument, or registration date (sorted by name by default).

### Technical implications
- **Endpoints:** `GET /api/musicians`, `GET /api/musicians/:id`.
- **Filters & Search:** Query params `band` (ObjectId), `active` (`true`/`false`), `search` (`$or: [{ name: regex }, { email: regex }]`).
- **Sorting:** Allowed `sort` fields: `name`, `instrument`, `createdAt` (default `name`).
- **Detail Populate:** `GET /api/musicians/:id` uses `.populate('band')`.

---

## US10 - Register and Update Equipment Inventory

### Requirements
- As an equipment manager, I want to add musical equipment (such as microphones, amplifiers, or percussion support) to the center's inventory and update item details or total stock without corrupting the count of units currently loaned out.

### Acceptance Criteria
- New equipment requires a unique name, a valid category (audio, strings/cuerdas, or percussion/percusion), and a total number of units greater than zero.
- When a new piece of equipment is added, all of its units start as available for borrowing.
- Updating an equipment record allows changing its name, category, or total units, but does not allow manually overriding the available units count directly.
- If I try to reduce an item's total units to a number smaller than the amount currently borrowed by bands, the system blocks the change with a conflict message.

### Technical implications
- **`Equipment` Mongoose Schema:**
  - `name`: `String`, required, unique, trimmed.
  - `category`: `String`, required, enum `['audio', 'cuerdas', 'percusion']`.
  - `totalUnits`: `Number`, required, integer, `min: 1`.
  - `availableUnits`: `Number`, integer, `min: 0`, initialized equal to `totalUnits`.
  - `timestamps: true`.
- **Schema Index:** Unique index on `Equipment.name`.
- **Endpoints:** `POST /api/equipment` (`201`), `PUT /api/equipment/:id` (`200`).
- **Inventory Rule on `PUT /api/equipment/:id`:** Ignore or reject direct modifications to `availableUnits`. Compute `loanedUnits = equipment.totalUnits - equipment.availableUnits`. If `newTotalUnits < loanedUnits`, return HTTP `409`. Otherwise update `totalUnits = newTotalUnits` and `availableUnits = newTotalUnits - loanedUnits`.

---

## US11 - Browse, Filter, Search, and Remove Equipment

### Requirements
- As a user of the cultural center system, I want to view the catalog of musical equipment, filter by category or by what is currently in stock, search by equipment name, sort the catalog, and delete equipment only if none of its units are currently loaned out.

### Acceptance Criteria
- I can filter the equipment list by category (audio, strings, percussion) and filter to show only items that have at least one unit available right now.
- I can search equipment by name (case-insensitive) and sort by name, available units, or creation date (sorted by name by default).
- If I try to delete an equipment item while one or more of its units are currently borrowed by a band, the system rejects the deletion with a conflict message.

### Technical implications
- **Endpoints:** `GET /api/equipment`, `GET /api/equipment/:id`, `DELETE /api/equipment/:id`.
- **Filters & Sorting:** Query params `category` and `available=true` (`availableUnits: { $gt: 0 }`). Allowed `sort` fields: `name`, `availableUnits`, `createdAt` (default `name`). `search` matches `name`.
- **Protected Delete (`DELETE /api/equipment/:id`):** If `equipment.availableUnits < equipment.totalUnits`, return HTTP `409`.

---

## US12 - Create a Room Booking with Equipment Loans

### Requirements
- As a band representative, I want to reserve an available rehearsal room for today or a future date during operating hours (8:00 AM to 10:00 PM) for up to 3 hours, and optionally request musical equipment in the same step, ensuring that if anything goes wrong, neither the room nor the equipment is partially reserved.

### Acceptance Criteria
- The booking starts between 8:00 and 21:00, ends between 9:00 and 22:00, has an end hour strictly after the start hour, and lasts at most 3 hours.
- The booking date must be today or in the future, and optional notes cannot exceed 200 characters.
- The requested room must exist and be marked as available (not under maintenance).
- The requesting band must exist, be active, have at least one active musician, and its number of active musicians must fit within the room's maximum capacity.
- The room cannot already have a reserved or confirmed booking on that date whose hours overlap with the requested time slot (canceled bookings do not block the room).
- I can optionally request one or more equipment items; the list cannot repeat the same equipment item twice, and each requested quantity must be available in inventory.
- When the booking succeeds, it starts in the "reserved" (`apartada`) state, shows its calculated duration in hours, creates a loan record for each requested equipment item, and deducts those units from available inventory—all as a single all-or-nothing operation.

### Technical implications
- **`Booking` Mongoose Schema:**
  - `room`: `ObjectId` (`ref: 'Room'`), required.
  - `band`: `ObjectId` (`ref: 'Band'`), required.
  - `date`: `String` (`YYYY-MM-DD`) or normalized `Date`, required.
  - `startHour`: `Number`, integer `8..21`, required.
  - `endHour`: `Number`, integer `9..22`, required.
  - `status`: `String`, enum `['apartada', 'confirmada', 'cancelada']`, default `'apartada'`.
  - `notes`: `String`, optional, `maxlength: 200`.
  - Virtual `durationHours`: returns `this.endHour - this.startHour` (`toJSON: { virtuals: true }`).
  - Compound Index: `{ room: 1, date: 1, startHour: 1 }`.
- **Transaction (`POST /api/bookings`):**
  - Open session with `mongoose.startSession()` and execute inside `session.withTransaction(async () => { ... })`.
  - Check overlap: `Booking.findOne({ room, date, status: { $in: ['apartada', 'confirmada'] }, startHour: { $lt: endHour }, endHour: { $gt: startHour } }).session(session)`.
  - For each `{ equipmentId, units }` in `req.body.equipment`: verify no duplicate `equipmentId`, verify `availableUnits >= units`, update inventory with `Equipment.updateOne({ _id: equipmentId, availableUnits: { $gte: units } }, { $inc: { availableUnits: -units } }, { session })`, and create `Loan` (`{ booking, equipment: equipmentId, units, status: 'prestado' }`).
  - Return `409` on any business rule conflict (overlap, maintenance room, inactive band, 0 active musicians, capacity exceeded, insufficient equipment stock).

---

## US13 - Update a Booking or Confirm Attendance

### Requirements
- As a cultural center coordinator, I want to modify an existing booking's room, band, date, hours, or notes, or change its status from reserved to confirmed, while making sure all scheduling and capacity rules are still respected.

### Acceptance Criteria
- When changing the room, band, date, or hours of a booking, the system re-checks all scheduling rules (operating hours, max 3 hours, today/future date, room availability, active band with active musicians fitting capacity, and no schedule overlap with *other* bookings).
- When checking for schedule overlaps during an edit, the booking does not conflict with itself.
- A reserved booking can be changed to confirmed (or canceled); a confirmed booking can only be changed to canceled; a canceled booking cannot be changed to any other status.
- Confirming a booking does not alter equipment inventory because the equipment was already set aside when the booking was created.
- Equipment items cannot be added or modified when editing an existing booking.

### Technical implications
- **Endpoint:** `PUT /api/bookings/:id`.
- Exclude self in overlap query: `_id: { $ne: booking._id }`.
- Enforce state machine transitions:
  - `'apartada'` $\rightarrow$ `'confirmada'` | `'cancelada'`
  - `'confirmada'` $\rightarrow$ `'cancelada'`
  - `'cancelada'` $\rightarrow$ none (reject transitions with `409`).
- If `PUT` transitions status to `'cancelada'`, execute the cancellation transaction so loans are marked `'devuelto'` and `Equipment.availableUnits` are restored via `$inc`.

---

## US14 - Cancel a Booking and Return Borrowed Equipment

### Requirements
- As a band representative or coordinator, I want to cancel a reserved or confirmed booking so that the room becomes free for other bands and any borrowed equipment is automatically returned to the center's available inventory at the exact same moment.

### Acceptance Criteria
- Canceling a reserved or confirmed booking changes its status to canceled.
- All equipment loans linked to that booking are marked as returned, and their quantities are added back to the available inventory in a single all-or-nothing operation.
- Once canceled, the time slot in that room is immediately available for another band to book.
- Attempting to cancel a booking that is already canceled is rejected with a conflict message.

### Technical implications
- **Endpoint:** `PATCH /api/bookings/:id/cancel`.
- **Transaction:** Uses `mongoose.startSession()` and `session.withTransaction()`:
  1. Fetch `Booking` with `{ session }`; if not found return `404`; if `status === 'cancelada'` return `409`.
  2. Set `booking.status = 'cancelada'` and save with `{ session }`.
  3. Find all `Loan` documents for `{ booking: booking._id, status: 'prestado' }` with `{ session }`.
  4. For each loan, update `loan.status = 'devuelto'` and run `Equipment.updateOne({ _id: loan.equipment }, { $inc: { availableUnits: loan.units } }, { session })`.

---

## US15 - Browse, Filter, Search, and Inspect Bookings

### Requirements
- As a user of the cultural center system, I want to search and filter rehearsal bookings by status, room, band, specific date, or date range, sort them chronologically, and inspect a single booking's complete details including its room, band with active musicians, and borrowed equipment.

### Acceptance Criteria
- When browsing bookings, I can filter by booking status, room, band, an exact date, or a start/end date range (`from` and `to`).
- I can search bookings by text inside their notes (ignoring uppercase/lowercase) and sort by date, start hour, or creation date (sorted by date and then start hour by default).
- When viewing a single booking's detail page, I see:
  - The room's name, capacity, and status.
  - The band's name, genre, and its list of active musicians.
  - All equipment loans for that booking, with the details of each piece of equipment.
  - The booking's calculated duration in hours.

### Technical implications
- **Endpoints:** `GET /api/bookings`, `GET /api/bookings/:id`.
- **Filters:** `status`, `room`, `band`, `date`, `from` (`date >= from`), `to` (`date <= to`), `search` (case-insensitive regex on `notes`).
- **Sorting:** Allowed `sort` fields: `date`, `startHour`, `createdAt`. Default sort: `{ date: 1, startHour: 1 }`.
- **Detailed Populate (`GET /api/bookings/:id`):**
  - Populate `room` selecting `name capacity status`.
  - Populate `band` selecting `name genre` with nested populate on `musicians` filtered by `{ active: true }`.
  - Populate `loans` (virtual or query on `Loan.find({ booking: id }).populate('equipment')`).

---

## US16 - Browse, Filter, Search, and Inspect Equipment Loans

### Requirements
- As an equipment manager, I want to view all equipment loan records, filter them by whether they are currently borrowed or returned, filter by booking or equipment item, and inspect individual loan details.

### Acceptance Criteria
- Loan records cannot be created manually on their own; they only originate when a booking is created.
- When browsing loans, I can filter by loan status (borrowed or returned), by booking, or by equipment item.
- I can search loans by status and sort them by number of units or creation date (sorted by creation date by default).
- Viewing a single loan shows both the booking information and the equipment information.

### Technical implications
- **`Loan` Mongoose Schema:**
  - `booking`: `ObjectId` (`ref: 'Booking'`), required.
  - `equipment`: `ObjectId` (`ref: 'Equipment'`), required.
  - `units`: `Number`, integer, `min: 1`, required.
  - `status`: `String`, enum `['prestado', 'devuelto']`, required.
  - `timestamps: true`.
- **Schema Indexes:** `{ booking: 1 }` and compound `{ equipment: 1, status: 1 }`.
- **Endpoints:** `GET /api/loans` and `GET /api/loans/:id` (with `.populate('booking').populate('equipment')`). No `POST /api/loans` endpoint.
- **Sorting & Search:** Allowed `sort` fields: `units`, `createdAt` (default `createdAt`). `search` matches `status` (case-insensitive).

---

## US17 - View Cultural Center Summary Statistics

### Requirements
- As a cultural center director, I want quick summary reports showing how many bookings exist in each status, how many active bookings each rehearsal room has, and how many total vs. available equipment units exist in each equipment category.

### Acceptance Criteria
- The booking statistics report shows the total count of bookings grouped by status (reserved, confirmed, canceled).
- The room statistics report lists each room along with how many active bookings (reserved or confirmed) it currently has.
- The equipment statistics report lists each equipment category (audio, strings, percussion) with the sum of total units and the sum of available units.

### Technical implications
- Must be implemented using MongoDB **aggregation pipelines** (`Model.aggregate([...])`):
  1. `GET /api/bookings/stats`: `$group` by `$status` with `{ $sum: 1 }`.
  2. `GET /api/rooms/stats`: Aggregate from `Room` with `$lookup` on `bookings` filtering `status: { $in: ['apartada', 'confirmada'] }` (or `$group` from `Booking` joined with `Room`) to return active booking counts per room.
  3. `GET /api/equipment/stats`: `$group` by `$category` summing `$totalUnits` and `$availableUnits`.
- **Route Priority:** Register `/stats` routes before `/:id` routes in Express routers.

---

## US18 - Unified Error Handling and Input Validation

### Requirements
- As an API consumer, I want clear and predictable feedback whenever I send incomplete or invalid information, look up an item that does not exist, or break a cultural center rule.

### Acceptance Criteria
- Sending malformed data, missing required fields, an invalid ID format, or invalid pagination/sorting options returns a "Bad Request" response (`400`).
- Requesting, updating, or deleting an item whose ID does not exist in the system returns a "Not Found" response (`404`).
- Breaking any business rule (such as duplicate names/emails, room overlaps, exceeding room capacity, booking a room under maintenance, insufficient equipment, invalid status changes, or deleting protected records) returns a "Conflict" response (`409`).
- Every error response follows the exact same structure with a clear message explaining what went wrong.

### Technical implications
- Central Express error-handling middleware normalizing responses to `{ "error": "mensaje claro" }`:
  - Mongoose `ValidationError` / `CastError` / invalid query params $\rightarrow$ HTTP `400`.
  - Missing resource $\rightarrow$ HTTP `404`.
  - MongoDB duplicate key error (`code === 11000`) or domain rule conflict $\rightarrow$ HTTP `409`.

---

## US19 - Postman Testing Suite and Project Documentation

### Requirements
- As a team member and evaluator, I want a ready-to-run Postman collection that demonstrates every feature, query filter, and error case in order, alongside a clear project guide explaining how to install, run, and understand the system's indexes and transactions.

### Acceptance Criteria
- The Postman collection is named **Cabina**, describes the API and folder execution order (`salas`, `bandas`, `músicos`, `equipo`, `reservas`, `consultas`, `errores`), and uses an environment file with `baseUrl`, `roomId`, `bandId`, `musicianId`, `equipmentId`, `bookingId`, and `loanId`.
- Every request has a descriptive name, expected status code in its description, and a saved example response.
- Creation requests automatically save the created IDs into environment variables, and happy-path requests include automated status code tests.
- The `consultas` folder demonstrates page 1, page 2, ascending sort, descending sort, search with results, search without results across all 6 listings, plus room schedule, band musicians, band bookings, and the 3 stats endpoints.
- The `errores` folder demonstrates all 13 required error scenarios (duplicate name, duplicate email, inverted hours, >3h booking, maintenance room, inactive band, band exceeding capacity, overlap, insufficient equipment, invalid status transition, deleting room with active booking, deleting loaned equipment, and non-existent ID).
- The `README.md` includes team members, installation steps, `.env` instructions, `npm run dev`, the **Índices** section (explaining which query each index serves), and the **Transacción** section (explaining functions, collections touched, and how `409` rollbacks were tested).

### Technical implications
- Export `Cabina.postman_collection.json` and `Cabina.postman_environment.json` into the repository.
- Maintain comprehensive `README.md` satisfying all academic defense requirements.
