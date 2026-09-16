# Odoo Experience and partner registration

Target: the selected Odoo 19 control environment. Use its configured HTTPS origin
for registration links; deployment-specific addresses belong in dated reports.
Install `odoo_experience` and upgrade `oduflow_partner` for the registration
models, ACLs, stored instance ownership and administrator search/grouping.

## Booth operation

The Odoo Experience application contains Visitors and Events. Event 2026 is
seeded with stand A3. Assign `Odoo Experience Booth Staff` to booth operators;
assign `Odoo Experience Manager` for event configuration. Partners and customers
cannot read these internal visitor records or notes. Records follow allowed
Odoo companies.

In the event, an Oduflow administrator configures the default Oduflow Partner
as OduSphere (the active partner attached to the operating company's contact).
A disabled, provider-independent placeholder plan marks pending plan selection.
A missing or inactive operator blocks User registration rather than creating
an unowned instance. Changing the event URL affects future link display; keep
it on the trusted HTTPS control origin.

Create a visitor with first name, last name, email, company and Integrator/User.
Visitors start in New with Integrator selected by default. Event is hidden on
the visitor form. Use Ready to generate the personal link and move to
Ready; this approves Integrators immediately. Expires At is then set to the
issuance time plus the Event Token Days (30 days by default), displayed in the
current user timezone. It stays empty while New. Changing Token Days does not
change already-issued expiry dates. The link can be revoked before use.
Use Add Notes to record observations; mobile keyboard dictation can populate
this text field. Notes retain author and creation time and are append-only for
booth staff. The Android integration supports independent, idempotent note posts.

The invitation token is signed, scoped to the visitor, checked for expiry and
consumed under a database lock. It travels in a URL fragment and POST body,
not in access-log query strings. Prefill excludes private notes. Public input
cannot change role, operating company, owner or infrastructure configuration.
User registration creates a company with OduSphere ownership, a child portal
contact/user and a draft Instance. Integrator registration creates a company,
child partner portal user and Oduflow Partner; source keys sync through the queue.
No customer server is provisioned by either form.

After reviewing the booth form, the visitor is redirected to password setup.
Odoo validates the signed account token and authenticates the session. The
server selects `/my/oduflow` for User and `/my/partner` for Integrator; a submitted
redirect cannot change this. No invitation or welcome email is sent by this flow.
An administrator can issue another activation link using the normal account
recovery process if the visitor closes the window before saving a password.

## Ordinary registration

- `/oduflow/register/partner`: anonymous application with administrator review
  in Oduflow / Partner Applications. Approval creates the company, child user
  and source-access partner. Rejection grants no access. Copy the activation
  link from the approved application to deliver it manually.
- `/my/partner/register/client`: authenticated partners can register a client
  directly and copy their activation link, or create an email-bound invitation.
- `/oduflow/register/client`: invitation-based self-registration, valid for
  seven days. The issuing partner must still be active and belong to the same
  organization when the invitation is redeemed. No VM is created here.

No registration reuses an existing account or attaches an unknown person to an
existing company by matching email. Duplicate logins require support review.
Ordinary duplicate pending partner applications receive the same acknowledgement.
Company membership controls access for child contacts. Instance ownership is a
stored, readonly relation to the customer's commercial company; transfer the
customer company through authorized administration rather than writing the
instance owner. Global ORM rules continue to apply if another addon grants ACLs.

The Android app is a separate deliverable. Its implementation prompt and exact
HTTP payloads are in [odoo-experience-android-prompt.md](odoo-experience-android-prompt.md).

## Passing the booth screen to a visitor

The published Website page `/odoo-experience` appears in the website navigation
and can be edited through Website Pages. The self-registration screen is also
listed there as Odoo Experience Self-registration at
`/odoo-experience/self-register`. Both pages hide the site header and footer.
The landing page button opens the staff-operated screen. In the visitor list, Self-registration appears next to
New and opens the same screen directly. Only authenticated booth staff can
start a session; anonymous visitors cannot use it to approve Integrators.

Before use, an Experience Manager must set Event → Booth Password (at least
five characters of any kind). The password is hashed, never redisplayed, and can be
replaced in the same field. No default or demo password is installed.

The visitor completes the styled web form. Confirm saves one Visitor in New;
Cancel saves no visitor. Both show the booth-password screen, without the
visitor's details. Neither creates a portal user, partner agreement or Instance.
Staff can review the new record and issue its invitation afterward.

Until the correct event password is entered, the same browser cannot make
backend/RPC requests or navigate to other authenticated Odoo pages. A server
record ties the lock to the existing browser session. Five failed password
attempts impose a one-minute delay. The password unlocks this previously
started session; it is not a login credential for another browser. Successful
entry returns to the Odoo Experience visitor list. Form nonces and database
locking prevent duplicate submissions and reuse of a previous visitor's form.

## Visitor action buttons

Self Register is orange in the backend; the Website self-registration buttons
use the same color. The visitor list and form include Invite, Picture! and
Delete. Ready prepares the personal registration link for a New visitor without sending
email. Invite then opens a delivery confirmation dialog: select Email Sent
and/or Photo Printed to record a completed handoff. At least one must be
selected. Invited stores the delivery channels, confirmation time and staff
member. The link and its expiry remain unchanged and work in both Ready and
Invited. This manual confirmation does not send email or operate a printer. Picture! is reserved for the future camera application;
it currently displays a notice that the application is not connected. No photo
is captured, uploaded or sent by this placeholder.

Delete asks for confirmation and removes the visitor and their notes. Existing
customer accounts, partner agreements and instances remain. Booth staff may only
delete visitors in their allowed companies. Deleting a visitor does not unlock
an associated screen session; the booth password remains required.

The status order is New → Ready → Invited → Registered, with Revoked for
cancelled links. Upgrade 19.0.1.3.0 moves the former Invited records to Ready
because those records only proved link preparation. Tokens and expiry dates
are preserved; Registered and Revoked records remain in their existing states.
Android scan creation now returns Ready with the prepared registration URL.


### Registration validation and recovery

The visitor registration form uses English labels and actionable errors for
required fields, field lengths, invalid email addresses, existing accounts, and
invalid, expired, revoked or already used invitations. Existing accounts are
checked by case-insensitive login and normalized email, including archived users;
registration never attaches a visitor to an existing account. A duplicate email
shows: "This email is already registered. Please sign in or use another email address."

After a failed submission, editable fields stay in the form and the invitation
remains usable. Account creation and visitor completion roll back together.
Password activation checks length and confirmation and preserves the activation
token if password setup fails. Booth unlock passwords and account passwords retain
their separate policies. Unexpected failures show a support reference; logs record
that reference and exception class without exception messages, SQL values or tokens.
Transient database failures still propagate to Odoo's retry handling.


Ready and Invited visitors display a locally generated QR code below their
registration link. The code contains the full registration URL, including the
fragment. New links use `/odoo-experience/register#<invitation>`; previously
issued `#token=<invitation>` links remain supported. QR codes disappear with
the invitation link when a visitor is registered or revoked. No external QR
service receives invitation credentials.
