# Smart Digital Locker Management System

## Complete Product Requirements, Architecture, Database, Security, UX, Business Rules and Implementation Plan

**Document:** Smart Digital Locker --- V1.0 Baseline Specification\
**Status:** Requirements consolidated and decisions finalized\
**Primary implementation:** Node.js backend, React web application,
MongoDB, ESP32 IoT hardware\
**Initial operating model:** Simulated UPI payments, simulated
SMS/email, simulated IoT mode before physical integration\
**Currency:** INR (₹)\
**Timezone for display and facility operations:** Asia/Kolkata\
**Date of specification:** 25 November 2026

------------------------------------------------------------------------

# 0. Document purpose and authority

This document is the consolidated baseline for building the Smart
Digital Locker product. It includes the requirements, finalized
decisions, user and admin flows, business rules, state machines,
database structures and relationships, APIs, UI structure, security
controls, IoT architecture, error handling, testing, deployment, and
phased delivery plan discussed for this project.

The implementation team must use this document as the source of truth.
Where a technical detail was not explicitly decided, this document
identifies a recommended implementation default rather than pretending
it was an already-approved business decision.

**No implementation may weaken the security or business invariants in
this document.** In particular:

-   Payment success is not the same as physical unlock success.
-   An expired booking does not make an occupied locker available.
-   A locker becomes available only after the system has confirmed final
    retrieval, door closure, and locked state.
-   The backend is authoritative for identity, pricing, payment state,
    booking state, access authorization, and hardware command
    authorization.
-   The browser must never be trusted to assert that payment succeeded
    or that a physical locker opened.
-   Personal contact data must be encrypted at rest and searchable
    through keyed blind indexes.
-   OTPs must be context-bound, short-lived, single-use challenges.

## 0.1 Scope

The system supports:

1.  User registration with name, email, phone number and a choice of
    email OTP or mobile OTP.
2.  Login using either a previously verified email or previously
    verified mobile number, with a fresh OTP on every login.
3.  User profile and contact verification status.
4.  Viewing available physical lockers.
5.  Selecting a locker and rental duration.
6.  A 10-minute temporary locker reservation while checkout/payment is
    completed.
7.  Fixed-duration rental pricing.
8.  Simulated UPI QR payment for development.
9.  Activation of a paid storage booking.
10. Unlocking a physical locker through an ESP32-controlled electronic
    lock.
11. Door-open and door-closed sensing.
12. Partial retrieval while retaining the locker.
13. Complete retrieval and release of the locker.
14. Expiry detection and SMS/email notification through the verified
    contact method.
15. Overdue charges of ₹20 per started hour.
16. Payment of overdue charges before retrieval after expiry.
17. Paid-through billing for partial retrieval after expiry.
18. Admin management of locations, lockers, bookings, payments, users,
    devices and reports.
19. Audit trails, monitoring, failure recovery and operational controls.

## 0.2 Explicitly out of scope for the initial simulation

These are not to be represented as real capabilities until integrated
and tested:

-   Real UPI payment collection or bank settlement.
-   Real SMS or email provider delivery.
-   Production-grade remote physical access to lockers.
-   Insurance, liability assessment or item-content verification.
-   Automated recognition of items placed inside lockers.
-   Dynamic pricing or discounts.
-   Refund policy beyond recording and controlled handling of payment
    adjustments.
-   Multi-tenant franchise billing.
-   Native mobile applications.

The architecture must allow real providers and devices to replace
simulations through adapters without rewriting the core booking and
access business logic.

------------------------------------------------------------------------

# 1. Finalized product decisions

The following decisions are confirmed and must be treated as fixed for
V1 unless a future change is explicitly approved.

  -----------------------------------------------------------------------
  Topic                               Final decision
  ----------------------------------- -----------------------------------
  Platform                            Responsive web application

  User interface                      React.js

  Backend                             Node.js with Express.js

  Language style                      JavaScript ES modules; modern
                                      maintained APIs

  Database                            MongoDB with Mongoose

  Signup fields                       Name, email, phone number

  Signup verification                 User chooses either email OTP or
                                      mobile OTP

  Initial verification                Only the chosen method is verified
                                      at signup

  Login identifier                    Either email or mobile

  Login verification                  Fresh OTP sent to the selected
                                      contact for every login

  Unverified login contact            Must not be used to establish a
                                      session

  OTP design                          Context-based verification
                                      challenge with unique verification
                                      ID

  OTP size                            Six digits

  OTP expiry                          Five minutes

  OTP attempts                        Maximum five per challenge

  OTP resend                          Cooldown, previous challenge
                                      invalidated

  Contact encryption                  AES-256-GCM

  Contact lookup                      HMAC-SHA256 blind indexes using a
                                      separate secret

  Access authentication               Short-lived JWT access token and
                                      rotating refresh token

  Locker reservation timeout          10 minutes

  Initial payment                     Simulated UPI QR

  Rental durations                    4, 6, 8, 12, 24 hours only

  Rental prices                       ₹80, ₹90, ₹100, ₹120, ₹240
                                      respectively

  Overdue charge                      ₹20 per started hour

  Overdue billing                     Started-hour ceiling calculation

  Partial retrieval after expiry      Paid-through billing

  Expiry notification                 Send to verified contact only

  Notification preference             Use the verified contact method; no
                                      unverified destination

  Background jobs                     BullMQ + Redis

  IoT communication                   MQTT with authenticated device
                                      identity and TLS

  Hardware controller                 ESP32

  Initial IoT development             Simulated device adapter before
                                      physical integration

  Architecture                        Modular monolith, not microservices
                                      for V1

  API version                         `/api/v1`

  API documentation                   OpenAPI / Swagger

  Logging                             Pino with redaction

  Validation                          Zod

  Testing                             Unit, integration, concurrency and
                                      end-to-end tests

  Deployment                          Docker and CI/CD
  -----------------------------------------------------------------------

## 1.1 Notification decision in detail

-   Signup requires one contact method to be verified.
-   The other contact may be stored encrypted but remains unverified.
-   Expiry notifications must be sent only to a contact marked verified.
-   The verified contact is the default notification destination.
-   If the user later verifies the second contact, the system may treat
    it as eligible for notifications. The notification service must use
    the current verified-contact policy and must never send to an
    unverified contact.
-   Notification delivery status must be stored.
-   A failed notification must be retried safely without creating
    duplicate successful notifications.
-   Do not include unnecessary sensitive details in SMS/email.

## 1.2 Reservation timeout decision in detail

The selected locker is held for **10 minutes** from successful
reservation creation.

-   The server creates `reservationExpiresAt`.
-   The frontend displays the remaining checkout time.
-   The server checks the expiry on every payment and booking
    transition.
-   A background job releases abandoned reservations.
-   A MongoDB TTL index may clean up old documents, but TTL deletion is
    asynchronous and must not be the mechanism that enforces
    availability.
-   If payment confirmation arrives after the reservation has expired or
    the locker has been reassigned, the system must not activate that
    booking or unlock the locker. It must reconcile the payment and
    create a controlled refund/exception workflow.
-   The reservation must be atomic so two users cannot reserve the same
    locker.

## 1.3 Paid-through overdue billing decision in detail

For a booking that has expired, the system calculates unpaid overdue
time from the latest successfully paid-through timestamp. If no overdue
payment has ever been made, the baseline is the original
`scheduledEndAt`.

When an overdue payment succeeds:

-   Store the amount, calculation snapshot, covered start time and
    covered-through timestamp.
-   Advance `overduePaidThrough` only after confirmed payment.
-   The covered-through timestamp is the payment
    authorization/confirmation time used by the server's billing policy.
-   If the user partially retrieves and retains the locker, the next
    overdue period starts at the paid-through timestamp.
-   At the next access request, calculate only the new unpaid period.
-   Do not charge the same time interval twice.
-   A failed or pending payment must not advance the paid-through
    timestamp.
-   If a user pays but does not open the locker, the amount remains
    credited to the booking's paid-through period; do not charge the
    same period again.
-   A payment reconciliation process must resolve uncertain payment
    outcomes.

Example:

-   Scheduled expiry: 2:00 PM.
-   User returns: 4:20 PM.
-   Started overdue hours: `ceil(2h20m / 1h) = 3`.
-   Amount: `3 × ₹20 = ₹60`.
-   Payment confirmed at 4:20 PM; paid-through timestamp becomes 4:20
    PM.
-   User takes only some items and closes the locker.
-   User returns at 6:35 PM.
-   New overdue interval: 2h15m.
-   New charge: `ceil(2.25) × ₹20 = ₹60`.
-   The original 2:00 PM--4:20 PM interval is not charged again.

The precise server timestamp used for payment coverage must be persisted
and reused consistently. Do not derive it from a browser clock.

------------------------------------------------------------------------

# 2. Product actors and permissions

## 2.1 User

A user can:

-   Register and verify one contact method.
-   Log in using either verified contact method with fresh OTP.
-   View and update allowed profile details.
-   Verify the second contact method.
-   View available lockers.
-   View locker details and pricing.
-   Reserve a locker.
-   Pay the initial simulated rental charge.
-   View active and expired bookings.
-   Request access to their own occupied locker.
-   Pay overdue charges.
-   Choose partial or complete retrieval.
-   View booking and payment history.
-   View notifications.

A user cannot:

-   Access another user's booking or locker.
-   Choose or override a payment amount.
-   mark a payment successful from an ordinary production API.
-   Send an arbitrary unlock command.
-   Mark a door closed without device confirmation.
-   release a locker while the door is open or the lock state is
    unknown.
-   Change the authoritative booking expiry or paid-through time.

## 2.2 Administrator

An administrator can, subject to role-based authorization:

-   View dashboard and operational metrics.
-   Create and manage locations.
-   Create, edit, disable and maintain lockers.
-   Register and monitor IoT devices.
-   View users and verification status with appropriately masked
    personal data.
-   View bookings and payment history.
-   Review device health and failed commands.
-   View audit logs.
-   Configure pricing plans for future bookings.
-   Initiate controlled maintenance/recovery workflows.
-   Review notification delivery status.

Admin actions must be audited. Admins must not casually force an
occupied locker to `available`. Emergency access and manual overrides
require a separately authorized, documented, audited recovery workflow.

## 2.3 IoT device

An ESP32 device can:

-   Authenticate to the MQTT broker.
-   Publish heartbeat and device status.
-   Receive commands addressed to its device identity.
-   Validate command ID, expiry, sequence/replay protection and command
    format.
-   Operate the lock driver within configured safety limits.
-   Read door sensor state.
-   Publish command acknowledgement and execution result.
-   Publish door-open, door-closed and lock-state events.

An ESP32 cannot decide whether a user has paid. The backend authorizes
access and issues a limited command.

------------------------------------------------------------------------

# 3. Complete user journeys and flows

## 3.1 Signup flow

### User interface fields

-   Full name.
-   Email address.
-   Phone number.
-   Verification method choice:
    -   Verify using email.
    -   Verify using mobile.
-   Consent to terms/privacy notice as required by the product's legal
    review.

### Signup sequence

1.  User opens `/signup`.
2.  User enters name, email and phone.
3.  User selects email or mobile verification.
4.  Frontend sends the signup request.
5.  Backend validates and normalizes the fields.
6.  Backend calculates keyed HMAC lookup values for email and phone.
7.  Backend checks uniqueness using the blind indexes.
8.  If a contact is already registered, return a safe conflict response
    without leaking another user's account details.
9.  Create a pending registration with encrypted contact values and an
    expiry.
10. Generate a cryptographically secure six-digit OTP.
11. Generate a cryptographically random UUID verification ID.
12. Bind the challenge to the pending registration, selected method and
    `signup` purpose.
