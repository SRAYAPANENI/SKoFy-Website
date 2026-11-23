# Online jobs — design

Built 2026-11-23 (migration 119). The job shape's `venue = ONLINE` (JOB_SHAPE_DESIGN.md).

## 1. What an online job is
- **Nobody travels and nothing is visited.**
  - No map, no trip, no arrival code, no inspection, no visiting fee.
  - The backend's steps for it were already right (`job_policy._steps`):
    - priced by quote: `QUOTE → COMPLETE → RATE_BOTH_WAYS`;
    - paid up front: `PAY_UPFRONT → COMPLETE → RATE_BOTH_WAYS`.
- **Matched by skill and trust score alone, anywhere** (`OnlineMatching`). Job details says "Online — open to providers anywhere" (`matching.reach_km` → none).
- **The commitment is the price, not a visit fee.**
  - A quoted online job: the provider sends a price from the start, and the customer pays it into escrow before any work begins.
  - `fees.visiting_fee(online=True)` is always 0. Before this, the hired provider's own visiting fee was copied onto online jobs.

- **Where the provider is stays theirs** (`member_travels`, app/domain/business.py). The same holds for work at their own place.
  - The provider app follows a trip only for a job they travel to.
  - The server passes on no position: not on `/jobs/{id}/track`, not on the job socket (older apps still send it), not on the business's team map.

## 2. The customer's screen
- **In place of the map:** an "Online session" panel with the session time, and "join from your phone, or a computer to share screens" (`track-provider.tsx`).
- **Steps** (`services/tracking-stage.ts`, `online`):
  - quoted: Booked → **Price** → Work → Done. Before the price it says "Waiting for the price".
  - up front: Booked → Session → Done.
- **No visiting-fee prompt.**
  - After hiring, the shared hire flow (`useHireFlow(…, online)`) says the price comes next.
  - The applicant's profile hides "Visiting Fee".
- **Home card:** "Hired · online", then "Work in progress" — not "Provider on the way".

## 3. Meeting links (Path A)
Zoom, Google Meet, Microsoft Teams and Webex already do screen sharing, run on laptops (Meet and Teams in the browser) and on phones. Dodorez pins the link to the job and makes it easy to open.

- **Who:**
  - The customer or the hired provider can add, change or remove a link (`PUT`/`DELETE /jobs/{id}/meeting`). The other side is pushed (`meeting_link`), and both open screens refresh at once (`meeting_changed` on `/ws/jobs/{id}`).
  - Allowed while the job is ACCEPTED, INVOICE_PENDING or IN_PROGRESS.
  - Only those two see it: `JobResponse.meeting`, from `get_job`.
- **Which links** (`app/domain/meeting_links.py`):
  - `https` only, on zoom.us, zoomgov.com, meet.google.com, teams.microsoft.com, teams.live.com or webex.com (and their subdomains).
  - No user part (`https://zoom.us@evil…`), no port other than 443, no surrounding text, at most 500 characters.
  - A link anyone is asked to open is never a way to phish them.
- **On a computer:** each link gets a short code. `<PUBLIC_BASE_URL>/s/<code>` redirects to the meeting (`app/api/short_links.py`, at the site root so it's short to type).
  - 7 characters, without 0/O/1/I/L; typed loosely ("k7p-2qx a") still works.
  - Works only while the job is open; afterwards it says "This session has ended".
  - The code reveals nothing beyond the meeting link itself. Each address may try 30 codes per 10 minutes.
  - The code is kept when the link changes, so an address already typed keeps working. Removing the link removes the code.
- **Calendar:** no email needed (the backend has no email sender yet).
  - **iPhone:** `/s/<code>/calendar.ics`, a standard calendar file with the time, the link and a 10-minute alarm. Long lines are folded to 75 bytes, as the standard requires, because Teams links run to hundreds of characters. Safari hands it to Calendar.
  - **Android:** Google Calendar's own "add event" page, which adds it in one tap (a calendar file would only download).
  - Either way it syncs to a laptop's Google, Outlook or Apple calendar.
- **Repeat bookings:** when an online visit is confirmed (`SeriesService.after_step` HIRED), it takes the link the series used last (`MeetingService.carry_series_link`), so a weekly lesson keeps its room and its reminder. A visit that already has its own link keeps it.
- **Reminder:**
  - `remind_online_sessions` (every 5 min) pushes both sides once, 10–15 min before a session with a set time and a link (`meeting_reminded_at`).
  - Changing the link resets it, so the reminder carries the new one.
- **Whose room it is:**
  - A business gives an online job to a teammate: the previous member's link comes off, and the customer's own stays.
  - A job opened again after a cancellation starts without a link.
  - A repeat booking only carries its own provider's (or the customer's) link.
- **Where it shows:**
  - customer: the meeting card in the tracking sheet (`components/meeting-card.tsx`);
  - provider: the Service Room, in place of "Navigate to Site". Same card in both apps.

## 4. Next (not built)
- **Deliver → approve** before release (JOB_SHAPE_DESIGN.md §9 question 6): the provider marks the work delivered with files, and the customer approves or asks for changes, with an auto-approve window. Needs a decision on the window and on how many revision requests are allowed.
- **Path B — sessions inside Dodorez:**
  - a web session room on the self-hosted LiveKit server (video, screen share, chat, files), with a laptop signed in by scanning a QR code with the phone;
  - keeps sessions on-platform and allows recordings, with consent, for disputes;
  - needs a bigger media server (video and screen share use far more bandwidth than the current voice-only `dodorez-media`), video storage, iOS screen-share work, and the lawyer on recording video (LEGAL_QUESTIONS.md).
