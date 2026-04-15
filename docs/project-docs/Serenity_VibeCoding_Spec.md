# Serenity — Production App Specification
### For vibe-coding platforms (Lovable / Bolt / v0 / Cursor)

---

## PART 1 — PROJECT CONTEXT

**App name:** Serenity  
**Type:** Mobile-first matrimony / matchmaking app (web-responsive also acceptable)  
**Tone:** Calm, trust-first, non-judgmental  
**Core promise:** No fees for discovery, matching, or connecting with matches. Conversations happen on the users’ own messaging apps (e.g. WhatsApp, Telegram, Instagram)—not in a built-in chat box. Revenue only from optional services.

---

## PART 2 — RECOMMENDED TECH STACK

### Frontend

- **Next.js** — React-based framework
- **React** — Core UI library inside Next.js
- **TypeScript** — Recommended for large-scale apps
- **Tailwind CSS** — Styling
- **Axios** — API requests
- **React Query** — Server state management and caching
- **Zustand** — Client/UI state management
- **React Hook Form** — Form handling
- **Zod** — Form validation

### Backend

- **Django** — Core backend framework
- **Django REST Framework** — API layer
- **Gunicorn** — Production WSGI server
- **Nginx** — Reverse proxy and traffic handling

### Database

- **PostgreSQL** — Production database
- **SQLite** — Development database
- **Django ORM** — Database interaction layer

### Cache and queue

- **Redis** — Caching and message broker
- **Celery** — Background task processing using Redis

### Authentication and security

- **JWT authentication** — `djangorestframework-simplejwt`
- **Django authentication system**
- **CORS** — `django-cors-headers`
- **Password hashing** — Django built-in security

### Storage

- **AWS S3** or S3-compatible storage — File uploads and media storage
- **Presigned URLs** — Secure uploads

### Notifications

- **Firebase Cloud Messaging** — Push notifications
- **Email service** — SendGrid / AWS SES / Resend
- **Celery workers** — Async notification delivery

### Search system

- **Elasticsearch** or **OpenSearch** — Fast filtering and profile search at scale

### Payments

- **Razorpay** — Payment gateway
- **Webhooks** — Handled via Django backend

### Infrastructure and deployment

- **Docker** — Containerization
- **Docker Compose** — Local development
- **Nginx** — Load balancing and routing
- **Load balancer** — AWS ALB or equivalent cloud service
- **Horizontal scaling** — Multiple backend instances
- **CDN** — Cloudflare or Vercel Edge for frontend assets

### Monitoring and logging

- **Sentry** — Error tracking
- **Prometheus** — Metrics collection
- **Grafana** — System monitoring dashboards

### Communication protocols

- **HTTPS** — Secure communication
- **REST APIs** — DRF-based
- **JSON** — Data format
- **Axios** — Frontend API communication

### 2.1 External messaging (MVP)

Serenity uses **no in-app message thread** for matched users. User-configured contact handoff uses deep links to WhatsApp (`wa.me`), Telegram (`t.me`), Instagram profile / IG handle (details below). Optional later: WhatsApp Business Cloud API only if the product needs **server-initiated** templates—not required for MVP handoff.

---

Serenity **does not** relay chat through its servers for matched users. It **orchestrates consent and visibility**: after rules are satisfied, each user sees the other’s **enabled** contact identifiers and taps to open the third-party app.

| Platform | Typical handoff | MVP implementation |
|----------|-----------------|-------------------|
| **WhatsApp** | `https://wa.me/<e164_without_plus>?text=<prefill>` | Prefer `User.mobile` (E.164) as WhatsApp destination when user opts in; optional override number stored only if different. |
| **Telegram** | `https://t.me/<username>` | Store `@username` or bare username; validate format server-side. |
| **Instagram** | Profile / DM entry | Store handle (no `@` prefix in DB, display with `@`). **Note:** Instagram does not offer a generic “DM any user” API like WhatsApp/Telegram; MVP is **open profile / user copies handle** and messaging happens in the Instagram app under Meta’s policies. |

**Product copy:** Explain that Serenity cannot read or moderate messages sent outside the app; safety tools (report/block) remain in Serenity.

### 2.2 Secrets and third-party credentials (~100 users scale)

Store **application** secrets (SMS API key, email API key, Razorpay key secret + webhook secret, JWT signing secret, S3/R2 credentials or IAM role) in **environment variables** on the server or a **secrets manager** (e.g. Doppler, Infisical, AWS Secrets Manager)—**never** in the client or in git.

- **Razorpay:** Key ID may appear in client where required; **key secret** and **webhook secret** are server-only; verify webhook signatures.
- **Do not** store vendor master keys in per-user database rows. If you later store OAuth tokens (e.g. Meta Business), encrypt at rest (envelope encryption + KMS or equivalent) and rotate on schedule.

---


## PART 3 — DATA MODELS

### 3.1 User

