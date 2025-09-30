# Backend Feature Requirements – Airbnb Clone

This document provides technical and functional requirement specifications for three core backend features: **User Authentication**, **Property Management**, and **Booking System**. Each section includes API endpoints, request/response schemas, validation rules, and error handling.

---

## 1. User Authentication

### 1.1 Overview

Provide secure user registration, login, token refresh, logout, and optional OAuth. Use JWT for authentication (short-lived access token + refresh token). Support roles: `guest`, `host`, `admin`.

### 1.2 Endpoints

- `POST /api/v1/auth/register`

  - Description: Register a new user (guest or host).
  - Request body (JSON):
    ```json
    {
      "first_name": "John",
      "last_name": "Doe",
      "email": "john@example.com",
      "password": "P@ssw0rd!",
      "phone_number": "+2349011111111",
      "role": "guest"
    }
    ```
  - Response (201 Created):
    ```json
    {
      "user_id": "uuid",
      "email": "john@example.com",
      "role": "guest",
      "created_at": "2025-09-30T12:00:00Z"
    }
    ```
  - Validation:
    - `email` must be a valid email format and unique.
    - `password` minimum 8 chars, include uppercase, lowercase, number, special char.
    - `role` must be one of `guest`, `host`, `admin` (admin registration restricted).
  - Errors:
    - 400 Bad Request for validation errors.
    - 409 Conflict if email already exists.

- `POST /api/v1/auth/login`

  - Description: Authenticate user and return JWT pair (access + refresh).
  - Request body (JSON):
    ```json
    {
      "email": "john@example.com",
      "password": "P@ssw0rd!"
    }
    ```
  - Response (200 OK):
    ```json
    {
      "access_token": "jwt.access.token",
      "refresh_token": "jwt.refresh.token",
      "expires_in": 900
    }
    ```
  - Validation/Errors:
    - 401 Unauthorized for bad credentials.
    - 429 Too Many Requests if brute-force detected.

- `POST /api/v1/auth/refresh`

  - Description: Exchange refresh token for new access token.
  - Request body:
    ```json
    { "refresh_token": "jwt.refresh.token" }
    ```
  - Response (200 OK):
    ```json
    { "access_token": "new.jwt.access.token", "expires_in": 900 }
    ```
  - Errors: 401 for invalid/expired refresh token.

- `POST /api/v1/auth/logout`

  - Description: Revoke refresh token (logout).
  - Auth: Requires valid access token.
  - Response: 204 No Content.

- Optional OAuth endpoints for Google/Facebook using OAuth2 flows:
  - `POST /api/v1/auth/oauth/google` (exchange provider token for app JWT).

---

## 2. Property Management

### 2.1 Overview

Hosts should be able to create, update, delete, and view property listings. Listings include metadata (title, description, location: city/state/country), price_per_night, amenities, images (stored in cloud or local storage), and availability calendar.

### 2.2 Endpoints

- `POST /api/v1/properties`

  - Auth: Host or Admin (RBAC)
  - Request body (JSON):
    ```json
    {
      "name": "Cozy Apartment in Lagos",
      "description": "A modern 2-bedroom apartment near the beach.",
      "location": { "city": "Lagos", "state": "Lagos", "country": "Nigeria" },
      "price_per_night": 150.0,
      "amenities": ["wifi", "air_conditioning", "kitchen"],
      "images": ["https://cdn.example.com/properties/1/1.jpg"],
      "max_guests": 4
    }
    ```
  - Response (201 Created):
    ```json
    {
      "property_id": "uuid",
      "host_id": "uuid",
      "created_at": "2025-08-29T12:34:00Z"
    }
    ```
  - Validation:
    - `name`, `description`, `location`, `price_per_night` required.
    - `price_per_night` must be >= 0.01 (currency validated separately).
    - `amenities` must be a list of predefined strings if normalized, or freeform (JSONB) if not.
  - Image uploads via separate endpoints (signed URLs) are recommended.

- `GET /api/v1/properties`

  - Description: List properties with filters & pagination.
  - Query parameters: `location`, `price_min`, `price_max`, `guests`, `amenities`, `available_start`, `available_end`, `page`, `page_size`, `sort`.
  - Response (200 OK):
    ```json
    {
      "results": [ { "property_id": "uuid", "name": "...", "price_per_night": 150.0, "location": {...} } ],
      "page": 1,
      "page_size": 20,
      "total": 1234
    }
    ```
  - Performance:
    - Use indexed columns for filtering (location, price).
    - Cache frequent queries (Redis) with cache invalidation on property updates.
    - p95 latency for cached reads < 200ms; uncached < 700ms.

- `GET /api/v1/properties/{id}`

  - Response includes property details, host summary (id, name), average rating, and availability calendar (or a lightweight availability summary).

- `PATCH /api/v1/properties/{id}`

  - Auth: Host (owner) or Admin.
  - Partial update; same validation rules as create.

- `DELETE /api/v1/properties/{id}`
  - Auth: Host (owner) or Admin.
  - Soft-delete recommended (set `deleted_at`) to preserve history and bookings.

### 2.3 Availability & Calendar

