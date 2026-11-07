# Restaurant Reservation Service: API Contracts

| | |
|---|---|
| **Author** | Hadil Trabelsi |
| **Date** | 2026-11-06 |
| **Status** | Draft for team review |
| **Related** | [Requirements and invariants](requirements-and-invariants.md) · [Data model](data-model.md) |

## 1. Conventions

| Topic | Convention |
|---|---|
| Base path | `/api/v1` |
| Format | JSON, UTF-8. `Content-Type: application/json` on requests with a body. |
| Date | `YYYY-MM-DD`, in the restaurant's time zone (Africa/Tunis) |
| Date and time | ISO 8601 with an explicit offset, for example `2026-11-20T19:00:00+01:00`. A value without an offset is rejected. Responses use the restaurant's offset. |
| Guest credentials | Header `X-Guest-Phone` with the phone number used at booking. The reference code is in the path. |
| Staff credentials | Header `Authorization: Bearer <token>` |
| Errors | One JSON envelope for all errors (section 5) |

Credentials are part of the contract now. Enforcement and permission-denial behavior are implemented in Phase 2.

## 2. Endpoints

| # | Method and path | Purpose | Access | Delivered |
|---|---|---|---|---|
| 1 | `GET /restaurant` | Opening hours and booking rules | Public | Phase 1 slice |
| 2 | `GET /availability` | Free slots for a date and party size | Public | Phase 1 slice |
| 3 | `POST /reservations` | Create a reservation | Public | Phase 2 |
| 4 | `GET /reservations/{reference_code}` | View a reservation | Guest (own), Staff | Phase 2 |
| 5 | `PATCH /reservations/{reference_code}` | Modify a reservation | Guest (own), Staff | Phase 2 |
| 6 | `POST /reservations/{reference_code}/cancel` | Cancel a reservation | Guest (own), Staff | Phase 2 |
| 7 | `GET /reservations` | List reservations for a date | Staff | Phase 2 |

Cancel is a `POST` action and not a `DELETE`, because the reservation is kept with status `cancelled`.

## 3. Resources

### Reservation

| Field | Type | Notes |
|---|---|---|
| `reference_code` | string | For example `RSV-7K3F9Q` |
| `status` | string | `confirmed` or `cancelled` |
| `guest_name` | string | 2–80 characters |
| `guest_phone` | string | International format |
| `party_size` | integer | 1–6 |
| `starts_at` | string | ISO 8601 with offset |
| `ends_at` | string | `starts_at` plus 90 minutes |
| `special_request` | string or null | Up to 200 characters |
| `created_at` | string | ISO 8601 |
| `updated_at` | string | ISO 8601 |
| `cancelled_at` | string or null | Set only when cancelled |

Staff responses also include `table_label` (for example `T05`). Guest responses do not.

```json
{
  "reference_code": "RSV-7K3F9Q",
  "status": "confirmed",
  "guest_name": "Amira Ben Salah",
  "guest_phone": "+21620000001",
  "party_size": 4,
  "starts_at": "2026-11-20T19:00:00+01:00",
  "ends_at": "2026-11-20T20:30:00+01:00",
  "special_request": null,
  "created_at": "2026-11-06T21:15:00+01:00",
  "updated_at": "2026-11-06T21:15:00+01:00",
  "cancelled_at": null
}
```

## 4. Endpoint details

### 4.1 `GET /restaurant`

Returns the opening hours and the booking rules, so clients do not hard-code them. Days that are not listed are closed.

**Response `200`**

```json
{
  "name": "Le Jasmin",
  "timezone": "Africa/Tunis",
  "opening_hours": [
    { "day": "tuesday", "opens_at": "12:00", "closes_at": "23:00" },
    { "day": "wednesday", "opens_at": "12:00", "closes_at": "23:00" },
    { "day": "thursday", "opens_at": "12:00", "closes_at": "23:00" },
    { "day": "friday", "opens_at": "12:00", "closes_at": "23:00" },
    { "day": "saturday", "opens_at": "12:00", "closes_at": "23:00" },
    { "day": "sunday", "opens_at": "12:00", "closes_at": "23:00" }
  ],
  "closed_dates": [],
  "reservation_rules": {
    "slot_minutes": 30,
    "duration_minutes": 90,
    "min_party_size": 1,
    "max_party_size": 6,
    "min_lead_minutes": 120,
    "max_advance_days": 60
  }
}
```

