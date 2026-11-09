# Assignment Requirements — School Admission System

This document is a concise implementation reference for the five-page take-home assignment. It is not permission to add features beyond the brief.

## Business scenario
A school accepts admission applications on a first-come, first-served basis; there is no lottery. Each class/shift combination is an independent admission category with its own seat capacity. A successful application temporarily holds one seat for a student. Payment is simulated: clicking the payment action is assumed successful and confirms the admission. An unpaid hold expires and releases the seat automatically.

## Required user workflows
### School admin
- Create a class/admission category by entering its name and total seat count.
- View class list and seat counts.

### Parent/student
- No login flow is required by the brief.
- Enter the student's 17-digit birth registration number; use it as the student identifier.
- View classes, capacity, held seats, confirmed seats, available seats, and apply.
- View an application, its status, and expiration time/countdown.
- Simulate payment to confirm an eligible reservation; cancel an active reservation.
- View the student's applications.
- After page refresh or browser restart, restore the entered student context and show the student's existing applications.
- Refresh class availability automatically within a few seconds; show understandable errors, not blank pages or console-only failures.

## Exact API contract
Do not rename routes, request fields, response fields, or required headers. Additional response fields are allowed, but required fields must remain.

| # | Method and route | Required contract |
|---|---|---|
| 1 | `POST /classes` | Body: `{ "name": string, "totalSeats": number }`; success `201`; response includes `id`, `name`, `totalSeats`. |
| 2 | `GET /classes` | `200`; array with `id`, `name`, `totalSeats`, `heldSeats`, `confirmedSeats`, `availableSeats`. |
| 3 | `GET /classes/{classId}` | `200`; same class/seat fields as #2. |
| 4 | `POST /classes/{classId}/holds` | Required header `X-Student-Id`; success `201`; response includes `holdId`, `classId`, `studentId`, `status`, `expiresAt`; no seat returns `409`. |
| 5 | `GET /holds/{holdId}` | `200`; response includes `holdId`, `classId`, `status`, `expiresAt`. |
| 6 | `POST /holds/{holdId}/confirm` | Required header `X-Student-Id`; success `200`; response includes `holdId`, `status`. |
| 7 | `DELETE /holds/{holdId}` | Required header `X-Student-Id`; successful cancellation returns `204 No Content`. |
| 8 | `GET /me/holds` | Required header `X-Student-Id`; `200`; array includes `holdId`, `classId`, `className`, `status`, `expiresAt`. |

Allowed application statuses: `HELD`, `CONFIRMED`, `EXPIRED`, `CANCELLED`.

IDs in the assignment (`c_123`, `h_789`) are examples; the brief permits choosing the ID format. Keep IDs JSON-string-compatible and document the choice.

## Mandatory backend invariants
1. At all times, for every admission class: `active HELD + CONFIRMED <= totalSeats`.
2. Hold duration comes from `HOLD_DURATION_SECONDS`, default `600` seconds. Expiration must occur without requiring an API call.
3. A student cannot have more than one active `HELD` or `CONFIRMED` application for the same class. Different shifts are different classes.
4. Only the owning student can confirm or cancel a hold.
5. Only an unexpired `HELD` application may be confirmed.
6. Confirming the same application repeatedly must never allocate/increment a seat more than once.
7. Application data must survive application/container restart and `docker compose down` followed by `docker compose up`.

## Mandatory frontend and delivery constraints
- Admin and parent operations must be possible through the React UI; do not require Postman or direct database access for normal workflows.
- API URL and database connection must be configurable using environment variables.
- Root `docker-compose.yml` must start PostgreSQL, backend, and frontend together; backend listens at `http://localhost:8080`.
- At least three automated backend tests, including a concurrency test that demonstrates capacity cannot be exceeded.
- Include `ASSUMPTIONS.md`, `AI_USAGE.md`, and `README.md`.
- Submit a public GitHub repository before the deadline stated on the assignment cover. Do not publish the confidential assignment PDF.

## Out of scope unless explicitly approved
Real payment gateway, login/authentication, student account management, academic-year administration, notifications, analytics, microservices, Redis, message brokers, and Kubernetes.
