# Restful-Booker API Tests

[![API tests](https://github.com/llunasv/restful-booker-api-tests/actions/workflows/api-tests.yml/badge.svg?branch=main)](https://github.com/llunasv/restful-booker-api-tests/actions/workflows/api-tests.yml)

QA portfolio project testing the Restful-Booker API with Postman, Newman, HTML reports, and GitHub Actions.

The tests cover authentication, booking CRUD, missing authentication, invalid
credentials, and input validation. Main-folder assertions match observed API
behavior; separate known-bug assertions express the desired behavior and are
expected to fail while the issues remain.

- **API:** https://restful-booker.herokuapp.com
- **API documentation:** https://restful-booker.herokuapp.com/apidoc
- **Stack:** Postman Collection v2.1, JavaScript assertions, Newman,
  newman-reporter-htmlextra, GitHub Actions.

## Coverage

| Area | Endpoint | Scenario | What is checked |
| --- | --- | --- | --- |
| Health | `GET /ping` | API availability | Observed status `201` |
| Auth | `POST /auth` | Valid credentials | `200`, non-empty token, token saved for later requests |
| Bookings | `POST /booking` | Create a complete booking | `200`, JSON Content-Type, numeric ID, all returned fields match the request |
| Bookings | `GET /booking/{id}` | Retrieve created booking | `200`, JSON Content-Type, all fields match the submitted data |
| Bookings | `GET /booking?firstname=...&lastname=...` | Filter by name | `200`, array response, created ID included |
| Bookings | `PUT /booking/{id}` | Replace all booking fields | `200`, complete response matches the update |
| Bookings | `PATCH /booking/{id}` | Change firstname and price | `200`, selected fields changed, remaining fields unchanged |
| Bookings | `DELETE /booking/{id}` | Authenticated deletion | Observed status `201` |
| Bookings / Negative | `GET /booking/{id}` | Retrieve a deleted booking | `404`; the negative request also checks `Not Found` |
| Negative | `PUT /booking/{id}`, `DELETE /booking/{id}` | Missing token | `403`, `Forbidden`; stored cookies disabled |
| Negative | `POST /auth` | Incorrect password | Observed `200`, `Bad credentials` reason, no returned token |
| Negative | `POST /booking` | Empty object or missing firstname | Observed `500`, `Internal Server Error`; desired rejection tracked separately |
| Negative | `POST /booking` | Negative price or checkout before checkin | Observed `200`, returned invalid values, generated ID captured for cleanup |
| Known bugs | `DELETE /booking/{id}`, `POST /auth`, `POST /booking` | Desired status and validation behavior | Six intentionally failing assertions covering five reported issues |

A collection-level assertion checks that each executed request responds in less
than **2,000 ms**. This is a response-time threshold, not a load test or service
availability guarantee. The public demo service can be slow or reset its data.

Setup and cleanup requests are included in the collection. Validation bookings
that the API accepts are deleted immediately after their checks.

## Request chain and variables

Run the main folders in this order:

```text
Health → Auth → Bookings → Negative
```

1. `Create token` sends the public demo credentials and saves `token`.
2. `Create booking` generates a unique `bookingFirstname`, then saves `bookingId`
   and the submitted payload as `expectedBooking`.
3. GET and search requests use those variables to locate and verify the booking.
4. PUT and PATCH authenticate with `Cookie: token={{token}}` and update the
   expected payload used by subsequent checks.
5. DELETE removes the booking. The following GET reuses its ID to verify `404`.
6. Negative tests create a separate `negativeBookingId`. Validation requests use
   `validationBookingId` for immediate cleanup of accepted invalid inputs.

The known-bug DELETE scenario has its own setup request and `knownBugBookingId`.
Run **Auth**, then **Known bugs** when reproducing issues separately.

| Environment value | Initial value |
| --- | --- |
| `baseUrl` | `https://restful-booker.herokuapp.com` |
| `username` | `admin` |
| `password` | `password123` |
| Tokens, booking IDs, expected payload, generated firstname | Empty; populated by scripts |

The credentials above are public demo credentials. No personal API key or
Postman account token is required by Newman or CI. Booking IDs are obtained from
responses rather than hard-coded into requests.

## Run in Postman

1. Import both files from `postman/`:
   - `RestfulBooker.postman_collection.json`
   - `restful-booker.postman_environment.json`
2. Select **Restful Booker Demo** as the active environment.
3. Open the collection in **Collection Runner**.
4. Select **Health**, **Auth**, **Bookings**, and **Negative**; exclude **Known bugs**.
5. Run one iteration in the folder order shown above.

For known bugs, select only **Auth** and **Known bugs**. Allow the run to continue
after failed assertions so cleanup requests can execute.

Sending PUT, PATCH, or DELETE alone requires valid values from earlier requests.
If a token is no longer accepted, rerun Auth. If the demo API has removed a
booking, restart the chain with a new booking instead of manually inserting an ID.

## Run with Newman

Install Node.js and npm. CI is configured for Node.js 20; the local runs recorded
below used Node.js 22.14.0.

From the repository root:

```sh
npm ci
npm test
```

`npm ci` restores the versions in `package-lock.json`. `npm test` runs the four
main folders and generates terminal, HTML, and JSON output.

The equivalent Newman command is:

```sh
npx newman run postman/RestfulBooker.postman_collection.json -e postman/restful-booker.postman_environment.json --folder Health --folder Auth --folder Bookings --folder Negative --reporters cli,htmlextra,json --reporter-htmlextra-export reports/report.html --reporter-json-export reports/report.json
```

To reproduce known bugs separately:

```sh
npm run test:known-bugs
```

This command returns exit code **1** when the known-bug assertions fail. The main
command returns **0** when all its assertions pass. Neither command suppresses
failure exit codes.

### Reports

| Run | HTML report | JSON report |
| --- | --- | --- |
| Main folders | `reports/report.html` | `reports/report.json` |
| Auth + Known bugs | `reports/known-bugs.html` | `reports/known-bugs.json` |

Open an HTML report in a browser to inspect requests and assertions. A subsequent
run overwrites the same report paths. JSON reports support further automated
inspection. `reports/` and `node_modules/` are ignored by Git.

## GitHub Actions

The [API tests workflow](.github/workflows/api-tests.yml) runs on:

- Pushes to `main`.
- Pull requests, including those targeting `dev`.
- Manual `workflow_dispatch` runs, once available on the default branch.

The Ubuntu job checks out the repository, sets up Node.js 20, runs `npm ci`, and
executes `npm test`. The npm command explicitly selects the main folders, so
**Known bugs is excluded from CI**.

The report-upload step uses `if: always()` to preserve available reports after
test failures. Find the run under **Actions → API tests**, then download the
**api-test-reports** artifact and open `report.html`. Artifacts are retained for
14 days. If installation fails before reports are generated, the job remains
failed and the upload step warns that no report files were found.

## Known bugs

| ID | Observed issue | Desired behavior |
| --- | --- | --- |
| [BUG-001](docs/bug-reports.md#bug-001-delete-returns-a-creation-status) | DELETE returns `201 Created` | `200` or `204` for completed deletion |
| [BUG-002](docs/bug-reports.md#bug-002-incorrect-credentials-return-a-success-status) | Wrong credentials return `200` | Project expectation: `401` |
| [BUG-003](docs/bug-reports.md#bug-003-missing-required-booking-fields-return-500) | Empty body or missing firstname returns `500` | Reject missing required fields with `400` |
| [BUG-004](docs/bug-reports.md#bug-004-negative-totalprice-is-accepted) | Negative price is accepted | Project expectation: reject with `400` or `422` |
| [BUG-005](docs/bug-reports.md#bug-005-checkout-before-checkin-is-accepted) | Checkout before checkin is accepted | Project expectation: reject with `400` or `422` |

The demo API intentionally contains unusual behavior. Desired business rules
and authentication design expectations are distinguished from verified contract
requirements in the [bug reports](docs/bug-reports.md).

Known-bug assertions remain separate to demonstrate the defects without making
every ordinary regression run fail. Two BUG-003 variants account for six failing
assertions across five reported issues. Cleanup requests still run after these
assertions; do not use `--bail` when reproducing them.

## Verification snapshot

Live local runs on **2026-11-24**, using Newman **6.2.3**:

| Run | Requests | Assertions | Passed | Failed | Exit code |
| --- | --- | --- | --- | --- | --- |
| Main folders | 21 | 65 | 65 | 0 | 0 |
| Auth + Known bugs | 10 | 22 | 16 | 6 expected | 1 |

Setup and cleanup completed successfully. These are local results; the badge
at the top reports the GitHub Actions status for `main` separately.

<!-- Screenshot placeholder: add docs/images/runner.png, then uncomment the image below.
![Postman Collection Runner showing the main folders passing](docs/images/runner.png)
-->

## License

[MIT](LICENSE).