`closed_dates` lists upcoming closed dates as `{ "date": "YYYY-MM-DD", "reason": "..." }`.

### 4.2 `GET /availability`

| Query parameter | Required | Rule |
|---|:-:|---|
| `date` | Yes | `YYYY-MM-DD` |
| `party_size` | Yes | Integer, 1 or more. 7 or more returns `PARTY_SIZE_UNSUPPORTED`. |

**Response `200`** (slot list shortened)

```json
{
  "date": "2026-11-20",
  "party_size": 4,
  "timezone": "Africa/Tunis",
  "open": true,
  "closed_reason": null,
  "slots": [
    { "starts_at": "2026-11-20T12:00:00+01:00", "ends_at": "2026-11-20T13:30:00+01:00" },
    { "starts_at": "2026-11-20T12:30:00+01:00", "ends_at": "2026-11-20T14:00:00+01:00" }
  ]
}
```

- A closed day is a valid question. It returns `200` with `"open": false`, `"slots": []`, and `closed_reason` set to `weekly_closure` or `closed_date`.
- An open day that is fully booked returns `"open": true` and `"slots": []`.
- For today, slots starting less than 2 hours from now are left out.
- Table identifiers are never exposed.

**Errors:** `VALIDATION_ERROR` (422), `PARTY_SIZE_UNSUPPORTED` (422), `OUTSIDE_BOOKING_WINDOW` (422).

### 4.3 `POST /reservations`

**Request**

| Field | Required | Rule |
|---|:-:|---|
| `guest_name` | Yes | 2–80 characters |
| `guest_phone` | Yes | International format, for example `+21620000001` |
| `party_size` | Yes | Integer, 1 or more. 7 or more returns `PARTY_SIZE_UNSUPPORTED`. |
| `starts_at` | Yes | ISO 8601 with offset, on a 30-minute slot |
| `special_request` | No | Up to 200 characters |

```json
{
  "guest_name": "Amira Ben Salah",
  "guest_phone": "+21620000001",
  "party_size": 4,
  "starts_at": "2026-11-20T19:00:00+01:00",
  "special_request": null
}
```

**Response `201`** with header `Location: /api/v1/reservations/RSV-7K3F9Q` and the Reservation resource as the body.

**Checks run in this order**, and the first failure is returned:

1. Request validation: `VALIDATION_ERROR`
2. Party size above the limit: `PARTY_SIZE_UNSUPPORTED`
3. Booking window: `OUTSIDE_BOOKING_WINDOW`
4. Opening hours, closed days, slot grid: `RESTAURANT_CLOSED`
5. Table availability: `SLOT_UNAVAILABLE`

A start time that is not on the 30-minute grid is a `VALIDATION_ERROR` (`not_on_slot`). A time on the grid but outside opening hours is `RESTAURANT_CLOSED`.

### 4.4 `GET /reservations/{reference_code}`

Requires `X-Guest-Phone`, or staff credentials.

**Response `200`:** the Reservation resource.
**Errors:** `UNAUTHENTICATED` (401), `RESERVATION_NOT_FOUND` (404). A wrong phone number returns the same `404` as an unknown code.

### 4.5 `PATCH /reservations/{reference_code}`

Requires `X-Guest-Phone`, or staff credentials. Send only the fields to change. At least one is required.

| Field | Rule |
|---|---|
| `starts_at` | As in 4.3 |
| `party_size` | As in 4.3 |
| `special_request` | Up to 200 characters. `null` clears it. |

`guest_name` and `guest_phone` cannot be changed. Sending them returns `VALIDATION_ERROR`.

**Response `200`:** the updated Reservation resource.

**Checks run in this order:**

1. Request validation: `VALIDATION_ERROR`
2. Reservation lookup and ownership: `RESERVATION_NOT_FOUND`
3. Reservation already cancelled: `RESERVATION_CANCELLED`
4. Starts in 2 hours or less: `MODIFICATION_TOO_LATE`
5. The same checks as creation, in the same order: `PARTY_SIZE_UNSUPPORTED`, `OUTSIDE_BOOKING_WINDOW`, `RESTAURANT_CLOSED`, `SLOT_UNAVAILABLE`

