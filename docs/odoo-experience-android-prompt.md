# Programmer prompt: Odoo Experience Android booth assistant

Build an Android app for OduSphere booth staff at Odoo Experience 2026, stand A3.
The app captures business cards, lets staff review recognized details, registers
visitors in the control Odoo, and adds typed or dictated private notes. Produce
an installable APK, source, build instructions, and automated tests.

## User journey

1. Configure the HTTPS origin (default `https://control.demo.odusfera.pl`) and a
   personal, expiring Odoo API key. Each operator has their own internal account
   with the `Odoo Experience Booth Staff` group. Never embed a shared credential.
2. Fetch events and let staff select Odoo Experience 2026 / A3. Remember the
   selection, but refresh event availability before sending new visitors.
3. Scan a card using the camera or select a photo with Android Photo Picker.
   Crop, rotate, and run OCR on the device. Prefer CameraX and ML Kit Text
   Recognition v2. The Latin recognizer does not promise support for every
   language; show unrecognized fields as editable and support manual entry.
4. Review first name, last name, email, company (required), job title, phone
   (optional), and explicitly choose Integrator or User. Never infer this role
   from OCR. Integrator invitation issuance is approval for source repository
   access, so show that consequence before submitting. Never submit OCR results
   without staff review. Keep the original recognized text available locally
   while correcting the fields.
5. Submit the reviewed visitor. Display the resulting personal invitation as a
   QR code, copyable URL and Android share action. Sharing is an explicit staff
   action; do not email or text visitors automatically. Preserve the URL fragment:
   the token belongs directly after `#`, not in the query string or a URL shortener.
6. Open a visitor's Notes screen and add multiple dated notes. Support typing
   and speech-to-text with explicit microphone permission and visible recording
   state. Prefer on-device recognition where available; clearly disclose any
   system recognizer's network processing and always provide keyboard fallback.
   Staff must review dictated text before submitting. Do not store audio or
   card photos on the server. Notes are private to booth staff.

## Backend contract (already implemented)

Use HTTP JSON, not JSON-RPC. All API calls require
`Authorization: Bearer <personal Odoo API key>`. Cookies alone do not authenticate
these endpoints. POST requests use `Content-Type: application/json`; no CSRF token
is needed for these bearer-authenticated API routes. Browser registration forms
have separate CSRF protection. The account's allowed Odoo companies scope data.

`GET /odoo-experience/api/events` returns:

```json
{"events": [{"id": 42, "name": "Odoo Experience 2026", "year": 2026, "booth": "A3"}]}
```

Use the returned event ID; never hardcode 42.

`POST /odoo-experience/api/visitors` accepts:

```json
{
  "request_id": "1930effe-56db-4d20-a7d0-3e9ba4c32df9",
  "event_id": 42,
  "first_name": "Ada",
  "last_name": "Lovelace",
  "email": "ada@example.com",
  "company_name": "Example Ltd",
  "job_title": "Director",
  "phone": "+48123456789",
  "visitor_type": "integrator"
}
```

`visitor_type` is exactly `integrator` or `user`. Limits: first/last name 100
characters each; email 254; company/job title 200; phone 80. Optional string
fields may be omitted or empty, not `null`. Unknown ownership or permission
fields do not grant access. The API validates and normalizes email.

Response: HTTP 201 on creation; 200 on an identical retry:

```json
{
  "id": 123,
  "created": true,
  "state": "ready",
  "registration_url": "https://control.demo.odusfera.pl/odoo-experience/register#REDACTED"
}
```

Ready means the token link is prepared. Invited additionally means staff have
confirmed that the link was emailed and/or the photo was printed; merely
scanning a card does not mark delivery.

The same `event_id` + `request_id` + normalized payload returns the same visitor.
Reusing the UUID with different data returns HTTP 400. After registration or
revocation, a retry returns the record with its current state and a null URL.

`POST /odoo-experience/api/visitors/123/notes` accepts:

```json
{
  "request_id": "5c0fdf46-dfb5-4437-a409-4d58f7db5ca1",
  "body": "Interested in implementing Odoo for three warehouses."
}
```

Body: nonblank string, maximum 10,000 characters. Response: HTTP 201 for a new
note or 200 for an identical retry, `{"id": 456, "created": true}`. Note UUIDs
are scoped to the visitor. Reusing the UUID with different text returns 400.
API request bodies are limited to 32 KiB. Validation errors use `{"error": "..."}`.
Also handle non-JSON 401/403/404/413 and reverse proxy errors without crashing.
The current API does not provide visitor search/history: maintain the app's own
locally submitted records; server records are managed in Odoo's backend.

## Offline delivery and security

Use Kotlin, Jetpack Compose, CameraX, Room and WorkManager. Persist the reviewed
payload and its random UUID before dispatch. Retry timeouts, offline failures,
429 (honor Retry-After), and 5xx with exponential backoff and jitter using the
same UUID and unchanged payload. Never generate another UUID after an ambiguous
response: the server may already have created the record. Queue notes behind
the successful visitor creation. Distinguish draft, queued, sending, delivered,
and action-required states. Allow cancelling a local unsent draft; do not label
server records as deleted when only removing local history.

Do not silently change a payload that may have reached the server. Resolve its
outcome first; a changed UUID represents a separate visitor, not an update.
Do not retry validation errors or invalid credentials indefinitely. Ask staff
to correct data or replace the key. Do not delete unsent work on authentication
failure. Prevent two workers from sending conflicting versions of a record.

Protect keys using Android Keystore-backed storage and exclude sensitive data
from backups, analytics, crash reports and logs. Use normal TLS certificate
validation, reject cleartext HTTP, and do not forward Authorization across
origins. Keep invitation URLs out of logs and analytics too. Remove temporary
card images after recognition/review; provide a configurable local history
retention and clear-data action. Request only necessary permissions. Never
embed GitHub credentials or implement infrastructure provisioning in this app.

## Server-side registration outcomes

The visitor opens the personal link, reviews prefilled data, then sets a password
and enters their portal. The invitation expires and can only be consumed once.
The visitor cannot change their Integrator/User classification.

- Integrator: a company, child user/contact and approved Oduflow Partner are
  created. Repository access synchronization runs asynchronously.
- User: a client company and child portal user are created; a draft Instance
  belongs to OduSphere. No server starts and no subscription is charged by this
  registration. A plan and payment are required before activation.

## Acceptance tests and deliverables

Demonstrate manual registration, a clear Latin-script card, a rotated/poor card
with correction, missing OCR fields, permission denial, offline submission,
app restart while queued, a lost response followed by an idempotent retry,
repeated note submission, expired API credentials, foreign-company access
denial, Integrator/User selection, QR scanning with intact fragment, and no
sensitive information in logs. Tests must not register real visitors or send
messages. Include accessibility, large text and landscape/tablet checks.

Deliver source, reproducible build instructions, APK, short operator guide,
architecture notes and test results. Use a staging environment for integration
checks with transaction-backed fixtures or approved disposable test accounts.

Reference documentation:
- ML Kit: https://developers.google.com/ml-kit/vision/text-recognition/v2/android
- Odoo HTTP authentication: https://www.odoo.com/documentation/19.0/developer/reference/backend/http.html
