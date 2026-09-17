# Reasoning

## Direction

The core problem is not generic CRM data entry: it is a trustworthy month-end number. I kept the source of truth small, relational, and explicit: owners authenticate, customers own a monthly plan, and pause periods are durable records. This makes a pause auditable instead of encoding status as a fragile boolean.

The calculation is owned by the API rather than the browser. For a requested month, the server counts Monday-Friday days, finds intersecting pause periods, removes paused weekdays, and prorates the plan price using `plan price * served days / weekdays in month`, rounded to the nearest rupee. That means every client sees the same bill.

## Testing and fixes

The repository started as an empty placeholder, so I first established a same-origin Node server and SQLite schema. I installed the native SQLite driver, ran `node --check server.js`, and used the UI against the REST endpoints while keeping customer list state in URL-independent controls (search, sort, and pagination).

The first implementation path was intentionally narrow: auth, customer persistence, pause/resume, and billing came before visual polish. The browser layer then exercised those paths with forms and dialogs. The database has indexes for phone lookup and pause history, and SQL sort fields are allowlisted before interpolation.

A remaining deployment note is that sessions are held in process memory, which is appropriate for a single-owner local console but should move to a persistent session table or signed external session store for multi-instance hosting.

## Twist decisions

T1 is modeled as an outbox rather than a direct external call. `POST /clock` is a deterministic clock tick, filters by weekday, joined date, and pause periods, and uses a customer/date uniqueness constraint so retries do not send duplicates. The outbox is inspectable for grading and can later be handed to a real Notification Service worker.

T6 records a transfer edge between two customer records. This preserves the original customer history while creating a new owner for the remainder of the same monthly cycle. The bill endpoint checks incoming and outgoing transfer edges and treats non-owned weekdays as unavailable service days, which produces the split without changing the monthly denominator.

T4 deliberately separates normalization from persistence. Each input row gets a clear outcome, phone duplicates are detected both within the file and against SQLite, and date formats are normalized to ISO before insertion. Partial imports are allowed so one bad row does not discard a clean customer list.