13. Store only an HMAC of the OTP, not the plaintext OTP.
14. Send OTP through the selected channel.
15. Return verification ID and expiry duration, never the OTP.
16. User enters OTP.
17. Backend atomically verifies challenge, purpose, contact method,
    subject, expiry, attempt count and single-use state.
18. On success, consume the challenge.
19. Create the user record.
20. Mark only the chosen contact method as verified.
21. Mark pending registration as completed or consumed.
22. Create an authenticated session and issue access/refresh
    credentials.
23. Redirect to dashboard.

### Signup flow diagram

``` mermaid
flowchart TD
  A[Open signup] --> B[Enter name, email, phone]
  B --> C[Choose email or mobile verification]
  C --> D[Submit signup]
  D --> E[Validate and normalize]
  E --> F[Compute HMAC blind indexes]
  F --> G{Email and phone unique?}
  G -- No --> H[Return safe conflict response]
  G -- Yes --> I[Create pending registration]
  I --> J[Generate OTP and verification UUID]
  J --> K[Store OTP HMAC challenge]
  K --> L{Selected method}
  L -- Email --> M[Send email OTP]
  L -- Mobile --> N[Send SMS OTP]
  M --> O[User enters OTP]
  N --> O
  O --> P[Atomically verify challenge]
  P --> Q{Valid, unexpired, unused?}
  Q -- No --> R[Reject / retry / resend]
  Q -- Yes --> S[Consume OTP]
  S --> T[Create user and mark selected contact verified]
  T --> U[Create session and issue tokens]
  U --> V[Dashboard]
```

## 3.2 OTP context and security model

Every challenge must be bound to:

-   `verificationId`
-   subject type and subject ID
-   purpose (`signup`, `login`, `verify_contact`, and future approved
    purposes)
-   method (`email` or `phone`)
-   normalized-contact lookup reference or an immutable contact snapshot
    reference
-   OTP HMAC
-   expiry
-   attempt count and maximum attempts
-   status
-   consumed timestamp
-   creation timestamp

A six-digit OTP is **not required to be globally unique across all
users**. Security comes from binding it to a unique verification
challenge, user/pending-registration context, purpose and method. Two
users may coincidentally receive the same six-digit code without
compromising isolation.

### OTP generation

Use Node.js `crypto.randomInt(100000, 1000000)` or an equivalent
cryptographically secure generator.

### OTP storage

Use a keyed HMAC such as:

`HMAC-SHA256(otpSecret, verificationId + ":" + purpose + ":" + otp)`

Use a dedicated OTP HMAC secret, separate from encryption and
contact-lookup secrets. Compare digests in constant time. Do not store
or log the plaintext OTP.

### OTP policy

-   Six digits.
-   Five-minute expiry.
-   Maximum five incorrect attempts per challenge.
-   Resend cooldown: 30--60 seconds; default implementation setting 60
    seconds unless changed in configuration.
-   Resending invalidates the previous active challenge.
-   A consumed OTP cannot be reused.
-   An expired challenge cannot be consumed.
-   An exceeded-attempt challenge is locked.
-   Rate limits apply across IP, account/contact lookup key and
    challenge.
-   OTP delivery failures must not expose the code in API responses.
-   Development-only OTP visibility, if needed, must be behind an
    explicit local-only adapter and must never be enabled in
    staging/production.

### Atomic verification

The consume operation must be atomic. Two concurrent verification
requests must not both succeed. Use a conditional update requiring
`status=pending`, `consumedAt=null`, `expiresAt > now`, and
`attempts < maxAttempts`. Incorrect attempts must also be updated
atomically.

## 3.3 Login flow --- finalized Option B

A user may choose either email or mobile as the login identifier, but
the selected contact must already be verified.

1.  User opens `/login`.
2.  Selects email or mobile.
3.  Enters the contact.
4.  Backend normalizes it and calculates its HMAC blind index.
5.  Backend finds the account using the blind index.
6.  Backend checks that the selected contact is verified and the account
    is active.
7.  Backend creates a fresh `login` OTP challenge.
8.  Backend sends OTP to that exact verified contact.
9.  User enters OTP with the verification ID.
10. Backend verifies and consumes the challenge atomically.
11. Backend creates a new session or rotates the relevant session
    credentials.
12. Backend issues a short-lived access token and refresh token.
13. User is redirected to dashboard.

Unknown accounts, unverified contacts and disabled accounts must not
reveal sensitive account existence details. Responses should be designed
to reduce account enumeration. A verified email cannot be used to
receive a phone OTP and vice versa.

### Login flow diagram

``` mermaid
flowchart TD
  A[Login screen] --> B[Choose email or mobile]
  B --> C[Enter contact]
  C --> D[Normalize and HMAC lookup]
  D --> E{Account active and contact verified?}
  E -- No --> F[Generic authentication response]
  E -- Yes --> G[Create fresh login OTP challenge]
  G --> H[Send OTP to selected verified contact]
  H --> I[User submits OTP and verification ID]
  I --> J[Atomic challenge verification]
  J --> K{Valid?}
  K -- No --> L[Reject and enforce attempts]
  K -- Yes --> M[Consume OTP]
  M --> N[Create/rotate session credentials]
  N --> O[Dashboard]
```

## 3.4 Verify the second contact

The user may later verify the contact that was not selected at signup.

1.  User must be authenticated.
2.  User selects "Verify email" or "Verify phone".
3.  Backend checks the new contact is not associated with another
    account.
4.  Backend sends a fresh OTP challenge with purpose `verify_contact`.
5.  User enters OTP.
6.  Backend verifies the challenge and consumes it.
7.  Backend marks that contact verified.
8.  Audit event is recorded.
9.  The verified contact becomes eligible under the notification policy.

Changing an already verified contact must require a secure
re-verification flow. Do not silently replace a verified email or phone.

## 3.5 Locker discovery and reservation flow

1.  User opens locker listing.
2.  Backend returns only active, available lockers at the selected
    location.
3.  User selects a locker.
4.  Frontend requests current locker detail/availability.
5.  Backend atomically reserves the locker for that user.
6.  Reservation expires in 10 minutes.
7.  Backend creates a checkout/booking record in pending-payment state.
8.  Backend calculates price from server-side pricing configuration.
9.  Frontend displays the amount and duration.
10. User proceeds to payment.

Locker availability must account for:

-   Business status.
-   Active/reserved booking.
-   Maintenance state.
-   Device health/online policy.
-   Door state and lock state.
-   Any unresolved recovery state.

Do not show a locker as bookable if its hardware state is unknown or it
is under maintenance.

## 3.6 Initial payment flow

1.  User confirms rental option.
2.  Backend creates payment intent with unique transaction reference and
    idempotency key.
3.  Backend stores amount, currency, pricing snapshot, booking ID and
    expiry.
4.  Backend returns a clearly labelled simulated UPI QR or mock payment
    URI.
5.  User scans or clicks the simulation.
6.  In development, user selects a simulated success/failure action.
7.  Simulation endpoint validates payment intent, amount, expiry,
    ownership and idempotency.
8.  Backend updates payment state.
9.  On confirmed payment, booking is eligible to activate.
10. Backend establishes the storage session and creates an unlock
    authorization/command.
11. Physical device must acknowledge actual execution.
12. User stores belongings and closes door.
13. Door sensor reports closed; lock reports locked.
14. Backend records storage start according to the chosen storage-start
    policy and finalizes the active storage session.

**Important timing policy:** The original product description says the
rental timer starts after payment and the locker unlocks. For a real
physical deployment, the most accurate policy is to start
`storageStartedAt` when the device has successfully granted the initial
deposit access and the system has entered the storage session, with a
documented rule for any delay. Store payment confirmation, unlock
execution and door-close timestamps separately. The scheduled end must
be computed from the explicitly selected storage-start timestamp, not
from a client clock. If the business wants the timer to start
immediately at payment confirmation instead, that must be a deliberate
product policy change.

## 3.7 Retrieval flow

1.  User logs in using a fresh OTP.
2.  User opens "My Lockers".
3.  Backend lists only the user's active/expired occupied bookings.
4.  User selects a booking.
5.  Backend checks booking ownership, locker association, current state
    and outstanding amount.
6.  If not expired and no amount is due, user may request access.
7.  If expired, backend calculates outstanding overdue charge.
8.  User pays overdue charge.
9.  Backend verifies payment and advances `overduePaidThrough`.
10. Backend creates an access session.
11. Backend publishes a time-limited, unique unlock command.
12. Device acknowledges receipt and execution.
13. Door-open event confirms access.
14. User takes belongings.
15. User chooses partial or complete retrieval.
16. User closes door.
17. Device confirms door closed and lock engaged.
18. Backend finalizes access session and applies retrieval choice.

### Partial retrieval

-   Locker remains occupied.
-   Booking is not completed.
-   Original rental does not silently restart.
-   If booking has not expired, original scheduled expiry remains.
-   If booking is expired, paid-through billing applies.
-   Next overdue calculation begins at the last successful
    `overduePaidThrough`.
-   The locker is relocked and retained for the same booking.

### Complete retrieval

-   User chooses complete retrieval.
-   Device reports door closed and lock engaged.
-   Backend verifies there are no unresolved hardware events.
-   Access session is completed.
-   Booking is completed.
-   Locker becomes available only after all release invariants pass.

## 3.8 Expired booking and overdue payment flow

For an expired booking:

1.  Backend detects `scheduledEndAt <= now`.
2.  Booking moves to expired state while locker remains occupied.
3.  Expiry event is recorded once.
4.  Notification is queued for verified contact only.
5.  User logs in with fresh OTP.
6.  User selects the occupied locker.
7.  Backend calculates unpaid overdue duration from `overduePaidThrough`
    or `scheduledEndAt`.
8.  Started hours are calculated with ceiling.
9.  Backend creates an overdue payment intent for the exact amount.
10. User completes simulated payment.
11. Backend confirms payment idempotently.
12. Backend records covered period and advances paid-through timestamp.
13. User is authorized for a retrieval session.
14. User retrieves partially or completely.
15. If partial, the locker remains occupied and next billing starts from
    paid-through time.
16. If complete, physical closure and lock confirmation release the
    locker.

## 3.9 Expiry notification flow

-   A background worker identifies bookings whose scheduled end time has
    passed.
-   The transition to expired must be idempotent.
-   A unique expiry event or deduplication key prevents duplicate
    notification creation.
-   The notification service selects the verified contact only.
-   Contact is decrypted only at the point of delivery and is never
    written to logs.
-   A notification record is created with channel, type, destination
    reference, template/version, status and attempt count.
-   Provider adapter simulates SMS/email in development.
-   Retry transient failures with bounded exponential backoff.
-   Permanent failures are recorded for admin review.
-   Repeated job execution must not create duplicate successful
    notifications.

Suggested message:

> Your Smart Locker booking has expired. Additional charges of ₹20 per
> started hour apply until your belongings are collected. Please sign in
> to arrange retrieval.

Do not put OTPs, full contact data, access tokens or unnecessary
personal information in notifications.

## 3.10 Booking cancellation and abandoned checkout

-   A user may cancel only a booking that is eligible for cancellation.
-   Before payment, cancellation releases the reservation.
-   A pending payment must be checked before release.
-   If payment outcome is uncertain, reconcile before reassigning the
    locker.
-   After storage begins, ordinary cancellation is not equivalent to
    retrieval; the user must retrieve belongings.
-   A locker must not be released while it contains belongings.
-   Refund eligibility and refund rules are a future business policy and
    must not be invented by the implementation.

------------------------------------------------------------------------

# 4. Business rules and invariants

## 4.1 User and identity rules