- Availability should be derived from bookings.
- Provide an endpoint `GET /api/v1/properties/{id}/availability?start=&end=` that checks if any confirmed/pending booking overlaps the range and returns available dates or a boolean.
- Prevent race conditions with DB-level exclusion constraints (Postgres `EXCLUDE USING GIST` on daterange) and/or transactional checks + optimistic locking.

### 2.4 File Uploads (Images)

- Use signed upload URLs (PUT to S3 or Cloudinary direct upload) to avoid passing files through API servers.
- Validate: file type (jpeg/png), max size (e.g., 10MB), image dimensions if required.
- Store image metadata (url, width, height, thumbnail_url) in `property_images` table or JSONB field.

### 2.5 Security & Permissions

- Only owners (host_id) may edit/delete their properties.
- Admins can moderate listings (suspend/hide). Actions must be logged.

---

## 3. Booking System

### 3.1 Overview

Guests book properties for date ranges. The system must prevent double bookings, support booking lifecycle (pending -> confirmed -> completed/canceled), integrate payments, and enable hosts to confirm bookings if required by policy.

### 3.2 Endpoints

- `POST /api/v1/bookings`

  - Auth: Guest (and optionally host/admin)
  - Request body (JSON):
    ```json
    {
      "property_id": "uuid",
      "start_date": "2025-09-01",
      "end_date": "2025-09-05",
      "guests": 2
    }
    ```
  - Process:
    1. Validate dates (`end_date > start_date`, both in future or according to policy).
    2. Check property exists and `max_guests` >= `guests`.
    3. Check availability: ensure no existing booking with status in (pending, confirmed) overlaps the requested range.
    4. If available, create a `pending` booking record and calculate `total_price` on the fly as `(nights * price_per_night)`.
    5. Return booking details and payment instructions (amount, currency, payment intent id if using Stripe).
  - Response (201 Created):
    ```json
    {
      "booking_id": "uuid",
      "status": "pending",
      "nights": 4,
      "price_per_night": 150.0,
      "amount": 600.0
    }
    ```
  - Errors:
    - 400 Bad Request for validation errors.
    - 404 Not Found if property doesn't exist.
    - 409 Conflict if date overlap detected.
  - Concurrency controls:
    - Use DB-level exclusion constraints (Postgres) or transactional row locks. If using denormalized `total_price`, compute and store atomically.

- `GET /api/v1/bookings/{id}`

  - Auth: booking owner (guest), property host, or admin.
  - Response: booking record, payment status, related host/property summary.

- `GET /api/v1/bookings`

  - List bookings; support filters by `user_id`, `property_id`, `status`, `date_range`.
  - Pagination and sorting.

- `PATCH /api/v1/bookings/{id}`

  - Auth: booking owner (for cancellation) or host/admin (per policy).
  - Allowed transitions: `pending -> canceled`, `pending -> confirmed` (host), `confirmed -> canceled` (refund policy), `confirmed -> completed` (system job after end_date).
  - Return updated booking.

- `POST /api/v1/bookings/{id}/confirm`
  - Auth: Host or Admin.
  - Marks `pending` booking as `confirmed` if policy allows.

### 3.3 Payments Integration

- Integrate with Stripe/PayPal.
- Flow:
  1. Booking created in `pending` status.
  2. Server creates a payment intent with provider and returns client secret or redirect URL.
  3. Client completes payment; provider webhook notifies server of success/failure.
  4. On success: record payment, mark booking `confirmed` (or keep `pending` if host confirmation required), schedule host payout per payout rules.
  5. On failure: mark payment failed and keep booking `pending` or `canceled` per policy.
- Endpoints:
  - `POST /api/v1/bookings/{id}/payments` — create payment intent / record payment.
  - `POST /api/v1/webhooks/payments` — webhook receiver (idempotent handling required).

### 3.4 Validations & Business Rules

- `end_date > start_date` and `start_date >= today` (policy-dependent).
- Bookings may have minimum/maximum stay rules per property.
- Prevent overlapping bookings with status in (`pending`, `confirmed`) for same property.
- Reviews can only be created by guests who have a completed booking for that property.

### 3.5 Performance & Scaling

- Booking creation includes availability checks; ensure p95 of booking creation < 700ms.
- Use caching for read-heavy queries (property details, availability summaries) but ensure writes (new bookings) invalidate caches appropriately.
- For high concurrency, consider queuing payment-intent creation and using optimistic locking or DB exclusion constraints to ensure correctness.

### 3.6 Error Handling & Idempotency

- Payment webhooks must be idempotent: check for existing payment records before creating duplicates.
- Booking creation API should be resilient to replay — use client-generated idempotency keys for retries.

---

## Appendix: Common Error Responses

- `400 Bad Request` — validation error. Return JSON `{ "errors": { "field": "message" } }`.
- `401 Unauthorized` — missing/invalid token.
- `403 Forbidden` — insufficient permissions (RBAC).
- `404 Not Found` — resource not found.
- `409 Conflict` — business conflict (date overlap, unique constraint).
- `422 Unprocessable Entity` — semantic errors.
- `500 Internal Server Error` — unexpected server error (log correlation id).

---

## Change Log

- v1.0 — Initial detailed requirements authored (2025-08-29).