User {
  id: UUID (PK)
  mobile: string (unique, OTP-verified)
  name: string
  dob: date
  age: computed (derived from dob, never store raw DOB in public view)
  gender: enum(male, female, other)
  current_city: string
  gov_id_type: enum(aadhaar, passport, voter_id)
  gov_id_number: string (encrypted at rest)
  profile_photo_url: string
  identity_verified: boolean (default: false)
  search_mode: enum(active, inactive) (default: active)
  last_action_at: timestamp (request sent OR request accepted)
  requests_sent_today: int (reset daily at midnight)
  requests_sent_reset_at: date
  created_at: timestamp
  updated_at: timestamp
}
```

### 3.2 ProfileOptional

```
ProfileOptional {
  id: UUID (PK)
  user_id: UUID (FK -> User)

  // Personal #change
  religion: string | null
  caste: string | null
  mother_tongue: string | null
  education: string | null
  occupation: string | null
  family_details: text | null         // click-to-reveal

  // Physical
  height_cm: int | null
  weight_kg: int | null
  body_type: string | null

  // Financial
  salary_range: string | null         // click-to-reveal
  investments: string | null          // click-to-reveal
  properties: string | null           // click-to-reveal

  // Health
  physical_health_summary: text | null   // click-to-reveal
  mental_health_summary: text | null     // click-to-reveal

  // Lifestyle
  smoking: enum(never, occasionally, regularly) | null
  drinking: enum(never, occasionally, regularly) | null
  diet: enum(vegetarian, non_vegetarian, vegan, jain) | null
  fitness_habits: string | null
  relocation_preference: enum(open, city_bound, country_bound) | null

  updated_at: timestamp
}
```

### 3.3 FieldVerification

One row per verifiable field per user.

```
FieldVerification {
  id: UUID (PK)
  user_id: UUID (FK -> User)
  field_name: string              // e.g. "height_cm", "salary_range", "identity"
  status: enum(not_verified, pending, verified, rejected)
  verified_at: timestamp | null
  verified_by: string | null      // internal agent / partner ID
  service_booking_id: UUID | null (FK -> ServiceBooking)
  updated_at: timestamp
}
```

### 3.4 SearchFilter (locked daily)

```
SearchFilter {
  id: UUID (PK)
  user_id: UUID (FK -> User, unique per day)
  locked_at: timestamp
  locked_for_date: date

  // Each filter is a key-value. Only fields user has filled are stored.
  // verified_only = true means user must also be verified on that field.

  filters: JSONB  
  // example:
  // {
  //   "height_cm": { "min": 160, "max": 175, "verified_only": true },
  //   "religion":  { "value": "Hindu", "verified_only": false },
  //   "salary_range": { "value": "10-20L", "verified_only": false },
  //   "age": { "min": 25, "max": 32 },
  //   "preferred_city": "Hyderabad",
  //   "career_priority": "growth",
  //   "family_style": "nuclear",
  //   "relocation": "open",
  //   "children_timeline": "2-3 years"
  // }

  updated_at: timestamp
}
```

### 3.5 Request

```
Request {
  id: UUID (PK)
  from_user_id: UUID (FK -> User)
  to_user_id: UUID (FK -> User)
  status: enum(pending, accepted, declined, withdrawn)
  questions_answered_by_sender: boolean (default: false)
  questions_answered_by_receiver: boolean (default: false)
  created_at: timestamp
  updated_at: timestamp
}
```

### 3.6 Match

```
Match {
  id: UUID (PK)
  user_a_id: UUID (FK -> User)
  user_b_id: UUID (FK -> User)
  status: enum(active, soft_paused_by_a, soft_paused_by_b, unmatched)
  chat_enabled: boolean   // Semantics: "contact handoff enabled" — when true AND match active, each user may see the other's enabled contact links (WhatsApp / Telegram / Instagram). Not an in-app chat flag.
  matched_at: timestamp
  updated_at: timestamp
}
```

### 3.7 QuestionAnswer

```
QuestionAnswer {
  id: UUID (PK)
  request_id: UUID (FK -> Request)
  answered_by_user_id: UUID (FK -> User)
  question_key: string   // e.g. "living_preference", "children_timeline"
  answer_text: string
  created_at: timestamp
}
```

Preset question template set (examples, backend-validated keys):

| key | question | format |
|-----|----------|--------|
| living_preference | "Do you prefer living with family or independently after marriage?" | multiple_choice |
| career_next_5 | "What is your career priority in the next 5 years?" | short_text |
| preferred_city | "Which city do you prefer to live in after marriage?" | short_text |
| children_timeline | "What is your expected timeline for children?" | multiple_choice |
| lifestyle_expectation | "What is one daily lifestyle expectation you have from your partner?" | short_text |

### 3.8 UserContactPreference

Per-user settings for **which** external channels they expose to matches (and optional overrides). Validated server-side. Shown to the other party **only** when `Match.chat_enabled` is true and `Match.status = active` (see R9).

```
UserContactPreference {
  id: UUID (PK)
  user_id: UUID (FK -> User, unique)

  // WhatsApp: if whatsapp_enabled, use whatsapp_phone_e164 OR fall back to User.mobile (must be E.164) when use_registered_mobile_for_whatsapp = true
  whatsapp_enabled: boolean (default: false)
  use_registered_mobile_for_whatsapp: boolean (default: true)
  whatsapp_phone_e164: string | null

  telegram_enabled: boolean (default: false)
  telegram_username: string | null   // without @; store normalized

  instagram_enabled: boolean (default: false)
  instagram_handle: string | null    // without leading @; store normalized

  updated_at: timestamp
}
```

At least one channel should be enabled before a match can show a useful handoff; UI may prompt on first match if all disabled.

### 3.9 SafetyEvent

```
SafetyEvent {
  id: UUID (PK)
  reported_by: UUID (FK -> User)
  reported_user: UUID (FK -> User)
  type: enum(report, block)
  reason: string | null
  status: enum(open, reviewed, resolved)
  created_at: timestamp
}
```

### 3.10 ServiceBooking

```
ServiceBooking {
  id: UUID (PK)
  user_id: UUID (FK -> User)
  service_type: enum(identity_verification, financial_verification, health_verification, physical_verification, photography, wedding_planner)
  payment_status: enum(pending, paid, refunded)
  operational_status: enum(booked, in_progress, completed, cancelled)
  amount_inr: decimal
  booked_at: timestamp
  completed_at: timestamp | null
}
```

---

## PART 4 — BUSINESS LOGIC RULES (implement exactly)

### R1 — Daily request cap

```
BEFORE sending request:
  IF requests_sent_today >= 3:
    REJECT with "You have used all 3 requests for today."
  ELSE:
    CREATE Request
    INCREMENT requests_sent_today
    SET last_action_at = now()