1.  Name, email and phone are mandatory at signup.
2.  Email and phone must be normalized consistently.
3.  Email and phone must be unique across accounts.
4.  Store encrypted contact values.
5.  Use HMAC blind indexes for lookup and uniqueness.
6.  Signup verifies either email or phone, chosen by the user.
7.  The unselected contact remains unverified.
8.  Login can use either contact only if that contact is verified.
9.  Every login requires a fresh OTP.
10. OTP must be bound to a verification ID, subject, purpose and method.
11. OTP is single-use and expires after five minutes.
12. Maximum five incorrect attempts per challenge.
13. Resend invalidates the previous active challenge.
14. A contact change requires verification.
15. Do not reveal another user's account details in conflict or login
    responses.

## 4.2 Locker rules

1.  Locker number is unique within a location.
2.  Only active, available and healthy lockers can be reserved.
3.  A locker may have at most one current reservation/occupied booking.
4.  Reservation timeout is 10 minutes.
5.  Reservation is atomic.
6.  Maintenance and disabled lockers cannot be booked.
7.  Offline/unknown hardware state must prevent unsafe booking/access
    according to configured policy.
8.  Expiry never releases an occupied locker.
9.  Door open, door closed and lock state are separate signals.
10. Only confirmed final closure and lock engagement allow release.

## 4.3 Pricing rules

    Rental duration   Price
  ----------------- -------
            4 hours     ₹80
            6 hours     ₹90
            8 hours    ₹100
           12 hours    ₹120
           24 hours    ₹240

-   Only these duration values are accepted in V1.
-   Extra charge is ₹20 per started hour.
-   Use `ceil(overdueMilliseconds / 3,600,000) × 20`.
-   If overdue duration is zero or negative, overdue amount is zero.
-   Use server timestamps.
-   Use integer minor currency units (paise) internally where practical,
    or integer INR with explicit currency handling; never floating-point
    money calculations.
-   Save pricing snapshot with booking and payment.
-   Future pricing changes apply to new bookings only.
-   The frontend never supplies the authoritative amount.

## 4.4 Booking rules

1.  Booking must reference a user and locker.
2.  Initial payment must be confirmed before storage activation.
3.  Booking state changes must follow allowed transitions.
4.  Scheduled expiry is computed by the backend.
5.  Expiry job must be idempotent.
6.  A booking may be expired while the locker remains occupied.
7.  Partial retrieval does not complete a booking.
8.  Complete retrieval requires physical confirmation.
9.  One booking must not be attached to multiple lockers at once.
10. A locker must not have multiple current bookings.

## 4.5 Payment rules

1.  Payment amount is generated by backend pricing logic.
2.  Every payment has a unique transaction/reference ID.
3.  Payment confirmation is idempotent.
4.  Payment intent is bound to user and booking.
5.  A simulated success endpoint is development-only.
6.  Payment success must be persisted before related state transition.
7.  Failed payment cannot unlock a locker.
8.  A delayed/duplicate payment event must not activate a stale booking.
9.  Uncertain payment outcomes require reconciliation.
10. Payment records are immutable in their financial facts; corrections
    are recorded as separate adjustments/refunds.
11. Overdue payment advances paid-through timestamp only after confirmed
    success.
12. Payment and unlock are separate events.

## 4.6 Access and physical safety rules

1.  User must own the booking.
2.  Booking must reference the requested locker.
3.  User must be authenticated and session valid.
4.  Required payment must be confirmed.
5.  An access session must be created before sending an unlock command.
6.  Commands must have unique IDs, expiry and replay protection.
7.  Device must be authenticated.
8.  Device acknowledgement is required.
9.  Device execution and door events must be recorded separately.
10. An offline device cannot be represented as successfully unlocked.
11. A timeout creates a recoverable failure state.
12. Door-open state prevents booking completion.
13. Unknown lock/door state prevents release.
14. Emergency access must be controlled, documented and audited.

------------------------------------------------------------------------

# 5. Pricing calculation specification

## 5.1 Rental price lookup

Server-side pricing configuration:

``` js
export const RENTAL_PRICES_INR = Object.freeze({
  4: 80,
  6: 90,
  8: 100,
  12: 120,
  24: 240
});

export const EXTRA_CHARGE_PER_STARTED_HOUR_INR = 20;
```

Validate duration against the exact allowlist. Never accept arbitrary
duration or price from the client.

## 5.2 Overdue calculation

``` js
const HOUR_MS = 60 * 60 * 1000;

function calculateOverdue(startAt, now) {
  const elapsedMs = Math.max(0, now.getTime() - startAt.getTime());
  const startedHours = Math.ceil(elapsedMs / HOUR_MS);
  const amountInr = startedHours * 20;

  return {
    elapsedMs,
    startedHours,
    amountInr
  };
}
```

Production implementation must validate Date values, use an injected
clock for deterministic tests, and use integer monetary values.

## 5.3 Billing baseline

``` text
billingStart =
  overduePaidThrough ?? scheduledEndAt

unpaidOverdue =
  max(0, serverNow - billingStart)

startedHours =
  ceil(unpaidOverdue / 1 hour)

amountDue =
  startedHours * ₹20
```

Persist a billing snapshot for each overdue payment:

-   billing start timestamp
-   calculation timestamp
-   covered-through timestamp
-   started hours
-   rate per hour
-   amount
-   currency
-   booking ID
-   payment ID
-   pricing version

The exact covered-through timestamp is the timestamp established by the
backend payment confirmation policy. Use one consistent policy across
the payment service, booking service and audit record.

------------------------------------------------------------------------

# 6. Database design

Use MongoDB with Mongoose. Collections should be separate by
responsibility. Use ObjectId references and transactions for workflows
that require atomic multi-document state changes.

## 6.1 Collections overview

  Collection               Purpose
  ------------------------ ------------------------------------------------
  `users`                  Registered user profile and encrypted contacts
  `pendingregistrations`   Unverified signup data
  `otpverifications`       Context-bound OTP challenges
  `usersessions`           Refresh token sessions and revocation
  `locations`              Facility/location information
  `lockers`                Locker inventory and current hardware state
  `pricingplans`           Rental rates and overdue rate versions
  `bookings`               Reservation and storage lifecycle
  `payments`               Initial and overdue payment transactions
  `accesssessions`         User access/retrieval attempts
  `iotdevices`             ESP32 identity and health
  `iotcommands`            Unlock/lock commands and acknowledgements
  `notifications`          SMS/email delivery records
  `auditlogs`              Security and business audit trail
  `outboxevents`           Reliable dispatch of jobs/commands/events
  `idempotencyrecords`     Optional persisted API idempotency keys

## 6.2 Entity relationship diagram

``` mermaid
erDiagram
  USERS ||--o{ BOOKINGS : creates
  USERS ||--o{ OTPVERIFICATIONS : verifies
  USERS ||--o{ USERSESSIONS : owns
  USERS ||--o{ NOTIFICATIONS : receives
  USERS ||--o{ AUDITLOGS : generates

  LOCATIONS ||--o{ LOCKERS : contains
  LOCKERS ||--o{ BOOKINGS : assigned_to

  BOOKINGS ||--o{ PAYMENTS : has
  BOOKINGS ||--o{ ACCESSSESSIONS : records
  BOOKINGS ||--o{ NOTIFICATIONS : triggers

  LOCKERS ||--o{ IOTCOMMANDS : receives
  IOTDEVICES ||--o{ IOTCOMMANDS : executes

  PENDINGREGISTRATIONS ||--o{ OTPVERIFICATIONS : verifies

  BOOKINGS ||--o{ OUTBOXEVENTS : emits
```

## 6.3 Users schema

``` js
{
  _id: ObjectId,

  name: String,

  emailEncrypted: {
    ciphertext: String,
    iv: String,
    authTag: String,
    keyVersion: Number
  },
  emailLookupHash: String,

  phoneEncrypted: {
    ciphertext: String,
    iv: String,
    authTag: String,
    keyVersion: Number
  },
  phoneLookupHash: String,

  emailVerified: Boolean,
  phoneVerified: Boolean,

  role: "user" | "admin" | "operator",
  status: "active" | "suspended" | "disabled",

  createdAt: Date,
  updatedAt: Date
}
```

Indexes:

-   Unique `emailLookupHash` when present.
-   Unique `phoneLookupHash` when present.
-   Index on `status`.
-   Index on `createdAt`.

Use partial unique indexes for optional fields. Define how null/missing
values are stored; do not create duplicate-null index conflicts.
Normalize email consistently (trim and lowercase). Normalize phone into
a canonical E.164 representation using a maintained phone-number library
and an explicit country context.

## 6.4 PendingRegistrations schema

``` js
{
  _id: ObjectId,

  name: String,

  emailEncrypted: EncryptedValue,
  emailLookupHash: String,

  phoneEncrypted: EncryptedValue,
  phoneLookupHash: String,

  verificationMethod: "email" | "phone",

  status: "pending" | "verified" | "expired" | "consumed",

  expiresAt: Date,
  createdAt: Date,
  updatedAt: Date
}
```

-   TTL index on `expiresAt`.
-   Application must check expiry because TTL deletion is asynchronous.
-   Store encrypted contact values and blind indexes.
-   Do not create a full user until chosen signup verification succeeds.

## 6.5 OTPVerifications schema

``` js
{
  _id: ObjectId,

  verificationId: String, // UUID, unique

  subjectType: "pending_registration" | "user",
  subjectId: ObjectId,

  purpose: "signup" | "login" | "verify_contact",
  method: "email" | "phone",

  contactLookupHash: String,

  otpHash: String,

  attempts: Number,
  maxAttempts: Number,

  status: "pending" | "consumed" | "expired" | "locked" | "invalidated",

  expiresAt: Date,
  consumedAt: Date,

  createdAt: Date,
  updatedAt: Date
}
```

Indexes:

-   Unique `verificationId`.
-   Index for subject, purpose and status.
-   TTL index on `expiresAt` for cleanup only.
-   Ensure only the current active challenge is valid through atomic
    invalidation/creation logic.

## 6.6 UserSessions schema

``` js
{
  _id: ObjectId,
  userId: ObjectId,

  refreshTokenHash: String,
  tokenFamilyId: String,

  createdAt: Date,
  expiresAt: Date,
  lastUsedAt: Date,
  revokedAt: Date,

  userAgentHash: String,
  ipAddressHash: String
}
```

Store refresh token hashes, not raw refresh tokens. Rotate refresh
tokens and detect reuse. Apply retention and privacy policies to session
metadata.

## 6.7 Locations schema

``` js
{
  _id: ObjectId,

  name: String,
  address: String,

  city: String,
  state: String,
  country: String,

  timezone: "Asia/Kolkata",

  isActive: Boolean,

  createdAt: Date,
  updatedAt: Date
}
```

## 6.8 Lockers schema

``` js
{
  _id: ObjectId,

  locationId: ObjectId,

  lockerNumber: String,
  size: "small" | "medium" | "large",

  status:
    "available" |
    "reserved" |
    "occupied" |
    "maintenance" |
    "disabled" |
    "recovery_required",

  hardware: {
    deviceId: String,
    connectionStatus: "online" | "offline" | "unknown",
    lockState: "locked" | "unlocked" | "unknown",
    doorState: "open" | "closed" | "unknown",
    lastHeartbeatAt: Date,
    lastEventAt: Date
  },

  isActive: Boolean,

  createdAt: Date,
  updatedAt: Date
}
```

Indexes:

-   Unique compound `{ locationId, lockerNumber }`.
-   Index `{ locationId, status, size }`.
-   Unique `hardware.deviceId` when present.

Do not rely only on `status` to infer physical state. Track business
availability and hardware state separately.

## 6.9 PricingPlans schema

``` js
{
  _id: ObjectId,

  version: Number,
  currency: "INR",

  durationOptions: [
    { hours: 4, amountMinor: 8000 },
    { hours: 6, amountMinor: 9000 },
    { hours: 8, amountMinor: 10000 },
    { hours: 12, amountMinor: 12000 },
    { hours: 24, amountMinor: 24000 }
  ],

  extraChargePerStartedHourMinor: 2000,
  billingRule: "started_hour",

  isActive: Boolean,

  effectiveFrom: Date,
  effectiveTo: Date,

  createdAt: Date,
  updatedAt: Date
}
```

