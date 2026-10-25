# Deleting an account

**Status:** built 2026-10-24 (migration 083). Customer and provider accounts
can be deleted. A business owner closes the business first
(BUSINESS_PROVIDERS_DESIGN.md §12, built 2026-10-25), then can delete their
account. The privacy policy at `/v1/legal/privacy` states what's kept.

## Why

- **Apple** (guideline 5.1.1(v)) requires that deletion can be started in
  the app.
- **Google Play** requires the same, plus a web page to request it.
- **California's privacy law** (CCPA/CPRA) gives a right to delete,
  answered within 45 days. It has exceptions for records a business must
  keep (transactions, tax, security and fraud, legal claims).

## The model: close now, remove identity after 30 days, never delete history

A person's row is never deleted. Deleting it would take the other party's
jobs and payments with it. Since migration 083 the database **refuses**
that: the history tables' links to a person are `RESTRICT`, not
`CASCADE`.

1. **Ask.** In either app, Profile → Settings → **Delete account**, or the
   web page `/v1/account/delete`.
   - A code is texted to the number they log in with
     (`POST /v1/account/deletion/code`), then `POST /v1/account/deletion`.
   - It's one account across both apps.
2. **Blocked while anything is open.** Each reason is shown so they can act
   on it (`app/domain/account_deletion.py blockers`):
   - a job posted or booked;
   - a dispute;
   - money held;
   - an open guarantee claim;
   - money owed to Dodorez;
   - a regular booking;
   - running a business: close it first in Business → Close business. While
     business accounts are switched off, the owner can't reach that screen,
     so deleting the account closes the business too, under the same rules.
3. **Closed at once.** Signed out on every phone in both apps, push tokens
   gone, off job matching, pending applications withdrawn. A team member
   leaves their team straight away, without notice.
   - While it's closing, nobody can book them directly (first choice or
     backup) or propose a regular booking to them. Team invites waiting for
     them are cancelled.
4. **30 days to change their mind** (`ACCOUNT_DELETION_GRACE_DAYS`).
   Logging in again cancels it, and the app says "Welcome back"
   (`account_restored` on login).
5. **Then anonymised:** the daily `purge_deleted_accounts` task, skipped
   while an admin has set a **legal hold**. The task checks again on the day:
   a dispute or guarantee claim raised during the 30 days, on a job they'd
   finished, is settled first.

## What's removed and what's kept

| Removed (identifies the person) | Kept, without identity (what happened) |
|---|---|
| Name → "Former customer" / "Former provider" | Every job: what, when, city, price, status, rating |
| Phone → a placeholder; the real number is free to sign up again | The same internal ID on all their jobs, so traction counts still work |
| Email, photo, bio, ID number, ID documents and certificate files | Payments, fees, payouts, refunds, tips (tax and accounting, about 7 years) |
| Saved addresses; their jobs' street addresses and notes; exact job location (rounded to about 1 km) | Reviews and ratings, attributed to the former name |
| Saved cards (the Stripe customer is deleted; Stripe keeps its own legally required records) | Disputes, claims and strikes |
| Device tokens, sessions, notifications | Skill categories and the record of skill tests (answers removed) |
| Their chat and support messages, and their support tickets' subjects → "[deleted]" | |
| Live location, base address; their Stripe payout account is closed | |
| What they typed asking for a service Dodorez didn't offer | |
| Team invites sent to their number; the contact details of a business they closed | |

**Banned people** (an admin disabled them) keep only a keyed fingerprint of
their number (`banned_phone_fingerprints`), so the same number can't sign up
again. Fraud prevention is an allowed reason to keep it.

**No mapping back.** Nothing, in a table or in a log, may map the internal ID
back to the person once they're anonymised. That is what makes the kept data
lawful. The privacy policy must say plainly that deleted accounts' job and
payment records are kept without identity.

**Backups** still hold the old data until they age out. That should be
stated in the privacy policy, with how long backups are kept.

## Admin

- `GET /v1/admin/account-deletions`: closed accounts waiting for their date.
- `PUT /v1/admin/account-deletions/{id}/legal-hold`: stops anonymising for
  an investigation or lawsuit.
- Both need `customers.manage`. Portal: **Account Deletions** in the sidebar.

## Code

- **Rules (pure):** `app/domain/account_deletion.py`.
- **SQL:** `app/repositories/account_repo.py`.
- **Service:** `app/services/account_deletion_service.py`, with the routes in
  `app/api/v1/account.py` and the task in `app/tasks/account_tasks.py`.
- **Login side:** `AuthService.confirm_phone_code`, `sign_out_everywhere` and
  `_restore_if_closing`, plus the banned-number check at sign-up.
- **Team side:** `BusinessTeamService.leave_on_account_closure`.
- **Apps:** `app/delete-account.tsx` in both.
- **Smoke test against a database copy:** `scripts/smoke_account_deletion.py`.

## Still to do

- The lawyer's review of the privacy policy and these rules (LEGAL_QUESTIONS.md §12).
- A real privacy mailbox: `PRIVACY_CONTACT_EMAIL` is a placeholder.
- A backup retention period, so the policy's "backups are replaced" is true:
  `~/db-backups` on the server is kept indefinitely today.
