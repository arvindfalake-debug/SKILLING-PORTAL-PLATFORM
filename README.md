# Skilling Outcomes Portal — Complete Prototype

Built for the SIH problem statement: **"Difficulties in tracking employment
outcomes, skill gaps, and the impact of skilling initiatives."**

---

## 1. The real government problem

Government skilling programmes (like PMKVY, state ITI schemes, or a
college-run initiative) spend crores every year training candidates. What
they track well:

- Enrolment
- Attendance
- Assessment scores
- Certification

What they **lose track of the moment training ends**:

- Did the candidate actually get a job?
- Self-employed? Doing an apprenticeship? Still searching?
- What's their wage — did the training actually improve their income?
- Are they still employed 90 days later, or did they drop out of the job?

**Why this happens:**
1. Trainees change phone numbers or move cities after the course.
2. Employers don't report back to training centers — there's no incentive to.
3. Every scheme/state/center uses different candidate identifiers, so
   nobody can match "this trainee" across systems.
4. There's no independent way to verify a training center's claimed
   placement numbers — centers are financially motivated to over-report
   placements to keep getting funded, and there's no cross-check.
5. Duplicate/ghost enrolments (fake candidates, or one person registering
   multiple times) inflate enrolment numbers and siphon training funds,
   and nothing currently catches this automatically.

**The consequence:** the government can't compare which training centers
or courses actually work, can't tell which skills have genuinely low
market demand vs. which centers are just under-delivering, and ends up
re-funding the same underperforming programmes year after year — because
there's no credible outcome data to decide otherwise.

---

## 2. The solution, module by module

This prototype treats the problem as five connected modules, all built and
wired together in this codebase:

### A. Golden ID (`candidate/register.html`, `training-center/register-candidate.html`)
Every candidate gets **one Aadhaar-verified identity** the moment they
register — a Golden ID (e.g. `SKN26AIML007`) that stays with them from
enrolment through to their employment outcome, regardless of how many
times they change phone numbers. Registration requires both an Aadhaar
OTP check **and** a simulated DigiLocker linking step (`mockDigilockerLink`
in `app.js`) — a real deployment would redirect through DigiLocker's OAuth
flow here. Two ways to create this record:
- **Self-registration** — candidate registers themselves.
- **Training-center-assisted registration** — the training center registers
  a walk-in candidate who doesn't have their own device, and hands them
  the generated ID/password afterward.

Each candidate's record tracks an `ongoingCourses[]` and `finishedCourses[]`
list (not just one course), so their dashboard can show a course/certificate
vault rather than a single status line. The dashboard also shows a **skill-gap
match score**, a **career guidance** card, and a **recommended courses**
card — all static lookups in `app.js` (`JOB_MARKET`, `STRONG_SKILLS`,
`recommendedCoursesFor`), standing in for the AI models a real version
would use.

### B. Self-Report + Contact-Verified Confirmation (`candidate/dashboard.html`, `training-center/dashboard.html`)
The candidate can self-report their employment status any time after
completion — but a self-report alone is never trusted as final. It stays
**"Unverified"** until confirmed either by:
- a code sent to the candidate's email/SMS (simulated here), or
- the training center **calling or emailing the candidate directly** and
  logging that contact (method, date, and a note) before the status can
  be saved — the training-center dashboard's update form simply refuses
  to save an employment change without that log, or
- (production) an EPFO/e-Shram check — see module D.

This closes the "training centers over-report placements" gap: nothing
counts as a verified placement until it's confirmed by a source other
than the center's own guess, and every verified record carries a trail of
*how* it was confirmed.

Each candidate's dashboard also shows the **3/6/12-month tracking window**
(`followupCheckpoints` in `app.js`) — three checkpoints dated from
enrollment, each showing due/upcoming/recorded status. Verifying a status
update automatically fills in whichever checkpoint is currently due, and a
simple bar chart plots wage across the three checkpoints as a quick
progress view.

### C. Government Analytics, view-only (`policy-maker/dashboard.html`)
A single dashboard rolls up every candidate across every training center
into the indicators a policy maker actually needs to make funding
decisions: employment rate, average wage, 90-day retention, verified vs.
pending vs. unreachable outcomes, and — critically — **provider
performance, district performance, and skill-gap analysis** (which
courses have the lowest placement rate, i.e. where the curriculum or
market fit is weakest). Every chart is clickable and drills into the
underlying candidate list (read-only detail view — policy-maker accounts
cannot edit a record), and nine filters (state, district, provider,
course, gender, age, category, year, employment status) let a policy
maker slice the data any way they need.

Training centers get their **own**, narrower dashboard
(`training-center/dashboard.html`) scoped to only the candidates enrolled
through their center — this is where employment records actually get
updated, always through the contact-logging flow described in module B.
It also shows demographic analytics (gender/category breakdown) and an
enrollment count per course, standing in for a course catalog.

### D. Fraud & Duplicate-Registration Detection (`policy-maker/fraud-detection.html`)
Directly addresses ghost enrolments: the system automatically clusters
candidates who share a phone number or an IP/device, and flags them for
investigation *before* they're certified or paid for. In this prototype,
`app.js` seeds a deliberate 5-candidate cluster on the same phone+IP so
you can see the detection working immediately — in production this same
logic would run continuously against real registration data.

### E. Placement Funding Transparency (`policy-maker/funding-epfo.html`)
Models the idea of **not paying training centers on the strength of a
self-submitted placement report**. Each candidate's funding milestone
sits in a simulated escrow; running the simulated "EPFO check" mimics
finding a new EPF (provident fund) deposit from a verified employer,
which auto-releases that candidate's milestone and logs it to a
transaction ledger — a smart-contract-style release with no manual
sign-off needed once the outcome is independently confirmed.

