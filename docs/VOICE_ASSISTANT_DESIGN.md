# Voice assistant: the server asks, the model talks (design)

**Status (2026-10-29):** proposed. Quick fixes are built: the safety check runs alongside the reply, transcription uses Whisper turbo, the catalogue is held in memory, a rejected tool-call reply is retried once, voice requests get 60 seconds, and the silence wait is 1.2 seconds. The redesign below waits for approval.

## Why

Measured on the live server, 2026-10-29:

| Step | Time |
|---|---|
| Silence wait before sending | 2 s (now 1.2 s) |
| Transcription | about 1 s |
| Main AI reply | **13–36 s** |

- **The cause of the slow replies is the Groq free tier:** 8,000 tokens a minute and 1,000 requests a day. Each turn sends about 4,650 tokens, because the prompt carries all 341 professions and how each is booked. That's under 2 turns a minute for the whole app. The paid Developer tier fixes it, at about $0.001 a turn.
- **The questions per job type aren't enforced.** One long prompt asks the model to follow a fixed script (understand → confirm → when → photo → done). Nothing makes it ask a stay's dates, a babysitter's hours, or a pet's details. Voice jobs never get a care sheet.

## The design

The server owns the checklist. The model only does two things:
1. understand what the customer said;
2. phrase the next question.

### 1. Find the profession (first turn)
- The server picks the 5–8 professions that best match what the customer said (`find_skill`-style word overlap on skill names and customer phrases). The model chooses among those, or says none fits.
- The prompt drops from about 4,350 tokens (all 341 professions) to about 600.
- "Coming soon" and the logging of unmatched requests stay as they are.

### 2. The checklist (server, pure: `app/domain/voice_intake.py`)
Built from the profession's pricing type, its places, and what's in care:

| Always | By pricing type | By what's in care |
|---|---|---|
| What's needed, confirmed by the customer | QUOTE / VISIT: when (or "as soon as possible") | A pet: each pet's kind, name, size (dogs), vaccinations; who brings the food |
| Where, only if the skill is done in more than one place | HOUR: start and end time | An item: what, how many, how big |
| A photo, offered (skipped online) | NIGHT / DAY: drop-off and pick-up dates | A home: nothing extra |

Each question is asked once, and answers given early are kept.

### 3. Each turn
- **The app sends:** the conversation, plus the checklist state the server returned last turn. The server stays stateless.
- **The model receives:** the profession, what's known so far, the one next item to ask, and the customer's last message. It returns JSON: `{"extracted": {...}, "reply": "..."}`.
- **The server:**
  - merges and checks the answers with the existing rules (`clean_care_sheet`, `fit_shape`, dates in the future);
  - works out the next missing item;
  - once everything is known, asks the customer to confirm a summary, then returns `job_data` with the shape and the care sheet. The app's contract stays the same.
- **About 1,000 tokens a turn** instead of 4,650. On the paid tier, replies should take about 1 second.

### 4. Unchanged
- The safety check, which runs alongside the reply.
- Languages (the chosen language stays the last instruction).
- `job_data` → the app's review card → posting.
- The typed posting flow.

## Effort
About 2 days:
- the checklist domain and its tests;
- the prompts;
- passing the checklist state through the app;
- keeping all existing voice tests passing.