The update is atomic. On any error the reservation is unchanged. The reservation's own current slot does not count as a conflict with itself.

### 4.6 `POST /reservations/{reference_code}/cancel`

Requires `X-Guest-Phone`, or staff credentials. No request body.

**Response `200`:** the Reservation resource with `status` set to `cancelled` and `cancelled_at` filled in.

- Cancelling an already cancelled reservation returns `200` with the same resource and changes nothing.
- A reservation that has already started returns `RESERVATION_STARTED` (409).

**Errors:** `UNAUTHENTICATED` (401), `RESERVATION_NOT_FOUND` (404), `RESERVATION_STARTED` (409).

### 4.7 `GET /reservations`

Staff only. Lists the reservations of one day, ordered by `starts_at`.

| Query parameter | Required | Rule |
|---|:-:|---|
| `date` | Yes | `YYYY-MM-DD` |
| `status` | No | `confirmed` or `cancelled`. Both are returned when omitted. |

**Response `200`**

```json
{
  "date": "2026-11-20",
  "reservations": [
    {
      "reference_code": "RSV-7K3F9Q",
      "status": "confirmed",
      "table_label": "T05",
      "guest_name": "Amira Ben Salah",
      "guest_phone": "+21620000001",
      "party_size": 4,
      "starts_at": "2026-11-20T19:00:00+01:00",
      "ends_at": "2026-11-20T20:30:00+01:00",
      "special_request": null,
      "created_at": "2026-11-06T21:15:00+01:00",
      "updated_at": "2026-11-06T21:15:00+01:00",
      "cancelled_at": null
    }
  ]
}
```

The list is limited to one day, so it is not paginated.

**Errors:** `UNAUTHENTICATED` (401), `FORBIDDEN` (403), `VALIDATION_ERROR` (422).

## 5. Error format

Every error uses the same body:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request is not valid.",
    "details": [
      { "field": "party_size", "issue": "required" }
    ]
  }
}
```

| Field | Description |
|---|---|
| `code` | Stable identifier for programs, including the agent. Clients branch on this. |
| `message` | Short readable text. It may change and is not meant for parsing. |
| `details` | List of `{ field, issue }`. Present for `VALIDATION_ERROR`, otherwise empty. |

Issue values: `required`, `invalid_format`, `out_of_range`, `not_on_slot`, `not_allowed`.

The service converts the web framework's own validation errors to this format, so clients see one shape.

## 6. Error codes

| Code | HTTP | Meaning |
|---|:-:|---|
| `VALIDATION_ERROR` | 422 | A field is missing or invalid. `details` lists each one. |
| `PARTY_SIZE_UNSUPPORTED` | 422 | Party is larger than the online limit of 6. |
| `OUTSIDE_BOOKING_WINDOW` | 422 | Start is in the past, less than 2 hours ahead, or more than 60 days ahead. |
| `RESTAURANT_CLOSED` | 422 | Closed day, closed date, or time outside opening hours. |
| `SLOT_UNAVAILABLE` | 409 | No table is free for the requested time and party size. |
| `RESERVATION_CANCELLED` | 409 | The reservation is cancelled and cannot be modified. |
| `MODIFICATION_TOO_LATE` | 409 | The reservation starts in 2 hours or less. |
| `RESERVATION_STARTED` | 409 | The reservation has already started and cannot be cancelled. |
| `UNAUTHENTICATED` | 401 | Credentials are missing. |
| `FORBIDDEN` | 403 | The credentials are valid but do not allow this action. |
| `RESERVATION_NOT_FOUND` | 404 | Unknown code, or the phone number does not match. |
| `SERVICE_UNAVAILABLE` | 503 | A dependency such as the database is unavailable. Nothing was changed. Retry is safe. |
| `INTERNAL_ERROR` | 500 | Unexpected failure. |

## 7. Not yet defined

Duplicate-request handling (INV-10, failure case F12) is designed in Phase 2. It may add an idempotency mechanism to `POST /reservations`, and this document will be updated then. The resources and error codes above do not change.