Use integer paise internally. A booking stores a pricing snapshot so a
later pricing update does not alter an existing booking.

## 6.10 Bookings schema

``` js
{
  _id: ObjectId,

  bookingNumber: String,

  userId: ObjectId,
  lockerId: ObjectId,
  locationId: ObjectId,

  durationHours: Number,

  pricingSnapshot: {
    pricingPlanId: ObjectId,
    pricingVersion: Number,
    baseAmountMinor: Number,
    currency: "INR",
    extraChargePerStartedHourMinor: Number
  },

  status:
    "reserved" |
    "pending_payment" |
    "active" |
    "expired" |
    "completed" |
    "cancelled" |
    "recovery_required",

  reservedAt: Date,
  reservationExpiresAt: Date,

  initialPaymentId: ObjectId,
  paymentCompletedAt: Date,

  storageStartedAt: Date,
  scheduledEndAt: Date,
  expiredAt: Date,

  overduePaidThrough: Date,

  completedAt: Date,
  cancelledAt: Date,

  createdAt: Date,
  updatedAt: Date
}
```

Indexes:

-   Unique `bookingNumber`.
-   `{ userId, status, createdAt }`.
-   `{ lockerId, status }`.
-   `{ status, scheduledEndAt }`.
-   `{ status, reservationExpiresAt }`.

Enforce one current booking per locker. A partial unique index on
`lockerId` for states that hold the locker can be used if the state
model is carefully maintained. Consider representing the current
reservation/occupancy pointer on the locker and changing it atomically
in a transaction. The database invariant must prevent double booking
even under concurrent requests.

## 6.11 Payments schema

``` js
{
  _id: ObjectId,

  bookingId: ObjectId,
  userId: ObjectId,

  transactionId: String,
  idempotencyKey: String,

  paymentType: "initial_booking" | "overdue_charge",

  amountMinor: Number,
  currency: "INR",

  provider: "simulated_upi",

  status: "created" | "pending" | "succeeded" | "failed" | "expired" | "reconciliation_required",

  billingSnapshot: {
    billingStartAt: Date,
    calculatedAt: Date,
    coveredThroughAt: Date,
    startedHours: Number,
    ratePerStartedHourMinor: Number
  },

  providerReference: String,

  paidAt: Date,
  createdAt: Date,
  updatedAt: Date
}
```

Indexes:

-   Unique `transactionId`.
-   Unique or scoped unique idempotency key.
-   `{ bookingId, createdAt }`.
-   `{ userId, createdAt }`.
-   `{ status, createdAt }`.

Never allow the client to choose `amountMinor`, `status=succeeded` or
`paidAt`.

## 6.12 AccessSessions schema

``` js
{
  _id: ObjectId,

  bookingId: ObjectId,
  userId: ObjectId,
  lockerId: ObjectId,

  accessType: "initial_deposit" | "retrieval",

  status:
    "authorized" |
    "command_pending" |
    "unlock_confirmed" |
    "door_open" |
    "door_closed" |
    "locked" |
    "completed" |
    "failed" |
    "timed_out" |
    "recovery_required",

  retrievalType: "partial" | "complete" | null,

  authorizedAt: Date,
  openedAt: Date,
  closedAt: Date,
  lockedAt: Date,

  iotCommandId: String,

  failureCode: String,

  createdAt: Date,
  updatedAt: Date
}
```

An access session records the full physical access lifecycle. A request
to unlock is not proof that the locker opened.

## 6.13 IoTDevices schema

``` js
{
  _id: ObjectId,

  deviceId: String,
  lockerId: ObjectId,

  credentialReference: String,
  status: "online" | "offline" | "disabled" | "provisioning",

  firmwareVersion: String,

  lastHeartbeatAt: Date,
  lastSeenAt: Date,

  createdAt: Date,
  updatedAt: Date
}
```

Do not store a reusable plaintext device secret in ordinary application
records. Prefer per-device certificates or secrets managed through a
secure provisioning process. If a shared secret is unavoidable for a
prototype, store only a protected hash where verification permits, and
plan rotation.

## 6.14 IoTCommands schema

``` js
{
  _id: ObjectId,

  commandId: String,
  deviceId: String,
  lockerId: ObjectId,
  accessSessionId: ObjectId,

  command: "UNLOCK" | "LOCK" | "STATUS",

  payloadVersion: Number,

  status:
    "pending" |
    "published" |
    "acknowledged" |
    "executed" |
    "failed" |
    "expired",

  issuedAt: Date,
  expiresAt: Date,
  acknowledgedAt: Date,
  executedAt: Date,

  resultCode: String,

  createdAt: Date,
  updatedAt: Date
}
```

Indexes:

-   Unique `commandId`.
-   `{ deviceId, status, createdAt }`.
-   `{ status, expiresAt }`.

## 6.15 Notifications schema

``` js
{
  _id: ObjectId,

  userId: ObjectId,
  bookingId: ObjectId,

  channel: "email" | "sms",
  type: "booking_expired" | "booking_reminder" | "payment_receipt",

  recipientLookupHash: String,
  recipientEncryptedSnapshot: EncryptedValue,

  templateKey: String,
  templateVersion: Number,

  status: "pending" | "sent" | "failed" | "retrying",

  attempts: Number,
  nextAttemptAt: Date,

  providerReference: String,
  sentAt: Date,
  failedAt: Date,
  failureCode: String,

  deduplicationKey: String,

  createdAt: Date,
  updatedAt: Date
}
```

Do not store OTP content or unredacted sensitive message content in
ordinary logs. Notification records should contain only what is
necessary for delivery, audit and retry.

## 6.16 AuditLogs schema

``` js
{
  _id: ObjectId,

  actorType: "user" | "admin" | "system" | "device",
  actorId: ObjectId,

  action: String,

  resourceType: String,
  resourceId: ObjectId,

  requestId: String,

  ipAddressHash: String,

  metadata: Object,

  createdAt: Date
}
```

Audit records must not contain OTPs, raw JWTs, encryption keys, raw
refresh tokens, full payment secrets or unnecessary unencrypted
contacts. Restrict write access and consider append-only
storage/retention controls.

## 6.17 OutboxEvents schema

Use a transactional outbox so database state changes and the intention
to dispatch a notification, MQTT command or background job are committed
consistently.

``` js
{
  _id: ObjectId,

  eventId: String,
  eventType: String,

  aggregateType: String,
  aggregateId: ObjectId,

  payload: Object,

  status: "pending" | "processing" | "published" | "failed",

  attempts: Number,
  nextAttemptAt: Date,

  createdAt: Date,
  publishedAt: Date
}
```

The payload must not contain secrets or unnecessary personal data.

------------------------------------------------------------------------

# 7. Backend architecture and project structure

## 7.1 Architecture decision

Use a **modular monolith** for V1.

Reasons:

-   The product is at an early stage.
-   Core booking/payment/access rules need to evolve together.
-   A modular monolith is easier to test and operate than multiple
    microservices.
-   Transactions and state consistency are easier to manage.
-   Integrations can be isolated behind adapters.
-   Modules can be extracted later if actual scale and team structure
    justify it.

Do not create microservices merely for appearance.

## 7.2 Technology stack

  Layer                Technology
  -------------------- -----------------------------------------------------
  Runtime              Current supported Node.js LTS
  HTTP                 Express.js 5
  Language             JavaScript ES modules
  Database             MongoDB replica set for transactions
  ODM                  Mongoose
  Validation           Zod
  Queue                BullMQ
  Queue store          Redis
  OTP generation       Node.js Crypto
  OTP hashing          HMAC-SHA256
  Contact encryption   AES-256-GCM
  Contact lookup       HMAC-SHA256 blind indexes
  Authentication       JWT access + rotating refresh tokens
  Password hashing     Argon2id if passwords are introduced
  MQTT                 Maintained MQTT client and broker
  API docs             OpenAPI/Swagger
  Logging              Pino
  Security headers     Helmet
  Rate limiting        Redis-backed distributed limiter
  Testing              Vitest or Jest + Supertest
  Containerization     Docker
  CI/CD                Automated lint, test, build and deployment pipeline

Use current supported, mutually compatible package versions. Pin
dependencies with a lockfile. Do not use deprecated APIs, unmaintained
libraries or unnecessary dependencies.

## 7.3 Folder structure

``` text
smart-locker/
├── apps/
│   ├── api/
│   │   ├── src/
│   │   │   ├── app.js
│   │   │   ├── server.js
│   │   │   ├── config/
│   │   │   │   ├── env.js
│   │   │   │   ├── database.js
│   │   │   │   ├── redis.js
│   │   │   │   ├── mqtt.js
│   │   │   │   └── security.js
│   │   │   ├── modules/
│   │   │   │   ├── auth/
│   │   │   │   ├── users/
│   │   │   │   ├── otp/
│   │   │   │   ├── locations/
│   │   │   │   ├── lockers/
│   │   │   │   ├── pricing/
│   │   │   │   ├── bookings/
│   │   │   │   ├── payments/
│   │   │   │   ├── access/
│   │   │   │   ├── iot/
│   │   │   │   ├── notifications/
│   │   │   │   ├── admin/
│   │   │   │   └── audit/
│   │   │   ├── middleware/
│   │   │   ├── integrations/
│   │   │   │   ├── sms/
│   │   │   │   ├── email/
│   │   │   │   ├── payments/
│   │   │   │   └── mqtt/
│   │   │   ├── jobs/
│   │   │   ├── workers/
│   │   │   ├── shared/
│   │   │   │   ├── errors/
│   │   │   │   ├── logger/
│   │   │   │   ├── crypto/
│   │   │   │   ├── time/
│   │   │   │   └── constants/
│   │   │   └── docs/
│   │   └── tests/
│   │       ├── unit/
│   │       ├── integration/
│   │       ├── concurrency/
│   │       └── e2e/
│   │
│   ├── web/
│   │   ├── src/
│   │   │   ├── app/
│   │   │   ├── routes/
│   │   │   ├── features/
│   │   │   │   ├── auth/
│   │   │   │   ├── dashboard/
│   │   │   │   ├── lockers/
│   │   │   │   ├── booking/
│   │   │   │   ├── payment/
│   │   │   │   ├── retrieval/
│   │   │   │   ├── history/
│   │   │   │   ├── notifications/
│   │   │   │   ├── profile/
│   │   │   │   └── admin/
│   │   │   ├── components/
│   │   │   ├── services/
│   │   │   ├── hooks/
│   │   │   ├── layouts/
│   │   │   └── styles/
│   │   └── tests/
│   │
│   └── workers/
│       └── src/
│           ├── worker.js
│           ├── booking-expiry.worker.js
│           ├── reservation-cleanup.worker.js
│           ├── notification.worker.js
│           ├── outbox.worker.js
│           └── payment-reconciliation.worker.js
│
├── firmware/
│   └── esp32/
│       ├── src/
│       ├── include/
│       ├── platformio.ini
│       └── README.md
│
├── infrastructure/
│   ├── docker/
│   ├── mqtt/
│   ├── scripts/
│   └── deployment/
│
├── docs/
│   ├── architecture/
│   ├── api/
│   ├── flows/
│   ├── security/
│   ├── operations/
│   └── decisions/
│
├── .env.example
├── .gitignore
├── package.json
├── package-lock.json
├── docker-compose.yml
└── README.md
```

Each backend feature module should generally contain:

``` text
module-name/
├── module.routes.js
├── module.controller.js
├── module.service.js
├── module.repository.js
├── module.model.js
├── module.schema.js
├── module.constants.js
└── module.test.js
```

