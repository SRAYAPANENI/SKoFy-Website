# SkoFy — Tech Debt

> Source of truth is the `[[tech-debt-skofy]]` memory file — this doc mirrors it for offline/grep-able reference. If they disagree, the memory file is more recent; update both together.

## Standing decision

This debt is **intentionally not being fixed yet**. Priority right now is finishing planned features, then integrating third-party KYC/ID verification, *then* circling back to harden. Don't proactively suggest fixing these unless a new feature directly touches the same area, or the user signals they're starting the hardening pass.

## Project review, 2026-10-09

A whole-project review found the issues below. **Fixed the same day** (backend; 908 tests):

- **No-show sweep left the visiting fee in escrow** and, after a re-hire, blocked the new charge or paid it to the no-show provider. The sweep now goes through `JobService.remove_no_show_provider` — the same path as a provider cancelling.
- **One place decides where escrowed money goes** — `PaymentService.settle` + the pure rule in `app/domain/settlement.py`, used after completion, inspection, every cancel and the no-show timer. A **settlement sweep** (`settle_stale_payments`, every 30 min) retries payouts/refunds that failed instead of leaving money HELD forever. Pickup jobs' visiting fee (never released before) is paid on completion.
- **Re-hire after a refund** reused the old Stripe intent (idempotency key now includes the provider).
- **Reopened jobs kept the previous provider's progress** (arrival code/inspection/pickup "done", no-show timer never firing for the next provider) and their unpaid quote. Cleared on reopen.
- **Chargebacks and dashboard refunds** (`charge.dispute.*`, `charge.refunded` webhooks) are recorded (migration 048); nothing is released under an open chargeback. Logged as errors — **register these events on the Stripe webhook endpoint.**
- **Job status changes are compare-and-set** (`JobRepository.transition`, `JobService._move`): a stale screen or a background timer can no longer overwrite a newer change (409 `JobChangedError`).
- **Login and arrival codes**: generated with `secrets`; the wrong-guess cap holds under parallel guesses (`app/core/one_time_codes.py`); new arrival codes capped at 5/hour per job; texts capped per client IP and in total, besides per phone.
- **Stripe SDK calls** run on a worker thread (the API is one process — a blocking call froze everyone).
- **Live socket** now applies logout and single-device sign-out checks (`authenticate_access_token`).
- **Repeating jobs** are refused until series exist (posted as one visit before).

**Still open:**
- **Escrow release rule — needs a product decision.** A provider marking their own job complete is paid immediately, so a dispute after completion can't recover the money (`resolve_dispute` flags `requires_manual_refund`). Options: hold for N hours unless the customer disputes, or release on customer confirmation. `settlement_for` is where the rule goes.
- **Admin sign-in has no second factor** (needs admin-portal screens).
- **Launch ops** (deploy/README.md checklist): automated off-server DB backups, error monitoring/alerting (no Sentry), US region, single server.
- **Single API process** — live sockets are in memory (`ws_manager`); scaling past one process needs Redis pub/sub. Cloudinary uploads still block (item 1 below).
- **Code health**: `job_service.py` ~2,300 lines; provider `profile.tsx` 5,767 / `(tabs)/index.tsx` 3,750, customer `home.tsx` 3,150 (worse than item 6 says); two data-access styles (ORM models + ~200 raw SQL strings); the two apps share no code, so call/API fixes are made twice.
- **Calls run inside the chat screen** (`app/chat.tsx` owns the peer connection, audio and timers in both apps). Phase 2 of docs/CALLING_SYSTEM.md kept it there (a blind rewrite was too risky); moving it to a call service would make calls survive the chat screen being closed.
- Refunded payment rows are deleted when the job is re-charged (unique `job_id, purpose`) — refund history lives only in Stripe.

## The list (priority order)

1. **Blocking I/O inside async handlers** — `provider_service.add_document_from_file`'s local-disk fallback does sync `open().write()` inside `async def`; Cloudinary upload calls are also sync, blocking the event loop. Fix: `asyncio.to_thread` or move to Celery (already used elsewhere in `app/tasks/`).
2. **No upload limits or content validation** — `POST /providers/me/documents/upload` reads the whole file into memory, no max-size cap, trusts client `Content-Type` with no magic-byte check. Higher risk than usual since one document type is a government ID photo.
3. **Race condition on email uniqueness** — `update_profile`/`add_document` do check-then-update, not atomic. DB `UNIQUE` constraint prevents bad data but a concurrent collision surfaces as a raw 500 instead of a clean `409 EMAIL_ALREADY_EXISTS` (no `try/except IntegrityError`).
4. **Zero automated tests** — verification has been manual curl + throwaway scripts. No regression protection.
5. **Unmoderated skill taxonomy growth** — `SkillRepository.get_or_create` lets typos (`"Wireing"` vs `"Wiring"`) permanently fork the `skills` table. No normalization or admin merge tooling.
6. **Frontend god-files** — `skofy-service-provider-app/app/(tabs)/profile.tsx` and `app/register.tsx` are 2,000+ lines each, business logic inline in JSX handlers.
7. **No PII/retention policy on ID documents** — `provider_documents` stores government ID photos with no encryption-at-rest, no access audit log, no retention/deletion policy. Directly relevant once a real KYC vendor is integrated — that's the natural point to redesign this (likely replacing raw ID-photo storage with the vendor's own verification flow).
8. **No admin panel anywhere in the repo.** Currently blocks: dispute resolution (`POST /jobs/{id}/resolve-dispute` is real and tested, but only callable via curl/Postman) and will block manual identity-verification review once that flow exists. First screen to build when this gets picked up, since the backend's already done.

## Related, surfaced separately

- **Legacy `POST /jobs/{id}/start` skips the job's steps** (found 2026-10-09, phase 2 of [JOB_SHAPE_DESIGN.md](JOB_SHAPE_DESIGN.md)). It moves an ACCEPTED job straight to IN_PROGRESS after a location check — no arrival code, inspection or paid invoice. Neither app calls it any more (inspection + invoice replaced it), but the endpoint is still mounted, so a provider calling the API directly could bypass the protocol. Fix: remove the endpoint, or route it through `JobFlow.require_ready`. Awaiting founder go-ahead.

- US-market vendor/locale leftovers from earlier India-oriented build-out (`IndianRupee` icon in provider app, `timezone="Asia/Kolkata"` in `app/tasks/celery_app.py`, likely stray ₹-formatted strings) — these aren't "debt" in the engineering sense, they're a market-pivot cleanup item. Tracked in [VISION.md](VISION.md)'s market-mismatch callout and [ROADMAP.md](ROADMAP.md), not duplicated here.
- The AI Engine gaps (HCI Engine, Hiring Agent, Feasibility Engine, Support AI not built) are **missing features**, not debt — tracked in [IMPLEMENTATION_STATUS.md](IMPLEMENTATION_STATUS.md) and [ROADMAP.md](ROADMAP.md).