```

Reset rule: `requests_sent_today = 0` every day at midnight (cron job).

### R2 — Active match cap

```
BEFORE accepting a request:
  active_count = COUNT(Match WHERE (user_a OR user_b = current_user) AND status = 'active')
  IF active_count >= 3:
    REJECT with "You already have 3 active matches. Soft pause one to accept new."
  ELSE:
    CREATE Match (status = active, chat_enabled = true)
    SET Request.status = accepted
    SET last_action_at = now()
```

### R3 — Optional questions (max 5)

```
Question templates are examples; user is free to answer none, some, or up to 5.
Question keys must belong to the backend-supported preset key list.

BEFORE persisting request answers (send or accept flow):
  VALIDATE all provided question keys are allowed
  VALIDATE each provided answer is non-empty
  ENFORCE max 5 answers per user per request
  ALLOW send/accept even if no answers are provided
```

### R4 — Filter fairness

```
WHEN building filter options for user U:
  FOR each filter_field F:
    IF ProfileOptional[U][F] is null:
      SHOW filter as disabled with hint: "Add this to your profile to use this filter"
    ELSE IF user wants verified_only = true for F:
      CHECK FieldVerification[U][F].status = 'verified'
      IF NOT verified: SHOW verified_only toggle as disabled with hint: "Verify this field to use verified-only filter"
      IF verified: ALLOW
    ELSE:
      ALLOW filter use
```

### R5 — Search mode auto-pause

```
CRON: runs daily at midnight
  FOR each User U WHERE search_mode = 'active':
    days_since_action = today - U.last_action_at (in days)
    IF days_since_action >= 7:
      SET U.search_mode = 'inactive'
      SEND in-app notification: "Your profile has been paused. Tap to reactivate."
      SEND email (once only, check flag): "Your Serenity profile is now paused..."
      SET U.auto_pause_email_sent = true
```

Login reminder rule:

```
ON each user login:
  IF U.search_mode = 'inactive':
    SHOW banner: "Your profile is paused. Reactivate to appear in search."
    (every login until reactivated)
