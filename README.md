# Tiffin Service

Tiffin Service is an owner-friendly tiffin subscription desk. It keeps a monthly plan intact, records pause/resume periods, and calculates a fair bill from the weekdays actually served.

## Run locally

Requirements: Node.js 18+ and npm.

```bash
npm install
npm start
```

Open http://localhost:3000. The SQLite database is created as `tiffinservice.sqlite` on first run. For development, use `npm run dev`. Syntax checks run with `npm run check`.

## Product flow

1. Register an owner account or sign in.
2. Add customers with phone, address, join date, and monthly plan price.
3. Search by name or phone and sort the paginated list.
4. Pause a subscription for a date range, or resume an open pause.
5. Open a customer bill. The server calculates weekday count minus paused weekdays, then prorates the plan price.

## REST API

All customer endpoints require the `tiffinservice_session` cookie created by auth.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| POST | `/api/auth/register` | Create an owner and session |
| POST | `/api/auth/login` | Sign in and create a session |
| POST | `/api/auth/logout` | End the current session |
| GET | `/api/auth/me` | Check the current session |
| GET | `/api/customers?search=&page=&limit=&sort=&order=` | Search, paginate, and sort customers |
| POST | `/api/customers` | Create a customer subscription |
| GET | `/api/customers/:id` | Get a customer and pause history |
| POST | `/api/customers/:id/pause` | Add a pause period `{startDate, endDate?}` |
| DELETE | `/api/customers/:id/pause` | Resume the current open pause |
| GET | `/api/customers/:id/bill?month=YYYY-MM` | Return the pro-rated bill |
| POST | `/clock` | Notify active weekday deliveries for `{date}` into the outbox |
| GET | `/outbox?date=YYYY-MM-DD` | Inspect queued delivery notifications |
| POST | `/api/customers/import` | Import `{csv}` or `{rows}` and return `imported`, `deduped`, `rejected` |
| POST | `/api/customers/:id/transfer` | Transfer the subscription to `{name, phone, address, transferDate}` |

## Data model

SQLite stores `users`, `customers`, and `pause_periods`. Customer phone numbers are unique. A pause can be open-ended, which represents an active pause; resuming closes it on the current date. The bill endpoint computes weekdays in UTC to avoid timezone edge cases.

### Twist workflows

- **Morning notifications:** `POST /clock` with `{"date":"2026-09-18"}` queues one notification per active, unpaused customer when the date is a weekday. `GET /outbox?date=2026-09-18` returns the durable notification records. The unique customer/date constraint makes clock retries idempotent.
- **Mid-cycle transfer:** `POST /api/customers/:id/transfer` with the new customer details and `transferDate`. The new subscription keeps the original monthly plan price. Bills split weekday service at the transfer date: the original customer is billed before it, and the new customer after it.
- **Messy import:** `POST /api/customers/import` accepts CSV columns such as `name,phone,address,plan_price,joined_on`, or a JSON `rows` array. It accepts ISO dates, `DD/MM/YYYY`, `MM/DD/YYYY`, and common parseable dates. Duplicate phones are counted as `deduped`; blank or incomplete rows are `rejected` with row-level errors.

## Debugging

Use `npm run dev` for automatic server restarts. API errors are JSON with an `error` message. The browser console and network panel show client-side request details. Delete `tiffinservice.sqlite` to reset local data.

## Next three features

- Route-aware delivery runs with driver assignment and address clustering.
- Payment links, receipts, and outstanding-balance tracking.
- Daily kitchen quantities and delivery notes per customer.