Not every module needs every file. Keep the design simple and avoid
empty abstraction layers.

## 7.4 Layer responsibilities

-   **Routes:** HTTP route definitions and middleware wiring.
-   **Controllers:** Parse validated request data, call services and
    format responses.
-   **Services:** Business rules and workflow orchestration.
-   **Repositories:** Database access and query composition where
    useful.
-   **Models:** Mongoose schemas, indexes and persistence constraints.
-   **Validation schemas:** Zod request validation.
-   **Integrations/adapters:** SMS, email, payment and MQTT provider
    interfaces.
-   **Workers/jobs:** Expiry, retry, cleanup, reconciliation and outbox
    processing.
-   **Middleware:** Authentication, authorization, rate limiting,
    request ID, validation and error handling.
-   **Shared utilities:** Pure helpers for time, crypto, formatting and
    constants.

Business logic must not live in route handlers. Controllers must not
directly execute MongoDB workflows or publish MQTT commands.

------------------------------------------------------------------------

# 8. API specification

Base path: `/api/v1`

All endpoints must use validation, authentication/authorization as
appropriate, request IDs, consistent response envelopes and centralized
error handling.

## 8.1 Authentication APIs

  --------------------------------------------------------------------------------
  Method                  Endpoint                         Purpose
  ----------------------- -------------------------------- -----------------------
  POST                    `/auth/signup`                   Start registration and
                                                           send selected OTP

  POST                    `/auth/signup/verify`            Verify signup challenge
                                                           and create user

  POST                    `/auth/signup/resend-otp`        Resend signup OTP

  POST                    `/auth/login`                    Request login OTP using
                                                           email or mobile

  POST                    `/auth/login/verify`             Verify login OTP and
                                                           create session

  POST                    `/auth/login/resend-otp`         Resend login OTP

  POST                    `/auth/refresh`                  Rotate refresh token
                                                           and issue access token

  POST                    `/auth/logout`                   Revoke current session

  POST                    `/auth/logout-all`               Revoke all user
                                                           sessions

  GET                     `/auth/me`                       Return authenticated
                                                           profile

  POST                    `/auth/contact/verify`           Start verification for
                                                           second contact

  POST                    `/auth/contact/verify/confirm`   Complete second-contact
                                                           verification
  --------------------------------------------------------------------------------

Example signup request:

``` json
{
  "name": "Rahul",
  "email": "rahul@example.com",
  "phone": "+919876543210",
  "verificationMethod": "phone"
}
```

Example signup response:

``` json
{
  "success": true,
  "message": "Verification code sent",
  "data": {
    "verificationId": "uuid",
    "expiresInSeconds": 300,
    "resendAfterSeconds": 60
  },
  "requestId": "request-id"
}
```

Never return OTP plaintext.

Example verify request:

``` json
{
  "verificationId": "uuid",
  "otp": "482193"
}
```

## 8.2 Locker APIs

  Method   Endpoint                            Purpose
  -------- ----------------------------------- -----------------------
  GET      `/locations`                        List active locations
  GET      `/locations/:locationId/lockers`    List bookable lockers
  GET      `/lockers/:lockerId`                Locker details
  GET      `/lockers/:lockerId/availability`   Recheck availability

Admin:

  Method   Endpoint                      Purpose
  -------- ----------------------------- ---------------------------
  POST     `/admin/locations`            Create location
  PATCH    `/admin/locations/:id`        Update location
  POST     `/admin/lockers`              Create locker
  PATCH    `/admin/lockers/:id`          Update locker
  PATCH    `/admin/lockers/:id/status`   Maintenance/disable state

## 8.3 Pricing and booking APIs

  -------------------------------------------------------------------------------
  Method                  Endpoint                        Purpose
  ----------------------- ------------------------------- -----------------------
  GET                     `/pricing`                      Get current public
                                                          rental options

  POST                    `/bookings/quote`               Validate
                                                          locker/duration and
                                                          quote

  POST                    `/bookings`                     Atomically reserve
                                                          locker and create
                                                          checkout

  GET                     `/bookings/my`                  User's bookings

  GET                     `/bookings/:bookingId`          Booking details with
                                                          ownership check

  POST                    `/bookings/:bookingId/cancel`   Cancel eligible pending
                                                          booking
  -------------------------------------------------------------------------------

Quote and booking creation must revalidate price and availability on the
server. A quote is not a reservation.

## 8.4 Payment APIs

  Method   Endpoint                          Purpose
  -------- --------------------------------- ---------------------------------
  POST     `/payments/initialize`            Create simulated payment intent
  GET      `/payments/:paymentId`            Retrieve payment status
  GET      `/bookings/:bookingId/payments`   Payment history

Development-only simulation endpoints:

  ---------------------------------------------------------------------------------------------
  Method                  Endpoint                                      Purpose
  ----------------------- --------------------------------------------- -----------------------
  POST                    `/dev/payments/:paymentId/simulate-success`   Simulate successful
                                                                        payment

  POST                    `/dev/payments/:paymentId/simulate-failure`   Simulate failed payment
  ---------------------------------------------------------------------------------------------

These endpoints must be excluded from production builds or blocked by
deployment configuration and server-side environment guards. A public
production client must never be able to mark its own payment as
successful.

## 8.5 Access and retrieval APIs

  ---------------------------------------------------------------------------------------------
  Method                  Endpoint                                      Purpose
  ----------------------- --------------------------------------------- -----------------------
  GET                     `/access/my-lockers`                          List user's occupied
                                                                        lockers

  POST                    `/access/quote`                               Calculate current
                                                                        overdue amount

  POST                    `/access/unlock`                              Authorize and request
                                                                        locker unlock

  GET                     `/access/:accessSessionId/status`             Poll access/device
                                                                        status

  POST                    `/access/:accessSessionId/retrieval-choice`   Submit partial/complete
                                                                        choice

  GET                     `/access/:accessSessionId`                    Get access session
                                                                        details
  ---------------------------------------------------------------------------------------------

The retrieval-choice API records the user's intention. It must not mark
a booking completed until physical closure and lock confirmation have
been received and validated.

## 8.6 IoT APIs and MQTT events

Preferred device control is MQTT, not a publicly exposed unauthenticated
HTTP unlock endpoint.

HTTP device endpoints, if retained for provisioning/heartbeat, must use
device authentication:

  --------------------------------------------------------------------------------
  Method                  Endpoint                         Purpose
  ----------------------- -------------------------------- -----------------------
  POST                    `/iot/heartbeat`                 Device heartbeat

  POST                    `/iot/events`                    Authenticated device
                                                           event ingestion

  POST                    `/iot/commands/:commandId/ack`   Command acknowledgement
                                                           if HTTP adapter is used
  --------------------------------------------------------------------------------

## 8.7 Admin APIs

  ----------------------------------------------------------------------------------
  Method                  Endpoint                           Purpose
  ----------------------- ---------------------------------- -----------------------
  GET                     `/admin/dashboard`                 Operational metrics

  GET                     `/admin/users`                     User listing with
                                                             masked fields

  GET                     `/admin/bookings`                  Booking management

  GET                     `/admin/payments`                  Payment management

  GET                     `/admin/lockers`                   Locker inventory

  GET                     `/admin/devices`                   Device status

  GET                     `/admin/notifications`             Delivery monitoring

  GET                     `/admin/audit-logs`                Audit review

  POST                    `/admin/pricing-plans`             Create pricing version

  PATCH                   `/admin/pricing-plans/:id`         Update future pricing

  POST                    `/admin/lockers/:id/maintenance`   Enter controlled
                                                             maintenance

  POST                    `/admin/recovery-cases`            Create audited recovery
                                                             case
  ----------------------------------------------------------------------------------

## 8.8 Standard API response

Success:

``` json
{
  "success": true,
  "message": "Operation completed",
  "data": {},
  "requestId": "request-id"
}
```

Error:

``` json
{
  "success": false,
  "message": "The request could not be completed",
  "error": {
    "code": "OTP_INVALID",
    "details": []
  },
  "requestId": "request-id"
}
```

Do not expose stack traces, database messages, encryption internals or
secrets in production responses.

------------------------------------------------------------------------

# 9. Frontend UI and navigation specification

Build a responsive, mobile-first React application. The main user
journey should be simple enough for a first-time visitor at a station,
mall or public facility.

## 9.1 Public/authentication pages

  -----------------------------------------------------------------------
  Route                   Screen                  Main contents
  ----------------------- ----------------------- -----------------------
  `/`                     Landing page            Product explanation,
                                                  locations, sign in/sign
                                                  up

  `/signup`               Registration            Name, email, phone,
                                                  verification choice

  `/verify-signup`        OTP verification        Six-digit input,
                                                  countdown, resend

  `/login`                Login                   Choose email or mobile,
                                                  enter identifier

  `/verify-login`         Login OTP               OTP input, resend,
                                                  expiry

  `/terms`                Terms                   Terms and conditions

  `/privacy`              Privacy                 Privacy notice
  -----------------------------------------------------------------------

## 9.2 Authenticated user pages

  --------------------------------------------------------------------------------
  Route                            Screen                  Main contents
  -------------------------------- ----------------------- -----------------------
  `/dashboard`                     User dashboard          Available lockers,
                                                           active booking, alerts

  `/locations`                     Locations               Active facility list

  `/locations/:id/lockers`         Locker list             Available locker cards
                                                           and filters

  `/lockers/:lockerId`             Locker details          Size, status, price
                                                           options

  `/booking/checkout/:bookingId`   Checkout                Duration, amount,
                                                           10-minute timer

  `/payment/:paymentId`            Payment                 Simulated UPI QR,
                                                           status, instructions

  `/booking/:bookingId/success`    Confirmation            Locker number, booking
                                                           and access instructions

  `/my-lockers`                    My lockers              Active, expired,
                                                           retrieval-required

  `/my-lockers/:bookingId`         Booking detail          Timer, expiry, amount
                                                           due, access action

  `/retrieval/:bookingId`          Retrieval               OTP/session state,
                                                           access status, choices

  `/history`                       History                 Completed/cancelled
                                                           bookings

  `/payments`                      Payment history         Initial and overdue
                                                           payment records

  `/notifications`                 Notifications           Notification history

  `/profile`                       Profile                 Name, masked contacts,
                                                           verification status

  `/profile/verify-contact`        Contact verification    Verify second contact
  --------------------------------------------------------------------------------

## 9.3 Admin pages

  Route                    Screen
  ------------------------ ---------------------------------
  `/admin`                 Admin dashboard
  `/admin/locations`       Location management
  `/admin/lockers`         Locker inventory and status
  `/admin/bookings`        Booking management
  `/admin/payments`        Payment and reconciliation
  `/admin/users`           User management
  `/admin/devices`         ESP32 health and commands
  `/admin/notifications`   Delivery failures and retries
  `/admin/pricing`         Pricing versions
  `/admin/reports`         Utilization and revenue reports
  `/admin/audit-logs`      Security and operations audit
  `/admin/recovery`        Controlled recovery cases

## 9.4 Signup screen requirements

-   Name field.
-   Email field.
-   Phone field with country code.
-   Radio/segmented selection: email OTP or mobile OTP.
-   Clear explanation that only the selected contact is verified
    initially.
-   Validation messages.
-   Submit button with loading/disabled state.
-   Terms/privacy consent where required.

## 9.5 OTP screen requirements

-   Six digit input.
-   Verification ID retained by frontend.
-   Countdown showing remaining validity.
-   Resend disabled until cooldown completes.
-   Clear invalid/expired/locked feedback.
-   No display of OTP in production.
-   Avoid leaking whether a different user's contact exists.

## 9.6 Locker listing requirements

Each card should display:

-   Locker number.
-   Size.
-   Location.
-   Availability.
-   Hardware readiness indicator where appropriate.
-   Select action.