```

### R6 — Soft pause transparency

```
WHEN User A soft-pauses Match M:
  SET Match.status = 'soft_paused_by_a'
  SET Match.chat_enabled = false
  NOTIFY User B: "Your match has paused this connection. You can wait or unmatch."
  SHOW label on Match card for both users: "This match is paused"
  (User B must not see A's contact handoff links while paused — see R9.)
```

### R7 — Sensitive field visibility

The following fields are click-to-reveal on profile view (not auto-visible):
- caste
- salary_range
- investments
- properties
- physical_health_summary
- mental_health_summary
- family_details
- exact DOB (only age is shown by default)

All other filled fields are visible by default on profile.

### R8 — Filter lock

```
ONCE PER DAY:
  User builds their search filters on Filter Builder screen.
  On tap "Lock filters for today":
    SAVE SearchFilter with locked_for_date = today
    ALLOW discovery to run using these filters.
  IF already locked today:
    SHOW saved filters as read-only.
    SHOW: "Filters locked until tomorrow."
```

### R9 — External messaging (contact handoff) access

There is **no** in-app message thread. Serenity only controls **whether each user may see the other’s** `UserContactPreference` handoff links.

```
CONTACT HANDOFF is shown IF AND ONLY IF:
  Match.status = 'active' AND Match.chat_enabled = true

IF match is soft_paused OR unmatched:
  HIDE the other user's WhatsApp / Telegram / Instagram handoff buttons
  SHOW banner: "Contact sharing is paused for this match." (soft pause) or hide match actions (unmatched)

FOR EACH channel on UserContactPreference:
  SHOW button only if channel is enabled AND a valid identifier exists (or WhatsApp fallback to User.mobile when use_registered_mobile_for_whatsapp = true)

PREFILL (optional): Opening WhatsApp may append a short URL-encoded intro, e.g. "Hi — we matched on Serenity."
```

**Unmatch:** No message history in Serenity (none stored). Optional: show "Matched on [date]" only for UX continuity.

### R10 — Incoming interest spam protection

```
IF User A sends request to User B:
  CHECK if A has been blocked by B: REJECT silently
  CHECK if A has an open request to B already: REJECT "Request already sent"
  CHECK if A was previously declined by B within 30 days: REJECT "Cannot re-request within 30 days"
  CHECK if B has reported A: FLAG for review, HOLD request
```

---

## PART 5 — API ENDPOINTS

### Auth

| Method | Route | Description |
|--------|-------|-------------|
| POST | /auth/request-otp | Send OTP to mobile |
| POST | /auth/verify-otp | Verify OTP, return JWT |
| POST | /auth/logout | Invalidate session |

### Profile

| Method | Route | Description |
|--------|-------|-------------|
| GET | /profile/me | Get own full profile |
| PUT | /profile/identity | Update mandatory identity fields |
| PUT | /profile/optional | Update optional fields |
| GET | /profile/:userId | Get public profile (respects click-to-reveal) |
| GET | /profile/:userId/trust-card | Get trust card summary |

### Filters

| Method | Route | Description |
|--------|-------|-------------|
| GET | /filters/available | Get list of filters user is eligible to use |
| GET | /filters/current | Get today's locked filters |
| POST | /filters/lock | Save and lock filters for today |

### Discovery

| Method | Route | Description |
|--------|-------|-------------|
| GET | /discovery/feed | Get paginated discovery feed (applies locked filters) |
| POST | /discovery/skip/:userId | Skip a profile (no record) |

### Requests

| Method | Route | Description |
|--------|-------|-------------|
| POST | /requests/send/:toUserId | Send request (enforces R1, R3 validations) |
| POST | /requests/:requestId/accept | Accept request (enforces R2, R3 validations) |
| POST | /requests/:requestId/decline | Decline request |
| GET | /requests/incoming | List incoming pending requests |
| GET | /requests/sent | List sent requests with status |

### Questions

| Method | Route | Description |
|--------|-------|-------------|
| GET | /questions/list | Get preset question templates (examples) |
| POST | /questions/answer/:requestId | Add or update a single answer (key validated, max 5/user/request) |
| GET | /questions/answers/:requestId | Get all submitted answers for the request |

### Matches

| Method | Route | Description |
|--------|-------|-------------|
| GET | /matches | List active + soft-paused matches |
| POST | /matches/:matchId/soft-pause | Soft pause a match |
| POST | /matches/:matchId/reactivate | Reactivate a paused match |
| POST | /matches/:matchId/unmatch | Unmatch (hard) |

### Contact preferences (external messaging)

| Method | Route | Description |
|--------|-------|-------------|
| GET | /contact/me | Get own `UserContactPreference` |
| PUT | /contact/me | Update enabled channels and identifiers (validated) |
| GET | /matches/:matchId/contact-handoff | Returns **other** user’s handoff payload only if R9 allows (enabled channels + constructed deep-link URLs for client to open). Never return identifiers if match paused or not active. |

### Search Mode

| Method | Route | Description |
|--------|-------|-------------|
| PUT | /settings/search-mode | Set active or inactive manually |

### Safety

| Method | Route | Description |
|--------|-------|-------------|
| POST | /safety/report/:userId | Report a user (callable from Connect screen or profile) |
| POST | /safety/block/:userId | Block a user |
| GET | /safety/blocked | List blocked users |
| GET | /safety/reports | List own submitted reports |

### Services Marketplace

| Method | Route | Description |
|--------|-------|-------------|
| GET | /services | List available services with prices |
| POST | /services/book | Book a service |
| GET | /services/my-bookings | List own bookings |
| POST | /services/:bookingId/pay | Initiate Razorpay payment |
| POST | /services/payment-webhook | Razorpay payment callback |

### Verification

| Method | Route | Description |
|--------|-------|-------------|
| GET | /verification/status | Get verification state for all fields |
| POST | /verification/request/:fieldName | Link verification request to a service booking |

---

## PART 6 — SCREEN SPECIFICATIONS

### Registration UI Theme (applies to Screens 1–5)

- Visual style: modern, minimalist, clean, airy, and mobile-first.
- Colors:
  - Primary: `#708090` (Soft Slate Blue)
  - Secondary: `#F7CAC9` (Rose Quartz)
  - Accent / surface: `#FFFFFF` (Pure White)
- Components: rounded corners (large radius), soft shadows, high whitespace usage.
- Typography: clean modern sans-serif; medium weight headings; readable body text.
- Motion: subtle transitions between steps; no heavy animations.
- Consistency: maintain aligned spacing system and same input/button language across all steps.
- Step progress indicator: persistent top progress component on every onboarding screen (`Step X of 5`).

---

### Screen 1 — Welcome / Sign In (Step 1 of 5)

- Centered app logo: **Serenity**
- Minimal tagline: "Find your perfect match"
- Sign-in options (stacked rounded buttons, primary color treatment):
  - Continue with Mobile
  - Continue with Email
  - Continue with Google
  - Continue with Apple
- Overall look: premium, uncluttered, high-contrast on white background.

---

### Screen 2 — Profile For Whom (Step 2 of 5)

Title: **"Who are you creating this profile for?"**

Selectable card options (with icon + label):
- Yourself
- Son
- Daughter
- Brother
- Sister
- Relative
- Friend

Interaction rules:
- Cards use soft hover/pressed feedback and clear selected state.
- Selected state uses secondary color `#F7CAC9`.
- Only one option selected at a time.

---

### Screen 3 — Basic Details (Step 3 of 5)

Fields:
- First Name
- Last Name
- Date of Birth (date picker)

Gender logic:
- If **Yourself** selected in Screen 2: show selectable gender control (Male / Female).
- If **Son** or **Brother** selected: auto-set gender to Male (read-only).
- If **Daughter** or **Sister** selected: auto-set gender to Female (read-only).
- If **Relative** or **Friend** selected: show selectable gender control (Male / Female).

UI behavior:
- Keep a clean form layout with generous vertical spacing.
- Show helper text where gender is auto-filled to explain why it is locked.

---

### Screen 4 — Contact & Location (Step 4 of 5)

Fields:
- Mobile Number (with OTP verification UI)
- Email (with verification indicator state)
- Country (dropdown)
- State (dropdown)
- City (dropdown)

Interaction rules:
- Inputs use modern style with leading icons where suitable.
- OTP area includes send, resend timer, verify, and verified state.
- Email row supports pending/verified indicator.

---

### Screen 5 — Personal Details (Step 5 of 5)

Fields:
- Height (dropdown)
- Marital Status (dropdown)
- Profession / Work
- Salary
- Qualification

Layout:
- Use grouped card-style input sections.
- Maintain minimalist spacing and rounded visual language.

Primary CTA:
- **Complete Profile**

Final action:
- After successful completion, redirect user to main app dashboard (home screen with matches).

---

### Registration Transition Rule

- Onboarding screens 1–5 are mandatory for first-time users.
- If user exits mid-way, resume from last completed step.
- Returning users with completed onboarding skip directly to dashboard/matches home.

---

### Post-registration / Existing User Home Interface (Dashboard)

This dashboard is the landing screen after registration and for all returning users. It should visually align with a modern, minimalist mobile interface.

Required layout:
- Top-left: Serenity logo + greeting context (example: "Hello, Vikram K" and profile-for context like "You're viewing matches for your brother, Sanjay.").
- Top-right: **Connections** icon/button with unread badge support.
- **Do not show a search bar** on this screen.
- **Do not show inline filter icon/button** in the header area.
- Optional trust-promotion card near top (example: verification badge promo).
- Main body: Discovery feed cards with photo, name, age, trust/verified snippet, and quick actions.

Bottom navigation (fixed):
- Home
- Discover
- Filters
- Profile
- Settings

Navigation rules:
- Remove **Connections** from bottom navigation.
- Connections must open from the top-right shortcut only.
- Filters must open from the bottom `Filters` tab (not from the Home header).

---

### Screen 6 — Filter Builder

**Header:** "Your search filters"
**Sub-header:** "You can only use filters for fields you've filled in your profile."

Primary control (mandatory):
- `Verified profiles only` [toggle]
- If ON: discovery results must include only profiles with verified identity/trust status (`User.identity_verified = true` at minimum).
- If OFF: normal filter behavior applies.

Filter lock status banner:
- If not locked: "Filters not locked yet. Lock to activate search."
- If locked: "Filters locked until tomorrow. [View only]"

**Show every filter row (no hiding incomplete fields):**
- Render **all** filterable dimensions the product supports (Personal, Physical, Financial, Health, Lifestyle, plus partner-preference rows below), even when the user has not filled the corresponding profile field.
- **Enabled state:** user has filled that field in their own profile (and passes verified-only rules where applicable). Controls are interactive.
- **Disabled state:** row stays visible; show short reason: `Add [field name] in your profile to use this filter` — **not** a generic "fill in your bio" unless the field truly lives only in bio. Use field-specific copy (e.g. Caste → "Add caste in Personal to filter on caste").
- **Optional:** trailing `Off` or grey pill on disabled rows so it is obvious the filter exists but is inactive.

**Section groups — interaction requirements:**

Each section (Personal / Physical / Financial / Health / Lifestyle) renders field filters. **Age** and **height** (and any numeric range) must use **real controls**, not static text:

```
Age (partner)
[Dual-handle range slider OR two number inputs: Min / Max] — bound to allowed app range (e.g. 18–60); values persist in filter JSON until lock.
Verified only? [toggle] if product requires age verification parity (else omit).

Height (cm)
[Dual-handle range slider OR Min / Max inputs] — must be draggable/editable in prototype.
Verified only? [toggle] — disabled if user's own height not filled + not verified per rules.
────────────────────────────────
Religion [text / chips]
Caste [text / chips] — same enable/disable rules as other bio-backed fields
Education, Occupation, Body type, Diet, Smoking, Drinking, Relocation, Salary range, Investments, Properties, Health summaries — each as row with appropriate control (text, select, range) when enabled
```

**Partner preference filters (search-only, no bio required):**
- Preferred city [text input]
- Preferred age range — **must duplicate or merge** with Personal age logic: do not show a dead, non-editable age control; single source of truth for `age.min` / `age.max` in `SearchFilter.filters`.
- Family style [dropdown: nuclear / joint / flexible]
- Career priority [dropdown]
- Relocation open [yes/no toggle]

**Lock behavior:**
- When locked for the day, sliders/inputs become read-only but remain visible with current values.

**Layout suggestions:**
- Collapse sections with accordions; sticky section headers while scrolling.
- For many rows, use a compact list with chevron → sub-screen per category (Personal filters, Physical filters, …) to reduce clutter.

Bottom CTA: "Lock filters for today"

---

### Screen 7 — Incoming Interests

Header: "People interested in you"
Count badge on tab.

Each list item:
- Profile photo thumbnail
- Name, Age, City
- Trust Card mini
- Shared filter overlap chips
- Action buttons: `[ Decline ]` `[ View Profile ]` `[ Answer & Accept ]`

**Answer & Accept behavior:**
- Answering questions is optional; accept is not blocked by unanswered templates
- If user chooses to answer, allow up to 5 and enforce valid keys

---

### Screen 8 — Sent Requests

Tab list showing sent requests with status:
- `Pending` — waiting for response
- `Accepted` — opens match
- `Declined`
- `Withdrawn` (allow withdraw on pending)

---

### Screen 9 — Questions Exchange (Optional)

**Entry points:** from Send Request flow OR from Accept flow.

Header: "Compatibility Questions"
Sub: "Answer up to 5 compatibility questions (optional)."

Card stack (one question at a time):
- Question text
- Answer control (multiple choice OR short text input)
- Progress: "Question 2 of up to 5"

Navigation: Next / Back

Final card: "Submit Answers"

**After one or both users submit:**
- Show both answers side by side
- Label: "Their answer" / "Your answer"
- CTA: `[ Proceed to Request / Accept ]`

---

### Screen 10 — Matches

**Tab bar:** Active (n) | Paused (n)

**Active tab (max 3):**

Match row:
- Profile photo, Name, Age, City
- "Matched on [date]"
- `[ Connect ]` `[ Soft Pause ]`

**Soft Pause confirmation:**
- Modal: "Pausing will hide contact sharing on both sides. They will see this match is paused. Continue?"
- Confirm → sets status = soft_paused

**Paused tab:**

Match row:
- Profile photo, Name, Age, City
- Yellow label: "This match is paused by you"
- OR: "This match is paused by them"
- `[ Reactivate ]` (if active slots < 3) `[ Unmatch ]`

---

### Screen 11 — Connect (external messaging)

**Purpose:** After a match is active, users continue talking on **WhatsApp, Telegram, or Instagram**—not inside Serenity. This screen shows **handoff buttons** and safety actions.

**Top bar:**
- Profile name + Trust Card mini
- `[ Report ]` `[ Block ]` icons

**If `chat_enabled` = false (soft pause) or match not active:**
- Hide other party’s contact links
- Banner: "Contact sharing is paused for this match."

**When handoff is allowed (R9):**
- Short explainer: "You’ll chat in your own app. Serenity can’t read messages sent outside the app."
- **Your outgoing channels:** summary of what *you* have enabled (link to edit in Settings / Contact — same fields as `UserContactPreference`).
- **Their channels:** one primary button per enabled channel:
  - **WhatsApp** — opens `wa.me` with optional prefill
  - **Telegram** — opens `t.me/<username>`
  - **Instagram** — opens profile or app path for `@handle` (per platform capability; may be "Copy @handle" if OS cannot deep link to DM)

**No** message list, text field, or image upload on this screen.

---

### Screen 12 — Profile (Self View)

Header: "Your profile"

**Edit affordances (required):**
- Primary CTA: **Edit profile** (opens Screen 4 — Optional Profile Builder — or a unified editor with the same field set).
- **Per-row edit:** Each labeled row is tappable (chevron or pencil) and navigates to the correct tab/field in the editor, or opens inline edit for that field.
- Do not ship a read-only self profile with no path to change data.

**Sections:**
- Profile photo(s) — tap camera / photo to change (same as onboarding rules).
- Identity (mandatory fields shown, gov ID shown as type only for privacy)
- **Trust Overview** (existing mini summary) plus expanded trust presentation below.

**Always-visible optional rows (even when empty):**
- Under **Personal:** include **Caste** in the same table as Religion, Education, Occupation — show value or placeholder **Not added** (never omit the row).
- Under **Financial:** always show **Salary range**, **Investments**, **Properties** (from `ProfileOptional`) — empty state: **Not added · Tap to add** with verification pill `Not filled`.
- Ensures parity with filter builder and expectations for matrimony depth.

**Financial & Health trust cards (summary, not raw documents):**
- **Financial trust card** (dedicated rounded module below Personal/Financial rows):
  - Headline: e.g. "Financial verification"
  - Summary lines derived from verification pipeline (see product vision): e.g. "Salary: Verified via offer letter · IT filing: 2 years on file · Bank: Pending · Investments: Not submitted · Properties: Layer check — Not started"
  - Only **status labels** and high-level categories — never display bank statements, salary numbers from documents, or full file names publicly.
- **Health trust card:**
  - **Physical:** e.g. "Recent report card: Verified / Pending / Not submitted" (maps to lab report validation).
  - **Mental:** e.g. "In-app assessments: Complete · External consultations: 0/2" (maps to tests on app + two psychology consults).
- Each card links **Learn more** → Trust Card detail (Screen 13) or Verification Center.

**Rest:**
- Other optional sections with verification status pill per field (green / orange / grey as in mocks).
- Verification status banner: "X fields verified, Y pending"
- CTA: "Go to Verification Center"

---

### Screen 13 — Trust Card Detail (tapped from any profile)

Header: "Trust overview — [Name]"

Rows:
```
Identity          ✓ Verified
Physical          Height: Verified | Weight: Not provided
Financial         Salary: Provided (not verified) | Others: Not provided
Health            Physical: Provided | Mental: Not provided
Lifestyle         Filled 4 / 5 fields
```

Footer note: "Documents are never shown publicly. Only verification status is displayed."

---

### Screen 14 — Services Marketplace

Header: "Optional services"

Cards (one per service):

**Identity Verification**
- Description: "Get your government ID verified by our partner."
- Price: ₹XXX
- CTA: "Book"

**Financial Verification**
- Description: "Verify salary, investments, and properties."
- Price: ₹XXX
- CTA: "Book"

**Health Verification**
- Description: "Get your health summary verified."
- Price: ₹XXX
- CTA: "Book"

**Physical Verification**
- Description: "Get height and physical details verified."
- Price: ₹XXX
- CTA: "Book"

**Profile Photography**
- Description: "Professional profile photos."
- Price: ₹XXX
- CTA: "Book"

**Wedding Planner**
- Description: "Connect with our planning partners."
- Price: ₹XXX
- CTA: "Book"

On "Book": Razorpay payment sheet opens.

---

### Screen 15 — Settings + Safety Center

**Implementation rule:** Every row below must **do something** in the prototype: toggle updates local/app state (and would call API in production), navigation rows open a real sub-screen or modal, not dead UI.

**SEARCH MODE**
- **Profile visibility** (label may match "Search Mode" in copy): toggle **Active / Inactive** ↔ `User.search_mode`. When off, user is excluded from others' discovery; show short confirmation on first toggle if needed.
- Persist toggle across sessions in demo; reflect same state on Discovery top bar.

**PRIVACY** (navigation rows with working next screens)
- **Photo visibility:** sub-screen with options (e.g. matches only / all discovery / custom). Save returns to Settings with selection summarized under the row.
- **Click-to-reveal fields:** sub-screen listing sensitive categories (financial, health, family details) with per-category toggles and **confirm** pattern (e.g. "Reveal salary only after match") — store preferences in user settings model for prototype.

**NOTIFICATIONS**
- **Push notifications:** toggle — controls `notifications.push_enabled` (demo state); when off, show inline note that match alerts will not ping device.
- **Email notifications:** toggle — same for `notifications.email_enabled`.

**Optional settings rows (recommended for Serenity):**
- **Messaging / Connect:** sub-screen to edit `UserContactPreference` (enable WhatsApp using registered mobile or override; Telegram username; Instagram handle) with short privacy note.
- **Default filter reminder:** if today's filters not locked, one-tap deep link to Screen 6.
- **Verification status:** row → Verification Center / Services (Screen 14).

**SAFETY CENTER**
- **Blocked users:** navigates to list; each row **Unblock** works and removes from list in demo state.
- **Reports submitted:** read-only list with status chips (submitted / reviewed / closed); empty state copy.
- **Get help:** opens support modal or mailto / form with subject prefill; primary button must be tappable.

**Layout suggestions:**
- Grouped inset lists (iOS Settings style) with section headers matching mocks (SEARCH MODE, PRIVACY, NOTIFICATIONS, SAFETY CENTER).
- Haptics optional on toggle success.

---

### Handoff block — copy for AI / vibe-coding tools

Use this verbatim when regenerating UI:

1. **Settings:** All toggles and rows must be wired: Profile Visibility ↔ search mode; Privacy rows open sub-screens for photo visibility and click-to-reveal rules; Push/Email toggles persist; Safety rows navigate to lists/forms with working unblock and help.

2. **Profile (self):** Provide **Edit profile** and/or per-field navigation to editor. Always show **Caste**, **Salary**, **Investments**, **Properties** rows with empty states. Add **Financial** and **Health trust cards** with verification *summary* only (salary/IT/bank/investments/properties pipeline; physical report card + mental in-app + 2 doctor consults), linking to Trust detail — no raw documents.

3. **Search filters:** **Age** and **Height** must use editable range controls (sliders or min/max inputs). List **all** supported filters; unfilled profile fields show disabled rows with field-specific "Add X in profile" messaging (not generic bio text); optional `Off` indicator.

4. **Discovery:** **Tap card or explicit View profile** opens read-only expanded profile / trust view before Skip or Send Request.

5. **Layouts:** Accordions or category sub-pages for filters; half-sheet profile preview from discovery; Settings grouped sections with functional navigation.

6. **Connect:** No in-app chat thread; Connect screen (Screen 11) uses deep links only; Settings includes editable contact channel preferences.

---

## PART 7 — USER STORIES

### Auth
- As a new user, I can register with my mobile number and OTP so that I have a secure account.
- As a returning user, I can log in with OTP so that I don't need a password.

### Profile
- As a user, I must fill all mandatory identity fields before I can browse discovery.
- As a user, I can fill optional sections at any time without being blocked.
- As a user, I can see a verification status pill beside every optional field.

### Filters
- As a user, I can only enable a filter for a field I have filled in my own profile.
- As a user, I can only enable verified-only filtering if I myself am verified on that field.
- As a user, I can lock my filters once per day and they stay active until next day.

### Discovery
- As an active user, I can see discovery cards matching my locked filters.
- As a user, I can send up to 3 requests per day and am shown remaining count.
- As a user, I cannot send requests if I am in inactive mode.

### Requests and Matching
- As a user, I can optionally answer up to 5 preset-template questions when sending or accepting a request.
- As a user, I can see all submitted question answers on the request.
- As a user, I cannot hold more than 3 active matches at once.
- As a user, I can soft pause a match to free up a slot while keeping history.
- As a user I can see when a match has been paused by the other person.

### External messaging (Connect)
- As a matched user, I can open WhatsApp, Telegram, or Instagram to talk with my match when the match is active and contact sharing is enabled.
- As a user, I configure which channels I expose (and optional WhatsApp number override) before or after matching.
- As a user, contact handoff is hidden when a match is soft paused or ended.
- As a user, I can report or block from the Connect screen (and from profile flows).

### Safety
- As a user, I can block any profile and they will not appear in my discovery.
- As a user, I can report abuse and track my report status.

### Services
- As a user, I can book any verification or service independently.
- As a user, I can pay via Razorpay and track my booking status.

---

## PART 8 — EDGE CASES AND RULES MATRIX

| Scenario | System behavior |
|----------|-----------------|
| User tries to send 4th request today | Hard reject, show count remaining = 0 |
| User tries to accept a 4th match | Reject, show "Soft pause one match to accept" |
| User answers only 3 questions | Submission still allowed; optional flow continues |
| User submits 6 questions | Reject with validation error (max 5) |
| User submits unknown question key | Reject with validation error (invalid key) |
| User tries verified-only filter without being verified | Toggle disabled, hint shown |
| User tries filter on unfilled field | Filter row disabled, hint shown |
| Both users in match soft-pause | Both paused states persist independently |
| User A reactivates, User B still paused | Match stays in paused state (B's pause is active) |
| User B blocks User A | A's request to B silently fails; A is removed from B's discovery |
| User re-requests declined user within 30 days | Blocked silently at API |
| Auto-pause email already sent | Do not send second email; only show login banner |
| User reactivates search mode | Clear auto-pause flag, profile re-enters discovery |
| User deletes profile photo | Block until new photo uploaded (mandatory field) |
| User reports abuse from Connect / profile | Report stored; reviewed by moderation (no in-app message content to moderate) |
| User books service, payment fails | ServiceBooking.payment_status stays 'pending'; retry option shown |
| Verified field edited after verification | Set FieldVerification.status back to 'not_verified', require re-verification |

---

## PART 9 — NOTIFICATION EVENTS

| Trigger | Channel | Message |
|---------|---------|---------|
| New incoming request received | Push | "Someone is interested in you. View request and respond." |
| Request accepted | Push | "[Name] accepted your request. Connect when you’re ready." |
| Request declined | Push | "A request was not accepted. Keep exploring." |
| Active match reminder (optional) | In-app or push | Generic copy only (e.g. "You have an active match") — **no** off-platform message preview (Serenity does not receive third-party chat content) |
| Match soft-paused by other | Push | "Your match has paused this connection." |
| Match reactivated by other | Push | "[Name] has reactivated your match." |
| Profile auto-paused | Push + Email (once) | "Your profile has been paused. Log in to reactivate." |
| Login while paused | In-app banner | "Your profile is paused. Tap to reactivate." |
| Optional questions available | In-app | "You can answer up to 5 compatibility questions." |
| Service booking confirmed | Push + Email | "Your [service] booking is confirmed." |

---

## PART 10 — BUILD ORDER

### MVP (ship first)

1. Auth (OTP login)
2. Identity setup + mandatory field gate
3. Optional profile fields + verification status pills
4. Discovery feed (basic, no filters yet)
5. Request caps + optional questions flow (max 5)
6. Match creation + active cap enforcement
7. Soft pause + transparency
8. `UserContactPreference` + Connect handoff (GET `/matches/:matchId/contact-handoff`, deep links)
9. Safety basics (block + report)

### V2

10. Filter Builder + lock logic
11. Fair-filter enforcement in UI and API
12. Verified-only filter toggle
13. Trust Card + click-to-reveal sensitive fields
14. Auto-inactive cron + login banner
15. Notifications (push + email)
16. Services marketplace + Razorpay integration
17. Verification workflow (admin-side + field status updates)
18. Moderation queue (reports; off-platform abuse handled as policy + user reports, not message logs)

---

## PART 11 — NON-FUNCTIONAL REQUIREMENTS

- Government ID numbers must be encrypted at rest (AES-256 or equivalent).
- Sensitive profile fields (financial, health) must be transmitted only over HTTPS.
- Media uploads (photos) must be stored in private buckets; serve via signed URLs.
- No raw documents are ever exposed via API; only verification status is returned.
- JWT tokens must expire in 7 days; refresh token flow required.
- Daily request count reset must be atomic (no race condition on midnight reset).
- Active match count check must use a database-level transaction when accepting.
- All user-reported content must be retained for compliance and review.
- GDPR/DPDPA: users must be able to download their data and delete their account.
- **External messaging:** Conversations on WhatsApp, Telegram, or Instagram are outside Serenity’s control; the app must not claim to monitor or moderate those threads. Application secrets (SMS, email, Razorpay, storage, JWT) remain server-side per **Section 2.2**.

---

## PART 12 — OPEN (DECIDE BEFORE V2)

- Verification partner / vendor (identity, financial, health)
- Legal / compliance review for storing financial and health data
- Pricing for each service in the marketplace
- **Instagram / Meta:** review current Messaging / profile linking policies if product copy promises more than “open profile or copy handle.”
- **WhatsApp Business Cloud API:** only if product later needs server-initiated templates or business messaging—not required for `wa.me` handoff MVP.
