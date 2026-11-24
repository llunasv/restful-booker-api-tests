# Restful-Booker bug reports

These reports distinguish observed behavior from the desired behavior used by
this portfolio project. The demo API intentionally has unusual behavior. In
particular, BUG-002, BUG-004, and BUG-005 record project design expectations;
they are not claims that its published contract explicitly promises those rules.

Main-folder tests characterize the observed behavior. The `Known bugs` folder
asserts the desired behavior and is expected to fail while these issues remain.
It must be excluded from the normal CI run.

| ID | Title | Severity | Priority |
| --- | --- | --- | --- |
| [BUG-001](#bug-001-delete-returns-a-creation-status) | DELETE returns a creation status | Low | P3 |
| [BUG-002](#bug-002-incorrect-credentials-return-a-success-status) | Incorrect credentials return a success status | Low | P3 |
| [BUG-003](#bug-003-missing-required-booking-fields-return-500) | Missing required booking fields return 500 | Medium | P2 |
| [BUG-004](#bug-004-negative-totalprice-is-accepted) | Negative totalprice is accepted | Medium | P2 |
| [BUG-005](#bug-005-checkout-before-checkin-is-accepted) | Checkout before checkin is accepted | Medium | P2 |

## Common setup

- Base URL: `https://restful-booker.herokuapp.com`.
- JSON requests use `Content-Type: application/json`.
- Run `Auth / Create token` before any authenticated cleanup or deletion.
- Use the public demo credentials `admin` / `password123`.
- Booking IDs come from POST responses; no existing public bookings are targeted.
- Accepted validation bookings are deleted by the following cleanup request.
  Cleanup skips its HTTP request if the server rejects the input without creating
  a booking. Failed assertions do not stop cleanup in the normal Runner/Newman run.

## BUG-001: DELETE returns a creation status

- **Endpoint:** `DELETE /booking/{id}`.
- **Preconditions:** obtain a valid token and run `Create booking for BUG-001`.
- **Request:** `DELETE {{baseUrl}}/booking/{{knownBugBookingId}}`.
- **Headers:** `Cookie: token={{token}}`.
- **Body:** none.
- **Expected:** `200` or `204` for completed deletion. RFC 9110 section 9.3.5
  recommends these responses for completed DELETE operations; `201` describes
  resource creation (section 15.3.2).
- **Actual:** `201`, body `Created`.
- **Severity / priority:** Low / P3. Deletion works, but the status misrepresents
  the operation to clients.
- **Reproducibility:** reproduced manually in Postman on 2026-11-24 using a fresh
  booking. A stale booking produced `405`; this is not evidence for BUG-001.
- **Tracking request:** `Known bugs / BUG-001: DELETE returns a creation status`.
- **Tracking test:** `BUG-001: DELETE returns 200 or 204`.

## BUG-002: Incorrect credentials return a success status

- **Endpoint:** `POST /auth`.
- **Preconditions:** none.
- **Request:** `POST {{baseUrl}}/auth`.
- **Headers:** `Content-Type: application/json`.
- **Body:**

```json
{
    "username": "admin",
    "password": "wrong-password"
}
```

- **Expected:** the project's desired authentication failure status is `401`,
  whose meaning concerns invalid/missing authentication credentials (RFC 9110
  section 15.5.2). A complete 401 implementation would also provide the applicable
  `WWW-Authenticate` challenge. This is a design expectation for the JSON login
  endpoint, not a verified contract guarantee.
- **Actual:** `200`, body `{"reason":"Bad credentials"}`; no token is issued.
- **Severity / priority:** Low / P3. The login is rejected, but a client checking
  status alone could treat it as successful.
- **Reproducibility:** reproduced manually in Postman on 2026-11-24.
- **Tracking request:** `Known bugs / BUG-002: Wrong credentials return a success status`.
- **Tracking test:** `BUG-002: Wrong credentials return 401`.

## BUG-003: Missing required booking fields return 500

- **Endpoint:** `POST /booking`.
- **Preconditions:** none.
- **Request:** `POST {{baseUrl}}/booking`.
- **Headers:** `Content-Type: application/json`.
- **Bodies:** test each separately.

Empty object:

```json
{}
```

Missing `firstname`:

```json
{
    "lastname": "Booker",
    "totalprice": 150,
    "depositpaid": true,
    "bookingdates": {
        "checkin": "2027-01-10",
        "checkout": "2027-01-15"
    },
    "additionalneeds": "Breakfast"
}
```

- **Expected:** `400` with an explanation of the missing fields. Invalid client
  input should be classified as a client error rather than an unexpected server
  failure; compare RFC 9110 sections 15.5.1 and 15.6.1.
- **Actual:** both requests returned `500`, `Content-Type: text/plain; charset=utf-8`,
  body `Internal Server Error`.
- **Severity / priority:** Medium / P2. Invalid input triggers a server error and
  gives callers no field-level explanation.
- **Reproducibility:** each variant reproduced once by direct HTTP probe on
  2026-11-24 before adding assertions; automated results are recorded separately.
- **Tracking requests:** `Known bugs / BUG-003: Empty booking body` and
  `Known bugs / BUG-003: Missing firstname`.
- **Tracking tests:** `BUG-003: Empty booking body returns 400` and
  `BUG-003: Missing firstname returns 400`.

## BUG-004: Negative totalprice is accepted

- **Endpoint:** `POST /booking`.
- **Preconditions:** none for creation; obtain a token for cleanup.
- **Request:** `POST {{baseUrl}}/booking`.
- **Headers:** `Content-Type: application/json`.
- **Body:**

```json
{
    "firstname": "ValidationQA",
    "lastname": "Booker",
    "totalprice": -150,
    "depositpaid": true,
    "bookingdates": {
        "checkin": "2027-01-10",
        "checkout": "2027-01-15"
    },
    "additionalneeds": "Breakfast"
}
```

- **Expected:** `400` or `422` without creating a booking. The project assumes
  booking prices cannot be negative. HTTP 422 describes content that the server
  understands but cannot process semantically (RFC 9110 section 15.5.21).
- **Actual:** `200`, JSON `{"bookingid":<generated number>,"booking":<submitted body>}`;
  `booking.totalprice` was `-150`. The direct probe created booking `1592`, then
  deleted it successfully with `201`.
- **Severity / priority:** Medium / P2. The API creates data that violates the
  project's booking-price rule.
- **Reproducibility:** reproduced once by direct HTTP probe on 2026-11-24 before
  adding assertions; automated results are recorded separately.
- **Tracking request:** `Known bugs / BUG-004: Negative totalprice`.
- **Tracking test:** `BUG-004: Negative totalprice is rejected with 400 or 422`.

## BUG-005: Checkout before checkin is accepted

- **Endpoint:** `POST /booking`.
- **Preconditions:** none for creation; obtain a token for cleanup.
- **Request:** `POST {{baseUrl}}/booking`.
- **Headers:** `Content-Type: application/json`.
- **Body:**

```json
{
    "firstname": "ValidationQA",
    "lastname": "Booker",
    "totalprice": 150,
    "depositpaid": true,
    "bookingdates": {
        "checkin": "2027-01-15",
        "checkout": "2027-01-10"
    },
    "additionalneeds": "Breakfast"
}
```

- **Expected:** `400` or `422` without creating a booking. The project assumes
  checkout cannot precede checkin; HTTP 422 fits a semantically invalid date range
  (RFC 9110 section 15.5.21).
- **Actual:** `200`, JSON `{"bookingid":<generated number>,"booking":<submitted body>}`;
  checkin was `2027-01-15` and checkout `2027-01-10`. The direct probe created
  booking `1596`, then deleted it successfully with `201`.
- **Severity / priority:** Medium / P2. The API creates a booking with an invalid
  stay interval under the project's rule.
- **Reproducibility:** reproduced once by direct HTTP probe on 2026-11-24 before
  adding assertions; automated results are recorded separately.
- **Tracking request:** `Known bugs / BUG-005: Checkout before checkin`.
- **Tracking test:** `BUG-005: Checkout before checkin is rejected with 400 or 422`.

HTTP semantics reference: [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html).

## Automated verification

Verified against the live demo API on 2026-11-24 with Newman 6.2.3 and the
committed demo environment starting with empty runtime values:

| Run | Requests | Assertions | Passed | Failed | Exit code |
| --- | --- | --- | --- | --- | --- |
| Health, Auth, Bookings, Negative | 21 | 65 | 65 | 0 | 0 |
| Auth, Known bugs | 10 | 22 | 16 | 6 expected | 1 |

The six expected failures were BUG-001, BUG-002, both BUG-003 variants,
BUG-004, and BUG-005. No request or script errors occurred. All cleanup requests
succeeded, including those after the failed bug assertions. These results describe
the observed run, not a guarantee that the public demo service will remain unchanged.