Do not show an occupied, maintenance, disabled or unknown-hardware
locker as available.

## 9.7 Duration and checkout screen

Show only:

-   4 hours --- ₹80.
-   6 hours --- ₹90.
-   8 hours --- ₹100.
-   12 hours --- ₹120.
-   24 hours --- ₹240.

Display: - Selected locker. - Selected duration. - Total amount. -
10-minute checkout reservation countdown. - Payment action. -
Expiry/cancellation explanation.

## 9.8 Payment screen

For development: - Clearly label payment as simulated. - Show QR/mock
UPI URI. - Show transaction reference and amount. - Show
pending/success/failure state. - Never imply that a mock QR transfers
real money. - Never let frontend status alone activate a booking.

## 9.9 User dashboard

Display: - Available locations/lockers. - Active bookings. - Expired
bookings requiring retrieval. - Locker number and size. - Scheduled
expiry and remaining time. - Amount due if expired. - Quick "Access
locker" action. - Recent payments and notifications.

## 9.10 Retrieval screen

Show: - Booking and locker reference. - Expiry state. - Outstanding
amount and calculation details. - Payment action if required. - Unlock
request progress. - Hardware acknowledgement state. -
Door-open/door-closed state. - "Take all belongings" and "Take some
belongings" choices after access. - Clear instruction to close the
door. - Completion confirmation only after physical confirmation.

## 9.11 UI states

Every data-driven screen must handle:

-   Loading.
-   Empty state.
-   Success.
-   Validation error.
-   Network error.
-   Unauthorized/expired session.
-   Payment pending.
-   Payment failed.
-   Device offline.
-   Unlock timeout.
-   Door still open.
-   Recovery required.

Do not leave a button active while a duplicate payment/access request is
in progress. Use idempotency keys for relevant actions.

------------------------------------------------------------------------

# 10. IoT and ESP32 architecture

## 10.1 Architecture

``` mermaid
flowchart TD
  WEB[React web app] --> API[Node.js API]
  API --> AUTH[Authentication and authorization]
  AUTH --> ACCESS[Access authorization service]
  ACCESS --> DB[(MongoDB)]
  ACCESS --> OUTBOX[Transactional outbox]
  OUTBOX --> PUB[MQTT publisher]
  PUB --> BROKER[MQTT broker]
  BROKER --> ESP[ESP32]
  ESP --> DRIVER[Relay/MOSFET driver]
  DRIVER --> LOCK[Electronic lock]
  SENSOR[Door sensor] --> ESP
  ESP --> BROKER
  BROKER --> EVENTS[IoT event consumer]
  EVENTS --> DB
```

## 10.2 Hardware required

For a prototype:

-   ESP32 development board.
-   Electronic solenoid lock or appropriately selected electric cabinet
    lock.
-   Relay module or MOSFET driver suitable for the lock's
    voltage/current.
-   External power supply sized for the lock and electronics.
-   Flyback protection for inductive loads where required.
-   Door magnetic reed switch or suitable position sensor.
-   Wiring, connectors and enclosure.
-   Optional status LED/buzzer.
-   Wi-Fi network.
-   MQTT broker.
-   Optional manual emergency release.

Never drive a solenoid directly from an ESP32 GPIO. GPIO is a logic
signal, not a lock power source. Select the driver, supply, protection
and wiring based on the lock's electrical specifications. Use a
qualified hardware review before public deployment.

## 10.3 Device provisioning and identity

Each ESP32 must have its own identity and credentials.

-   No shared default credentials across all devices.
-   Secure provisioning process.
-   Credentials stored using suitable device protections.
-   MQTT over TLS.
-   Broker ACL restricts each device to its own command and event
    topics.
-   Rotate/revoke device credentials.
-   Firmware update process must be controlled and authenticated.
-   Do not expose broker credentials in frontend code.

## 10.4 MQTT topic design

Example topic pattern:

``` text
smart-locker/v1/devices/{deviceId}/commands
smart-locker/v1/devices/{deviceId}/ack
smart-locker/v1/devices/{deviceId}/events
smart-locker/v1/devices/{deviceId}/heartbeat
```

Device ACL must prevent one device from subscribing to another device's
commands or publishing as another device.

## 10.5 Unlock command example

``` json
{
  "schemaVersion": 1,
  "commandId": "uuid",
  "deviceId": "ESP32_L001",
  "lockerId": "locker-object-id",
  "accessSessionId": "access-session-id",
  "command": "UNLOCK",
  "issuedAt": "server timestamp",
  "expiresAt": "short command expiry",
  "durationSeconds": 10,
  "nonce": "unique nonce"
}
```

The device must validate: - Its own device ID. - Message
authentication/transport identity. - Schema version. - Command ID and
replay protection. - Expiry. - Allowed command. - Safe actuator timeout.

## 10.6 Device acknowledgements and events

Examples: - command received - unlock executed - unlock failed - door
opened - door closed - lock engaged - heartbeat - low power/fault where
supported

An acknowledgement that a message was received is not the same as
successful physical execution. Record separate received, executed, door
and lock events.

## 10.7 Hardware safety

-   Default lock state must be chosen based on the physical lock type
    and life-safety review.
-   Provide controlled emergency access.
-   Detect door open/closed independently of command response.
-   Avoid indefinite unlock.
-   Handle power failure and network loss.
-   Use appropriate fuse/protection and rated power components.
-   Do not make a locker available when sensor/lock state is unknown.
-   A public deployment requires electrical, mechanical, fire and
    facility safety review.
-   Test tampering, power cycling, network interruption and sensor
    faults.

## 10.8 Simulated IoT adapter

Before physical ESP32 integration, implement an `IoTProvider` interface:

-   `requestUnlock(command)`
-   `requestLock(command)`
-   `getDeviceStatus(deviceId)`
-   `handleDeviceEvent(event)`

Provide: - `SimulatedIoTProvider` for local tests. - `MqttIoTProvider`
for real hardware.

The same access service must use either provider. The simulation must
explicitly mark device events as simulated and must never be confused
with physical confirmation in production.

------------------------------------------------------------------------

# 11. Security architecture

## 11.1 Contact encryption and blind indexes

Email and phone must be encrypted using AES-256-GCM.

Store an envelope such as:

``` js
{
  ciphertext: "base64",
  iv: "base64",
  authTag: "base64",
  keyVersion: 1
}
```

Use a fresh random IV for every encryption operation. Store the
authentication tag. Verify the tag on decryption.

For searching and uniqueness: - Normalize the email/phone first. -
Compute HMAC-SHA256 using a dedicated lookup key. - Store the HMAC
output as the blind index. - Use separate keys for encryption, OTP
hashing and lookup. - Do not use a plain unsalted SHA hash for
phone/email lookup. - Do not expose blind indexes through APIs. - Plan
key versioning and rotation. - Do not commit keys to Git. - Use a
secrets manager or protected deployment secret store.

Encryption does not replace TLS, access control, database security,
secure backups or application authorization.

## 11.2 OTP protection

-   Crypto-secure six-digit random values.
-   HMAC stored digest, not plaintext.
-   Context-bound verification ID.
-   Five-minute expiry.
-   Five attempts.
-   Resend cooldown.
-   Previous challenge invalidation.
-   Atomic single-use consumption.
-   Rate limiting by IP, challenge and account/contact lookup.
-   No OTP in logs, traces or API response.
-   No production debug OTP.
-   Delivery provider credentials stored as secrets.

## 11.3 JWT and session security

-   Access token lifetime: recommended 10--15 minutes.
-   Refresh token: longer but bounded lifetime, rotated on every
    refresh.
-   Store refresh token hashes.
-   Detect refresh token reuse.
-   Revoke session on logout.
-   Support logout all sessions.
-   Use secure, HttpOnly, SameSite cookies for browser refresh
    credentials.
-   Protect cookie-authenticated state-changing requests against CSRF.
-   Use strict CORS allowlist.
-   Never store long-lived refresh credentials in localStorage.
-   Validate issuer, audience, expiry and signing algorithm.
-   Keep signing keys out of source control.

## 11.4 Authorization

-   Role-based access control for user/admin/operator.
-   Resource ownership check for every booking, payment and access
    request.
-   Do not trust IDs supplied by frontend without checking ownership.
-   Admin APIs require explicit admin role and audit.
-   IoT devices use separate device authentication, not user JWTs.
-   Apply least privilege to database, Redis, MQTT and deployment
    credentials.

## 11.5 Rate limiting

Initial configurable defaults:

  -----------------------------------------------------------------------
  Operation                           Suggested limit
  ----------------------------------- -----------------------------------
  Signup initiation                   5 per hour per IP

  Login OTP request                   5 per 15 minutes per
                                      account/contact

  OTP verification                    5 attempts per challenge

  OTP resend                          1 per 60 seconds per
                                      challenge/contact

  Unlock request                      5 per 10 minutes per booking, plus
                                      stricter state-based limits

  Payment simulation                  5 per minute per account in
                                      development

  General API                         100 requests per minute per
                                      authenticated user
  -----------------------------------------------------------------------

Use distributed Redis-backed limits. Apply multiple dimensions (IP,
contact HMAC, account, challenge, device and booking as appropriate).
Tune limits based on threat modelling and observed traffic. Return HTTP
429 with a safe retry indication.

## 11.6 HTTP security

-   HTTPS everywhere outside local development.
-   Helmet security headers.
-   Explicit CORS allowlist.
-   Request body size limits.
-   Zod validation.
-   Parameterized/ODM-safe database access.
-   No unsafe dynamic MongoDB operators from user input.
-   CSRF controls when cookie-based auth is used.
-   Secure cookies.
-   Dependency scanning and updates.
-   Secret scanning in CI.
-   Avoid leaking internal errors.
-   Protect admin routes.
-   Restrict file uploads if added in future.

------------------------------------------------------------------------

# 12. Error handling and observability

## 12.1 Error codes

  Code                                Meaning
  ----------------------------------- -----------------------------------
  `VALIDATION_ERROR`                  Invalid request
  `ACCOUNT_EXISTS`                    Registration conflict
  `AUTHENTICATION_FAILED`             Generic login failure
  `CONTACT_NOT_VERIFIED`              Contact is not verified
  `OTP_INVALID`                       Incorrect OTP
  `OTP_EXPIRED`                       OTP expired
  `OTP_ATTEMPTS_EXCEEDED`             Too many attempts
  `OTP_ALREADY_USED`                  Challenge consumed
  `RATE_LIMITED`                      Rate limit exceeded
  `UNAUTHORIZED`                      Missing/invalid session
  `FORBIDDEN`                         Insufficient permission
  `LOCKER_UNAVAILABLE`                Locker cannot be reserved
  `RESERVATION_EXPIRED`               10-minute reservation expired
  `BOOKING_INVALID_STATE`             Invalid transition
  `BOOKING_EXPIRED`                   Booking expired
  `PAYMENT_REQUIRED`                  Outstanding payment required
  `PAYMENT_FAILED`                    Payment failed
  `PAYMENT_ALREADY_PROCESSED`         Duplicate payment event
  `PAYMENT_RECONCILIATION_REQUIRED`   Payment outcome uncertain
  `DEVICE_OFFLINE`                    ESP32 unavailable
  `UNLOCK_FAILED`                     Hardware unlock failed
  `DOOR_NOT_CLOSED`                   Door remains open
  `LOCK_STATE_UNKNOWN`                Physical state unknown
  `RECOVERY_REQUIRED`                 Manual/controlled recovery needed
  `INTERNAL_ERROR`                    Unexpected server failure

## 12.2 Logging

Use structured Pino logs with: - timestamp - level - request ID -
trace/correlation ID - module - action - resource ID - safe error code -
duration

Redact: - OTP - JWT/access token - refresh token - encryption key - raw
email/phone - payment credentials - MQTT credentials - authorization
headers - secret query parameters