---

## 3. How the modules connect (end-to-end flow)

```
CANDIDATE                    TRAINING CENTER                 POLICY MAKER
──────────────────           ──────────────────               ──────────────────
register.html  ──┐
                  ├─→ Golden ID + password generated
register-        │   (training center can also do this
candidate.html ──┘    directly for walk-in candidates,
   (center-side)       locked to their own center)
        │
        ▼
   login.html                    login.html                      login.html
        │                    (per-center credentials)        (government credentials)
        ▼                             │                                │
  dashboard.html                      ▼                                ▼
  (own profile,                dashboard.html                   dashboard.html
   self-report                 (OWN candidates only,             (ALL candidates, all
   employment,                 update employment ONLY             centers, 9 filters,
   verify via                  after logging a call/email          clickable charts,
   email/SMS,                  with the candidate)                 read-only detail)
   career                              │                                │
   guidance)                          │                                ├──→ fraud-detection.html
        │                             │                                │
        │  status saved as            │  status saved as               └──→ funding-epfo.html
        │  "Unverified" until         │  "Verified" — contact
        │  confirmed                  │  log required
        ▼                             ▼
  Verified ✓ ─────────────────────────┘
```

All pages read and write the **same candidate records** (`assets/app.js`
→ `localStorage`), so an action on one page (e.g. a candidate self-reporting,
or a training center logging a verified update) instantly shows up
everywhere else it should (the training center's own table, the
policy-maker's aggregate view, the funding page's "eligible for EPFO
check" dropdown) — exactly like a real shared database would behave,
just running locally for the demo.

---

## 4. File map

```
skilling-portal/
├── index.html                        Landing page — routes to all three roles
├── assets/
│   ├── style.css                     Shared visual design (all pages)
│   └── app.js                        Shared data layer: candidate store,
│                                      ID generation, stats, fraud detection,
│                                      funding logic, job-market lookup,
│                                      per-role credentials, seed demo data
├── candidate/
│   ├── register.html                 Self-registration + Aadhaar OTP + ID generation
│   ├── login.html                    Candidate login
│   └── dashboard.html                Own profile, self-report + verify employment,
│                                      career guidance card
├── training-center/
│   ├── login.html                    Per-center login (separate creds per center)
│   ├── dashboard.html                OWN candidates only; employment updates require
│   │                                  a logged call/email before saving
│   └── register-candidate.html       Center-assisted registration (walk-in candidates),
│                                      locked to the logged-in center
└── policy-maker/
    ├── login.html                    Government login
    ├── dashboard.html                All candidates, all centers, indicators,
    │                                  clickable charts, filters — read-only detail view
    ├── fraud-detection.html          Duplicate phone/IP clustering — ghost-enrolment watch
    └── funding-epfo.html             Escrow, simulated EPFO check, smart-contract-style release
```

## How to run it

No build step. Open `index.html` in a browser (double-click, or serve the
folder with any static server). Data is seeded automatically into
`localStorage` on first load — 25 demo candidates across 3 training
centers, including a deliberate fraud cluster so the fraud-detection page
has something to show immediately.

- **Candidate:** register fresh, or note there's no pre-made login since
  every browser starts with fresh seed data — register once and use those
  credentials.
- **Candidate:** register fresh (generates a Golden ID + password), or log
  in with an existing one. Forgot your password? Use "Forgot password?" on
  the login page — OTP `000000` for this demo.
- **Training center:** ID `TC-SKN`, password `Center@123` (see
  `training-center/login.html` for the other two centers' IDs). Forgot
  password → OTP `000000`.
- **Policy maker:** ID `GOV-POLICY-01`, password `Policy@123`. Forgot
  password → OTP `000000`.

## Important limits of this prototype (read before a demo/pitch)

- **Storage is per-browser, not shared.** Everything lives in that one
  browser's `localStorage`. Opening the site on a different device or
  clearing browser data starts over. This is fine for a demo, not for
  real use — see the production roadmap below.
- Aadhaar verification, OTP delivery, and the EPFO check are all
  **simulated** — no real government API is called anywhere.
- Passwords are stored in plain text for demo simplicity — never do this
  in production.

## Production roadmap (what to build next, in order)

1. **Backend + real database.** Node/Express or Python/FastAPI with
   PostgreSQL, replacing `app.js`'s localStorage calls — keep the same
   field names so the frontend barely changes.
2. **Real Aadhaar verification** via DigiLocker/Aadhaar eKYC through an
   authorized integration (a college/training center cannot call the
   Aadhaar API directly — this needs government-approved access).
3. **Real outreach + verification**: MSG91/Twilio for SMS/IVR + an email
   service; verification cross-checked against EPFO/UAN (formal jobs) and
   e-Shram (informal/gig work) where API access is available.
4. **Auth & security**: hash passwords (bcrypt/argon2), add JWT/session
   auth and role-based access — a training center should only see its own
   candidates; state/district/program-level admins see progressively
   wider, aggregated data (see the earlier discussion on role-based,
   single sign-on-style login for state/district/center/candidate roles).
5. **Consent ledger** as its own auditable table (what was agreed to,
   when, and support for withdrawal) instead of a single boolean field.
6. **Real fraud signals beyond phone/IP**: registration timing patterns,
   biometric attendance mismatches, Aadhaar-linked device fingerprints.
7. **Real EPFO/UAN bridge + actual escrow account**, replacing the
   simulated smart-contract release with a real payments integration.
