# VIPTA Global — Package Log (Phase 1)
All setup info, rules, and staff instructions in one place.

═══════════════════════════════════════════════════════════
PART 1 — WHAT THIS IS
═══════════════════════════════════════════════════════════

A small, standalone tool that solves exactly one problem: matching a photo
and weight to the right customer, with an Excel export — nothing else from
the bigger VIPTA Global platform is included here.

What it does:
- Staff enter a shared passcode to get in
- Log a package: customer name (autocomplete from past entries), optional
  reference, weight, photo
- Packages are grouped into "shipments" so a new batch never mixes with
  an old one
- One-click "Export to Excel" downloads the current shipment (or full
  history) as a CSV

═══════════════════════════════════════════════════════════
PART 2 — SETUP (for you, the developer)
═══════════════════════════════════════════════════════════

1. Set the first passcode — the tool ships with a default passcode
   (`vipta2026`, set in `index.html` as `DEFAULT_PASSCODE`) that only
   applies until someone changes it. As soon as the client logs in once,
   they should go to the Settings panel inside the tool and set their own
   real passcode — from that point on it's stored in Firestore, not in
   the code, so they can change it themselves anytime without needing you.

2. Enable Anonymous sign-in — Firebase console → Authentication →
   Sign-in method → enable "Anonymous". This is the only new Firebase
   console step; everything else (Firestore, Storage, Blaze) is already
   set up from the main VIPTA Global app, since this tool reuses that
   same project under the hood (just its own collections and Storage
   folder, so it stays functionally separate).

3. Add the Firestore and Storage rules — see Part 3 below. These are
   ADDITIONS to your existing published rules, not replacements — don't
   remove anything already there.

4. Deploy — push this to its own GitHub repo (e.g.
   `fingerprintacoustic/vipta-package-log`), enable GitHub Pages the same
   way as the main app. This gives it its own separate URL, so it's
   clearly its own deliverable rather than a hidden part of the bigger
   platform.

5. Expect one index prompt the first time — the tool filters packages by
   shipment and sorts by date at the same time, which needs a one-time
   Firestore "composite index." The first time you view a shipment, the
   browser console will likely show an error with a blue link that says
   something like "create it here" — click it, then click "Create Index"
   in the Firebase page that opens, wait about a minute, and refresh.

Why it's built this way:
- Reuses the same Firebase project as the main app — no second Firebase
  setup, no second Blaze enrollment, no extra monthly cost beyond what's
  already running.
- Everything it writes lives in its own `packages`, `shipments`, and
  `packageLogSettings` collections, plus its own `packages/` Storage
  folder — completely separate from the main app's data.
- The shared-passcode gate is intentionally simple (not per-person
  accounts) to keep this a fast, cheap Phase 1 deliverable — and the
  passcode itself is stored in Firestore rather than hardcoded, so the
  client can change it themselves. If it's ever worth upgrading to
  individual staff logins, that's a natural Phase 2/3 addition.

═══════════════════════════════════════════════════════════
PART 3 — FIRESTORE & STORAGE RULES TO ADD
═══════════════════════════════════════════════════════════

These are ADDITIONS to your existing rules — don't replace what's
already published for the main VIPTA Global app. Add these blocks
alongside what's already there, inside the same `service` block.

--- FIRESTORE --- add inside your existing firestore.rules:

    match /packages/{packageId} {
      allow read, write: if request.auth != null;
    }

    match /packageLogSettings/{docId} {
      allow read, write: if request.auth != null;
    }

    match /shipments/{shipmentId} {
      allow read, write: if request.auth != null;
    }

Full context — your firestore.rules should end up looking like:

    rules_version = '2';
    service cloud.firestore {
      match /databases/{database}/documents {
        ...your existing users/settings/requests rules stay exactly as-is...

        match /packages/{packageId} {
          allow read, write: if request.auth != null;
        }

        match /packageLogSettings/{docId} {
          allow read, write: if request.auth != null;
        }

        match /shipments/{shipmentId} {
          allow read, write: if request.auth != null;
        }
      }
    }

--- STORAGE --- add inside your existing storage.rules:

    match /packages/{fileName} {
      allow read: if request.auth != null;
      allow write: if request.auth != null
                   && request.resource.size < 8 * 1024 * 1024
                   && request.resource.contentType.matches('image/.*');
    }

Full context — your storage.rules should end up looking like:

    rules_version = '2';
    service firebase.storage {
      match /b/{bucket}/o {
        ...your existing /weighins block stays exactly as-is...

        match /packages/{fileName} {
          allow read: if request.auth != null;
          allow write: if request.auth != null
                       && request.resource.size < 8 * 1024 * 1024
                       && request.resource.contentType.matches('image/.*');
        }
      }
    }

═══════════════════════════════════════════════════════════
PART 4 — STAFF GUIDE (how to use it day-to-day)
═══════════════════════════════════════════════════════════

GETTING IN
1. Open the Package Log link (bookmark it on your phone or computer).
2. Enter the staff passcode when asked.
3. You're in — no separate login needed.

LOGGING A PACKAGE
Do this for every package (or bundle of the same customer's items,
weighed together), right before it goes into a shipment:
1. Customer name — start typing; if they've shipped before, their name
   will show up so you can just pick it instead of retyping. If new,
   type the full name.
2. Order / reference (optional) — order number or note, if useful.
3. Weight — the actual weight in kilograms, from your scale.
4. Photo — take a photo showing the weight (e.g. the scale display, or
   the package with a visible label). On a phone, this opens your
   camera directly.
5. Tap "Log package".

That's it — the entry is saved immediately, photo and weight already
tied to that customer's name. No sorting through photos afterward.

STARTING A NEW SHIPMENT
Every package you log goes into whatever shipment is "current" — shown
at the top of the page (e.g. "Logging into: Shipment 1"). This keeps
one shipment's packages from mixing with the next one.

When a shipment is fully packed and ready to go:
1. Scroll to the "Shipments" panel near the bottom of the page.
2. Type a name for the next shipment (e.g. "Shipment — 25 July 2026").
3. Tap "Start new shipment".

From that moment on, every package you log goes into the new shipment.
The old one stays exactly as it was.

VIEWING A PAST SHIPMENT
Use the "Viewing shipment" dropdown above the package list to switch
between shipments, or choose "All shipments" to see everything at once.

GETTING THE EXCEL FILE
1. Use the "Viewing shipment" dropdown to pick the shipment you want
   (or "All shipments" for everything).
2. Tap "Export to Excel" at the top of the "Logged packages" list.
3. A file downloads automatically — open with Excel, Google Sheets, or
   Numbers.
4. It includes every package in your selection: customer name,
   reference, weight, which shipment it belongs to, photo link, and
   the date logged.

CHANGING THE PASSCODE
1. Log in as usual.
2. Scroll to the "Settings" panel at the bottom of the page.
3. Type the new passcode and tap "Update passcode".
4. Share the new passcode with staff.
No developer needed — this is fully in your control.

TIPS
- Log the package before it's sealed, while you can still clearly
  photograph it.
- If you make a mistake, there's no edit button yet — just log it
  again correctly and note the duplicate.
- The passcode is shared among staff — no individual accounts needed.