Do not log full request bodies indiscriminately.

## 12.3 Monitoring

Track: - API latency and error rate. - OTP delivery and verification
failures. - Rate-limit events. - Reservation conflicts and timeout
rate. - Payment pending/success/failure/reconciliation counts. - Expiry
worker lag. - Notification retry/failure count. - MQTT connection
state. - Device heartbeat age. - Unlock command latency and failure
rate. - Door-open duration. - Lock-state unknown events. -
Recovery-required bookings. - Database and Redis health.

Alert on occupied locker with unknown door/lock state, repeated device
failure, stale heartbeat, payment/booking inconsistency and worker
backlog.

------------------------------------------------------------------------

# 13. Background jobs and event processing

Use BullMQ with Redis for background processing. Jobs must be idempotent
and safe to retry.

## 13.1 Jobs

  Job                      Purpose
  ------------------------ ----------------------------------------------
  Reservation cleanup      Release expired unpaid reservations
  Booking expiry scan      Find bookings whose scheduled end has passed
  Expiry notification      Notify verified contact
  Notification retry       Retry transient SMS/email failures
  Payment reconciliation   Resolve pending/uncertain payment state
  IoT command timeout      Mark unacknowledged commands timed out
  Device health scan       Identify stale heartbeats
  Outbox publisher         Dispatch committed events
  Audit retention          Apply approved retention policy

## 13.2 Job design rules

-   Use unique job IDs/deduplication keys.
-   Expect at-least-once processing.
-   Make handlers idempotent.
-   Persist business state before acknowledging job completion.
-   Retry transient failures with bounded exponential backoff.
-   Send permanent failures to a dead-letter/review path.
-   Do not assume Redis job state is the only source of business truth.
-   Reconcile critical state from MongoDB.
-   Use server time.
-   Monitor worker lag and failure rates.

------------------------------------------------------------------------

# 14. State machines

## 14.1 Locker business status

``` text
available
  -> reserved
  -> occupied
  -> available

reserved
  -> available (reservation timeout/cancel)

available
  -> maintenance
  -> available

any unsafe/uncertain state
  -> recovery_required
  -> controlled recovery
  -> available or occupied
```

The locker's business status is separate from hardware door/lock state.

## 14.2 Booking state

Recommended statuses:

-   `reserved`
-   `pending_payment`
-   `active`
-   `expired`
-   `completed`
-   `cancelled`
-   `recovery_required`

Typical transitions:

``` text
reserved -> pending_payment
pending_payment -> active (confirmed payment and storage session start)
reserved/pending_payment -> cancelled (eligible cancellation/timeout)
active -> expired (scheduled end reached)
active -> completed (complete retrieval and physical confirmation)
expired -> completed (overdue paid, complete retrieval, physical confirmation)
active/expired -> recovery_required (hardware inconsistency)
recovery_required -> prior safe business state or completed/available through audited recovery
```

Partial retrieval does not create a new booking and does not complete
the current booking.

## 14.3 Payment state

Recommended statuses:

-   `created`
-   `pending`
-   `succeeded`
-   `failed`
-   `expired`
-   `reconciliation_required`
-   `refunded` or adjustment represented by a separate financial record
    where applicable

Payment state must not be inferred from frontend state.

## 14.4 Access session state

Recommended progression:

``` text
authorized
-> command_pending
-> unlock_confirmed
-> door_open
-> door_closed
-> locked
-> completed
```

Failure branches: - `failed` - `timed_out` - `recovery_required`

Only a validated physical closure/lock event can complete the retrieval
release path.

------------------------------------------------------------------------

# 15. Critical edge cases and required handling

  -----------------------------------------------------------------------
  Scenario                            Required behaviour
  ----------------------------------- -----------------------------------
  User closes browser during checkout Reservation expires after 10
                                      minutes and is released if payment
                                      is not confirmed

  Payment success arrives late        Reconcile; never activate a locker
                                      already reassigned

  Two users reserve same locker       Atomic reservation allows only one
  simultaneously                      winner

  OTP expired                         Reject and require a new challenge

  Multiple OTP resend requests        Rate limit; invalidate previous
                                      challenge

  Two concurrent OTP verification     Only one can consume challenge
  requests                            

  OTP guessed five times              Lock challenge and require a new
                                      controlled challenge

  Duplicate email/phone signup        Safe conflict response; no data
                                      leakage

  User logs in with unverified        Reject generically
  contact                             

  Device offline                      Do not report successful unlock

  Duplicate MQTT command              Device/backend deduplicate by
                                      command ID

  Command delivered but no execution  Timeout and reconcile device state
  acknowledgement                     

  Door remains open                   Keep locker unavailable; notify
                                      user/operations

  Booking expires while door is open  Do not force close or release
                                      locker

  User pays overdue charge but unlock Preserve payment/credit and create
  fails                               retry/recovery path; do not charge
                                      same period again

  User pays overdue amount but does   Keep paid-through credit; do not
  not open                            charge covered period again

  Partial retrieval after expiry      Keep locker occupied; next billing
                                      starts from paid-through timestamp

  User completes retrieval but door   Do not complete booking or release
  sensor says open                    locker

  Door closed but lock sensor unknown Do not release; recovery required

  Notification provider fails         Retry transient failures; record
                                      permanent failure

  Worker processes expiry twice       Idempotent status update and
                                      notification deduplication

  Admin disables occupied locker      Block ordinary action; require
                                      recovery workflow

  Database transaction fails          No partial state transition; retry
                                      safely

  Redis unavailable                   Fail/disable dependent queue
                                      operations safely; do not bypass
                                      security or physical authorization

  MQTT broker unavailable             Do not claim unlock; retain
                                      recoverable command/access state

  Power failure                       Physical safe-state and
                                      emergency-access procedure;
                                      reconcile on recovery

  Clock skew on ESP32                 Use server-issued expiry and
                                      bounded device clock policy; do not
                                      trust device clock alone for
                                      business pricing
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 16. Testing strategy and acceptance criteria

Testing must cover business logic, database concurrency, security and
physical-device failures.

## 16.1 Unit tests

-   Email normalization.
-   Phone normalization.
-   AES-256-GCM encrypt/decrypt and tamper detection.
-   HMAC blind index determinism.
-   OTP generation range and challenge digest.
-   OTP expiry and attempts.
-   OTP single-use consumption.
-   Rental price selection.
-   Started-hour overdue calculation.
-   Paid-through calculation.
-   Booking state transition validation.
-   Reservation expiry.
-   Idempotency key handling.
-   Notification deduplication.

## 16.2 Integration tests

-   Signup with email OTP.
-   Signup with mobile OTP.
-   Duplicate email.
-   Duplicate phone.
-   Login with verified email.
-   Login with verified phone.
-   Login using unverified contact.
-   Second contact verification.
-   Refresh token rotation.
-   Refresh token reuse detection.
-   Locker reservation.
-   Concurrent locker reservation.
-   Initial payment success/failure.
-   Late payment success.
-   Expiry job and notification.
-   Overdue payment.
-   Partial retrieval and paid-through update.
-   Complete retrieval and release.
-   MQTT simulated acknowledgement.
-   Device offline and timeout.
-   Door not closed.
-   Recovery-required transition.

## 16.3 Concurrency tests

Must specifically test:

-   Two users reserve the same locker at the same instant.
-   Two requests consume the same OTP.
-   Duplicate payment success events arrive.
-   Expiry worker and user access request run simultaneously.
-   Two overdue payment requests are initiated concurrently.
-   Two unlock requests are submitted for the same access session.
-   Device publishes duplicate acknowledgement/event.
-   Reservation cleanup runs while payment confirmation arrives.

## 16.4 End-to-end acceptance journeys

1.  New user signs up and verifies email.
2.  New user signs up and verifies phone.
3.  User logs in with verified email and fresh OTP.
4.  User logs in with verified phone and fresh OTP.
5.  User attempts login with unverified contact and is rejected.
6.  User selects a locker and pays within 10 minutes.
7.  User abandons checkout and reservation expires.
8.  User stores belongings and closes locker.
9.  User returns before expiry and retrieves all belongings.
10. User returns before expiry and partially retrieves belongings.
11. Booking expires and verified contact receives notification.
12. User returns after expiry, pays overdue amount and retrieves all
    belongings.
13. User returns after expiry, pays and partially retrieves.
14. User returns again and pays only for the new unpaid interval.
15. Device goes offline before unlock.
16. Door fails to close after retrieval.
17. Payment succeeds but physical unlock fails.
18. Admin monitors device and resolves recovery case.

## 16.5 Definition of done

V1 is not complete until: - All core acceptance journeys pass. -
Security tests pass. - Concurrency invariants pass. - Payment/booking
state consistency is demonstrated. - No occupied locker can be released
by an ordinary UI action. - Physical access is acknowledged by the
device adapter. - Failure/recovery paths are tested. - OpenAPI
documentation is accurate. - Logs redact sensitive information. -
Deployment secrets are not in source control. - Backup/restore and
operational runbooks exist.

------------------------------------------------------------------------

# 17. Development roadmap

## Phase 0 --- Product and engineering foundation

Deliverables: - Repository and workspace setup. - Coding conventions and
linting/formatting. - Environment configuration. - Docker development
environment. - MongoDB replica set. - Redis. - Pino logging and request
IDs. - Centralized error handling. - OpenAPI skeleton. - CI pipeline.

Exit criteria: - API starts and shuts down cleanly. - Database and Redis
health checks work. - Lint, tests and build run in CI. - No secrets are
committed.

## Phase 1 --- Authentication and data security

Deliverables: - User and pending registration models. - OTP verification
model. - Session model. - AES-256-GCM service. - HMAC blind-index
service. - Signup using email/mobile OTP. - Login using either verified
contact and fresh OTP. - Resend, expiry and attempt limits. -
Second-contact verification. - JWT access and refresh rotation. - Rate
limiting. - Auth middleware and role checks. - Authentication tests and
docs.

Exit criteria: - All authentication and encryption tests pass. - OTP
replay and concurrent consumption are blocked. - Duplicate contacts are
prevented. - No OTP/contact secrets appear in logs.

## Phase 2 --- Location and locker management

Deliverables: - Location model and admin CRUD. - Locker model and admin
CRUD. - Locker status and hardware state separation. - Availability
endpoint. - Atomic reservation foundation. - Maintenance/disabled
handling. - Locker admin UI.

Exit criteria: - Concurrent reservations cannot double-book. -
Unsafe/offline lockers cannot be booked.

## Phase 3 --- Pricing and booking

Deliverables: - Versioned pricing plans. - Exact duration allowlist. -
Quote endpoint. - Reservation creation. - 10-minute timeout. - Booking
lifecycle. - Reservation cleanup worker. - Booking UI.

Exit criteria: - Correct price for every allowed duration. - Invalid
duration rejected. - Abandoned reservation released safely. - Price
snapshot retained.

## Phase 4 --- Simulated payments

Deliverables: - Payment intent. - Mock UPI QR/URI. - Success/failure
simulation. - Idempotency. - Payment reconciliation foundation. -
Booking activation. - Payment history UI.

Exit criteria: - Client cannot forge payment success in production. -
Duplicate payment event does not duplicate activation. - Late payment
cannot activate a reassigned locker.

## Phase 5 --- Access and retrieval

Deliverables: - My lockers. - Access authorization. - Access sessions. -
Initial deposit access. - Retrieval access. - Overdue quote and
payment. - Partial retrieval. - Complete retrieval. - Paid-through
billing. - Device simulation adapter. - Access/retrieval UI.

Exit criteria: - Correct overdue amount. - Partial retrieval retains
locker. - Paid-through period is not double charged. - Complete
retrieval requires closure/lock confirmation.

## Phase 6 --- Expiry and notifications

Deliverables: - Expiry scan/worker. - Idempotent expiry transition. -
Verified-contact selection. - Simulated SMS/email adapter. - Retry and
notification history. - Admin notification monitoring.

Exit criteria: - Expiry event occurs once. - Notification only goes to
verified contact. - Failed delivery can be retried safely.

## Phase 7 --- Real ESP32 and MQTT integration

Deliverables: - MQTT broker setup. - Per-device identity/provisioning. -
TLS and ACL. - ESP32 firmware. - Lock driver control. - Door sensor. -
Heartbeat. - Unlock command. - Acknowledgements and events. - Command
timeout and recovery. - Hardware test procedures.

Exit criteria: - Real device acknowledges commands. - Door/lock state is
tracked. - Duplicate/replayed commands do not cause repeated unsafe
action. - Offline/failure states do not falsely complete access.

## Phase 8 --- Complete frontend and admin experience

Deliverables: - Responsive user UI. - Signup/login. - Locker
discovery. - Checkout and payment. - Active/expired bookings. -
Retrieval. - Profile and notifications. - Admin dashboard and
management. - Error/empty/loading states.

Exit criteria: - User journeys are usable on mobile and desktop. -
Frontend never controls authoritative payment/booking/access state.

## Phase 9 --- Hardening and deployment

Deliverables: - Full automated test suite. - Security review. -
Dependency and secret scanning. - Load/concurrency testing. - Docker
production images. - CI/CD. - Monitoring and alerting. - Backup and
restore. - Runbooks. - Physical safety and operational review.

Exit criteria: - Release checklist complete. - Critical recovery paths
demonstrated. - Monitoring and incident procedures available.

------------------------------------------------------------------------

# 18. Deployment and operations

## 18.1 Environments

Maintain separate: - local development - test/CI - staging - production

Each environment must have separate database, Redis, MQTT credentials,
signing keys, encryption keys and provider credentials.

## 18.2 Environment variables

Example names only; actual secret values must be supplied securely:

``` text
NODE_ENV=
PORT=

MONGODB_URI=
REDIS_URL=

JWT_ACCESS_SECRET=
JWT_REFRESH_SECRET=

CONTACT_ENCRYPTION_KEY_VERSION=
CONTACT_LOOKUP_HMAC_SECRET=
OTP_HMAC_SECRET=

MQTT_BROKER_URL=
MQTT_CA_CERT_PATH=
MQTT_CLIENT_ID=

EMAIL_PROVIDER_MODE=
EMAIL_PROVIDER_API_KEY=

SMS_PROVIDER_MODE=
SMS_PROVIDER_API_KEY=

PAYMENT_PROVIDER_MODE=

APP_ORIGIN=
CORS_ALLOWED_ORIGINS=
```

Do not put real secrets in `.env.example`. Validate all required
environment variables at startup. Prefer a secrets manager in
production. Plan encryption-key rotation and data re-encryption
procedures.

## 18.3 Docker services

Development `docker-compose` may include: - API - React web app -
MongoDB replica-set node(s) for transaction testing - Redis - MQTT
broker - Worker - Optional MongoDB admin UI only for local development

Do not expose database, Redis or MQTT administration ports publicly in
production.

## 18.4 CI/CD pipeline

1.  Install locked dependencies.
2.  Run formatting/lint checks.
3.  Run unit tests.
4.  Run integration tests.
5.  Run concurrency tests.
6.  Run dependency/security scans.
7.  Build frontend/backend/worker images.
8.  Deploy to staging.
9.  Run smoke tests.
10. Require approval for production.
11. Deploy with rollback strategy.
12. Verify health and monitor errors.

## 18.5 Backup and disaster recovery

-   Automated MongoDB backups.
-   Encryption at rest for backups.
-   Restore drills.
-   Document RPO/RTO once business requirements are known.
-   Back up configuration and firmware release artifacts securely.
-   Do not include raw secrets in backups without protected secret
    management.
-   Reconcile device state after backend/database recovery.
-   Keep physical emergency access procedures independent of application
    availability.

------------------------------------------------------------------------

# 19. Admin dashboard and operational reporting

## 19.1 Dashboard metrics

-   Total lockers.
-   Available lockers.
-   Reserved lockers.
-   Occupied lockers.
-   Maintenance lockers.
-   Offline devices.
-   Active bookings.
-   Expired occupied bookings.
-   Pending payments.
-   Payment reconciliation cases.
-   Failed unlock commands.
-   Doors left open.
-   Notification failures.
-   Revenue totals.

## 19.2 Locker management

Admin should be able to: - Create locker. - Assign location and size. -
Associate ESP32 device. - Change maintenance state. - View door and lock
state. - View current booking. - Review command history. - Open recovery
case.

Do not permit ordinary status edits that contradict physical state.

## 19.3 Reports

Potential V1 reports: - Booking count by day/location. - Revenue by
day/location. - Utilization by locker/size. - Average rental duration. -
Overdue charges. - Failed payment count. - Device uptime. - Unlock
success/failure. - Notification delivery.

Reports must use authoritative database records and clearly define
whether amounts are gross collected, pending, refunded or adjusted.

------------------------------------------------------------------------

# 20. Legal, privacy and operational considerations

Before public operation: - Publish privacy notice and terms. - Explain
what personal data is collected and why. - Define retention and deletion
policy. - Define payment/refund/overdue-charge terms clearly. - Obtain
appropriate consent where required. - Review applicable Indian data
protection and consumer/payment requirements with qualified counsel. -
Define liability and prohibited-item policy. - Define emergency access
and lost-access procedure. - Provide visible customer support contact. -
Assess physical safety, fire and electrical requirements. - Establish
incident response and breach handling. - Do not store or inspect item
contents unless a separately reviewed policy and lawful operational need
exists.

------------------------------------------------------------------------

# 21. Open implementation details that require explicit engineering validation

The following are technical details that were not all explicitly chosen
as product policies. The recommendations below are implementation
defaults and must be validated before production.

1.  **Rental timer start:** Store payment, unlock and door events
    separately. Recommended timer start is when the initial storage
    session is successfully established, with a documented
    fallback/timeout policy.
2.  **Initial deposit access:** Define what happens if payment succeeds
    but the user never opens the locker. Recommended: keep the
    booking/reservation in a controlled paid-but-not-started or recovery
    state; do not silently release an occupied/possibly opened locker.
3.  **Door left open:** Define a facility-specific alert/escalation
    period. Never automatically declare the locker available.
4.  **Payment provider reconciliation:** A real gateway's verified
    server callback/webhook is authoritative; the mock simulator is only
    for development.
5.  **Refund policy:** Business owner must define eligibility, fees and
    time limits. Do not invent refunds.
6.  **Emergency access:** Define who can authorize, how identity is
    checked, and how every action is audited.
7.  **Facility offline mode:** Define local access and power-failure
    operations before public launch.
8.  **Device clock:** Backend time is authoritative for pricing; device
    timestamps are event metadata and must be validated.
9.  **Notification contact selection after second verification:** Use
    the verified contact policy; do not send to an unverified
    destination.
10. **Multiple locations and locker sizes:** Initial schema supports
    these, but exact dimensions/capacity and user-facing size
    descriptions require facility data.

These items do not block the initial backend foundation, but must be
resolved before a live physical deployment where relevant.

------------------------------------------------------------------------

# 22. Final consolidated acceptance checklist

## Identity and security

-   [ ] Signup requires name, email and phone.
-   [ ] User chooses email or mobile OTP for signup.
-   [ ] Only selected contact is initially marked verified.
-   [ ] Both contacts are encrypted at rest.
-   [ ] HMAC blind indexes enforce lookup/uniqueness.
-   [ ] Login supports either verified contact.
-   [ ] Every login sends a fresh OTP.
-   [ ] OTP is context-bound, five-minute, six-digit and single-use.
-   [ ] Five-attempt limit is enforced.
-   [ ] Resend invalidates previous challenge.
-   [ ] Refresh token rotation/revocation works.
-   [ ] Rate limiting and authorization are tested.

## Locker and booking

-   [ ] User can view available lockers.
-   [ ] Locker selection is atomic.
-   [ ] Reservation timeout is exactly 10 minutes.
-   [ ] Abandoned reservation is released safely.
-   [ ] Only approved durations are accepted.
-   [ ] Backend calculates all prices.
-   [ ] Booking state transitions are enforced.
-   [ ] Expired booking remains occupied.

## Payments

-   [ ] Simulated UPI QR is clearly marked as simulation.
-   [ ] Payment is associated with user and booking.
-   [ ] Amount cannot be changed by client.
-   [ ] Payment idempotency works.
-   [ ] Duplicate and late payment events are handled.
-   [ ] Production cannot use development simulation endpoints.

## Retrieval and billing

-   [ ] User can access own locker only.
-   [ ] Overdue charge is ₹20 per started hour.
-   [ ] Overdue billing uses paid-through baseline.
-   [ ] Failed payment does not advance paid-through timestamp.
-   [ ] Partial retrieval keeps locker occupied.
-   [ ] Subsequent partial retrieval does not double-charge old periods.
-   [ ] Complete retrieval requires door closed and lock engaged.
-   [ ] Locker is released only after physical confirmation.

## Notifications

-   [ ] Expiry detection is automatic and idempotent.
-   [ ] Notification goes only to verified contact.
-   [ ] Notification status is recorded.
-   [ ] Retry does not duplicate successful delivery.

## IoT

-   [ ] ESP32 has unique identity.
-   [ ] MQTT uses TLS and device ACLs.
-   [ ] Commands have unique IDs and expiry.
-   [ ] Replay/duplicate command handling works.
-   [ ] Device acknowledgement and physical execution are separate.
-   [ ] Door sensor events are persisted.
-   [ ] Offline and unknown states block unsafe release.
-   [ ] Emergency access/recovery is documented.

## Quality and operations

-   [ ] Unit tests pass.
-   [ ] Integration tests pass.
-   [ ] Concurrency tests pass.
-   [ ] End-to-end journeys pass.
-   [ ] OpenAPI documentation is current.
-   [ ] Logs redact sensitive information.
-   [ ] CI/CD and deployment checks work.
-   [ ] Backups and restore are tested.
-   [ ] Monitoring and alerting are enabled.
-   [ ] Operational runbooks exist.

------------------------------------------------------------------------

# 23. Final project baseline

**Product:** Smart Digital Locker Management System\
**Version:** V1.0\
**Architecture:** Modular monolith\
**Backend:** Node.js + Express.js\
**Frontend:** React.js\
**Database:** MongoDB\
**Queue:** BullMQ + Redis\
**Authentication:** Context-bound OTP + JWT session management\
**Personal data protection:** AES-256-GCM + HMAC blind indexes\
**Payments:** Simulated UPI QR for development\
**Notifications:** Simulated SMS/email to verified contact only\
**Hardware:** ESP32 + electronic lock + door sensor\
**IoT communication:** MQTT over TLS\
**Reservation timeout:** 10 minutes\
**OTP expiry:** 5 minutes\
**OTP attempts:** 5\
**Rental options:** 4/6/8/12/24 hours\
**Rental prices:** ₹80/₹90/₹100/₹120/₹240\
**Overdue price:** ₹20 per started hour\
**Partial retrieval after expiry:** Paid-through billing\
**Architecture principle:** Backend is authoritative; physical release
requires device confirmation.

**Implementation order:** Foundation → Authentication/security → Locker
inventory → Pricing/booking → Payment simulation → Access/retrieval →
Expiry/notifications → ESP32/MQTT → Frontend/admin →
Hardening/deployment.

This document is the V1.0 implementation baseline. All core requirements
and finalized business decisions above must be preserved. Any future
change should be recorded as a versioned product/architecture decision
with its impact on database, APIs, state machines, UI, security and
tests.
