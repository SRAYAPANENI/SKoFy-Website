# Skill verification — keeping the test honest

Status: **Phase 1 built 2026-11-21** (migration 116). **Phase 2 — camera tests — built 2026-11-21** (migration 117). **Check levels per skill and tests in Spanish — built 2026-11-21** (migration 118, §4). The face is compared with the ID by a person, never by software, so no biometric data is computed (LEGAL_QUESTIONS.md §16). Backend, provider app and admin portal.

Code: `skofy-backend/app/domain/skill_test_integrity.py` (every rule, pure), `app/services/skill_test_service.py`, `app/services/spoken_answers.py`, `app/services/private_files.py`, `app/repositories/skill_evidence_repo.py`, `app/tasks/skill_test_tasks.py`; provider app `hooks/use-skill-test.ts`, `hooks/use-answer-recorder.ts`, `components/skill-test/`; portal `src/pages/SkillTests.tsx`, `src/components/skilltest/CameraEvidence.tsx`, `src/components/provider/SkillAttempts.tsx`.

## 1. The problem

A skill's "verified" badge feeds ranking and what customers trust. It was earned by a 5-question multiple-choice quiz on the provider's own phone, unwatched. Anyone could search each question, screenshot it into an AI model, or have someone else answer. What made it worse, as built before 2026-11-21:

- **The clock was the phone's.** The 30-second countdown ran in the app; the server never checked it. **Minimising the test paused it** — pause, look the answer up, resume.
- **Quitting was free.** Nothing was written until the fifth answer: kill the app on a hard question and start again, with no attempt counted and no wait.
- **The bank leaked.** After each answer the app was told right/wrong and the explanation. With 40 questions per skill and unlimited retries an hour apart, the whole bank could be learned.
- **Answers were in the same order every time**, so "it's C" could be passed on.

No unsupervised quiz can be made cheat-proof — AI answers trade questions better than most people. So the goal is **cheating doesn't pay**: slow and visible, never decisive on its own, with a person deciding any pass the signs don't trust, and real work confirming the badge over time.

## 2. Phase 1 — what's built

### 2.1 The server keeps the clock
Each question is issued with its own time (`issued_at`) and has `QUESTION_SECONDS = 30` plus `GRACE_SECONDS = 5` for the network. An answer after that is `TIMED_OUT`, whatever the phone said. The phone counts down from the `seconds_left` the server sends; there is no pause and no minimise.

### 2.2 Starting is an attempt
`GET /skills/{id}/test/question` (no session) writes the attempt as `IN_PROGRESS` and commits it **before** the first question is shown. It ends as:
- `COMPLETED` — five answers;
- `ABANDONED` — closed (`POST …/test/abandon`, after the app warns "it counts as a try"), or never finished. A test whose session is gone is ended as abandoned the next time they start, before the rules are asked, so it counts. An hourly sweep (`close_unfinished_skill_tests`) ends any still open after an hour, so none stays "open" in the portal for good. Until then, the retake rules already count it as the fail it's about to be.

A dropped network or a restarted phone **resumes** the same test (the session lives in Redis for 30 minutes, found again by `skill_test_active:{provider}:{skill}`). Its clock keeps running meanwhile, so resuming never buys time. Starts are locked per provider and skill, so a double tap doesn't open two.

### 2.3 Leaving the app loses the question
The app reports `LEFT_APP` (`POST …/test/event`) when it goes to the background mid-question, and again when it comes back (the first may never get out). The server marks that question lost, counted once. The answer sent afterwards is graded `LEFT_APP`, wrong whatever was chosen. The rules screen says so before the test starts — including that a phone call counts.

### 2.4 Screens
`expo-screen-capture` blocks screenshots and screen recording on Android and hides the screen from recordings on iOS. An iOS screenshot is reported (`SCREENSHOT`).

### 2.4.1 What the reviewer sees of it
- Each `LEFT_APP` and `SCREENSHOT` is kept once, in order, with its time, its question and the phone (`ios` / `android`, sent by the app at the start and with each event). It's stored as `integrity.events` with `integrity.platform`, at most 50.
- The portal's review page shows them as a timeline under Signs ("12:03:05 · Screenshot · on question 3 · iPhone").
- The provider's page shows each try's counts and phone.
- Only iPhones report screenshots, because Android blocks them outright.

### 2.5 Nothing leaks during the test
The answer response carries `counted` (`ANSWERED`, `TIMED_OUT`, `LEFT_APP`, `ALREADY`), never whether it was right. `is_correct` is always null now, and the explanation is empty. Only the score is told, at the end. Answers are shuffled per attempt (seeded by attempt and question, so a resumed test doesn't reshuffle). A repeated answer, sent after a dropped response, carries `question_number` and is never counted twice.

### 2.6 The bank grows
When a provider has fewer than 10 unseen questions left at their level, `grow_question_bank` (Celery) adds a batch of 40. The existing questions are named to the model so it doesn't repeat them, and near-duplicates are dropped. Growth is capped at 400 per skill and guarded to one run per skill every 10 minutes. Questions are now asked as on-the-job situations, not definitions a search answers in one line.

### 2.6.1 Questions written ahead of time, on their own queue
Writing a bank takes 10–60 s on the live model, longer than the app waits on a request, so a test never waits for it.
- **When it's written:** in the background when a skill is added (at most 3 per request, so a big list never floods the AI), when the test screen opens (`GET /skills/{id}/test/prepare`), and hourly for the 10 most-listed untested skills (`prepare_listed_skill_tests`).
- **Not ready on Start:** the server answers `TEST_PREPARING` (409); nothing starts and nothing counts. The app waits, checks every 4 s, and starts by itself.
- **Camera tests:** the multiple-choice and spoken banks are written at the same time.
- **Its own queues, two lanes** (2026-11-23, after a provider waited ~12 min behind 18 queued skills while Groq returned 429s). Neither lane can hold up job alerts and other pushes on the default queue.
  - **Fast lane:** `ai_now`, worker `celery_ai_now`, 2 at once. It's for a test someone has open (the test screen's prepare, or Start).
  - **Writing ahead:** `ai`, worker `celery_ai`, one at a time. It covers a skill being added, the hourly job and topping up a bank. Running more at once only hits Groq's rate limit.
- **Never queued twice:** everything is queued through `queue_test_writing` (`app/tasks/skill_test_tasks.py`).
  - Writing ahead is queued once per skill and language, marked by `skill_q_queued:*`.
  - The fast lane is queued once per skill and language, marked by `skill_q_now:*`, and listed with its start time in the sorted set `skill_q_now_all`.
  - Both marks are cleared when the run ends, whether it worked or failed, so a failed run is tried again the next time it's asked for. Each mark also has its own expiry, in case a run is lost: 30 min for writing ahead, 10 min for the fast lane. If handing the task to the queue fails, the mark is removed straight away.
- **Writing ahead gives way:** while the fast lane holds anyone queued in the last 10 min, writing-ahead runs (`prepare_skill_test`, `grow_question_bank`) wait 20 s and check again. They do this at most 15 times (5 min), so they can never stall for good.
- **Never written** for no-test skills or Pickup & Delivery.
- **Test accounts** (`SKILL_TEST_ANSWER_HINT_PHONES`, empty by default) see the right answer, or a spoken question's marking guide, in the question text. This is for trying a pass on purpose; empty it before launch.

### 2.7 The signs, and what happens to a pass

| Sign | Meaning | Weight |
|---|---|---|
| `LEFT_APP` | went to another app during a question | strong |
| `SCREENSHOT` | screenshot / screen recording | strong |
| `TOO_FAST` | 2+ right answers in under 3 s — quicker than reading | weak |
| `EVEN_TIMING` | 4+ answers all ≥ 10 s and within 1.5 s of each other — the rhythm of looking each one up | weak |
| `LATE_ANSWERS` | most of 3+ right answers in the last 6 seconds | weak |

| A pass with… | Review | Badge |
|---|---|---|
| a strong sign, or two weak ones | `HELD` | **not** given until a person approves. The provider sees "Passed — being reviewed" |
| one weak sign, or none but picked at random (1 in 20) | `SPOT_CHECK` | given now; a person looks later, and turning it down takes it away |
| nothing | `NONE` | given now |

A fail is never reviewed. The signs, answer times and their spread are kept in `skill_test_attempts.integrity`. The answers chosen stay in `answers`, which an account's removal clears.

### 2.8 Taking it again
- Fails in a row wait **1 hour, then 1 day, then 7 days** (an abandoned test is a fail; a pass resets the count).
- A pass turned down after review waits **7 days from the decision**.
- A held pass can't be retaken while it waits (`under_review`).
- An upgrade test still needs 3 completed jobs in that skill since the last pass that counts.
- **An admin can allow one retake now** (`POST /admin/providers/{id}/skills/{skill}/retake`, stored as `provider_skills.retake_granted_at`). It skips the wait once, because it only counts if it's newer than the last attempt. It's for a test cut short by something that wasn't their fault.

### 2.9 The portal
**Skill Tests** (sidebar, badge count; `providers.verify` decides, `providers.view` can look) lists held passes first, then spot checks, oldest first. Each attempt shows every question, the answer chosen, the right one, the seconds taken against 30, the signs and the provider's other tries. **Approve** or **Turn down** — a note is required to turn one down, and the provider reads it. Each decision notifies the provider (`skill_test_update` opens their Verification section). A provider's page lists each skill's tries with how they ended, any review and the signs, plus **Allow a retake now**.

| Decision | Held pass | Spot-checked pass |
|---|---|---|
| Approve | the skill is verified now; HCI recalculated | stays as it is |
| Turn down | counts as a fail; retake in 7 days | badge withdrawn (meter capped at 0.3); HCI recalculated; retake in 7 days |

### 2.10 What reads attempts
HCI learning velocity counts finished tests only, never abandoned, open or turned down. Admin stats leave out open tests. The dashboard prompt shows "being reviewed" for a held pass and "Resume the test" for an open one.

## 3. Phase 2 — camera tests (built)

Decided 2026-11-21: **a person compares the face with the ID by eye** (no face matching by software, so no face geometry is computed or stored), **an admin picks the skills** per skill, and camera tests include **spoken answers**.

### 3.1 Which tests
Skills whose check level is `CAMERA` (§4). `SKILL_TEST_PROCTORING_ENABLED` turns camera tests into plain tests everywhere at once.

### 3.2 The test
1. **Rules, then consent.** A screen says what is recorded, who sees it, that a person (not software) compares the face, and when it's deleted. The provider ticks "I agree". The start request carries `consent=true` and the attempt keeps `consented_at`. Without it the server refuses with `CAMERA_CONSENT_REQUIRED`: an older app shows the message, and a current one opens the consent screen. A camera test is never started silently, even when resuming one that had ended.
2. **The face.** The front camera, with an oval guide, takes a photo of their face, which they can retake. It's sent as `FACE_START` once the test has opened.
3. **5 multiple-choice questions** as in §2, with a small live preview of the camera in the corner (so they always know it's on). It takes a photo every 15 seconds (`PHOTO`), with no shutter sound.
4. **2 spoken questions**, each a job situation answered out loud: 90 seconds per question, recordings stop at 60 s. Each clip is kept, transcribed (Groq `GROQ_TRANSCRIBE_MODEL`) and scored 0–1 against what a good answer covers (`skill_spoken_questions.rubric`). The rubric is written by the model once per skill, 10 questions, in the same way as the multiple-choice bank. The scoring prompt treats the transcript as words only, never instructions. Checked live: a prompt-injection attempt scored 0, and unsafe advice ("open the switch while it's live") scored 0.1. A late answer, or one after leaving the app, scores 0 without going to the AI. If the speech service is down, the clip is kept unscored ("listen to it"), counted as neutral, and the provider isn't failed for our outage.

### 3.3 Passing and review
- A pass needs both parts: 4 of 5 multiple choice **and** spoken answers averaging ≥ 0.5. The score counts each spoken answer as one more question worth its 0–1.
- **Every camera-test pass is held** (`HELD`), because the person comparing photos is the identity check. New strong signs: `NO_FACE_PHOTO`, and `FEW_PHOTOS` (under half the photos the time taken means — camera covered or off).
- In the portal the reviewer sees their ID beside the face at the start (and their profile photo), the photos taken during the test with times, and each spoken answer with its transcript, AI score and reason, and a player. Then Approve or Turn down, as in §2.9.

### 3.4 Where the files are
- **Private storage only** (`app/services/private_files.py`): the server's own disk under `PRIVATE_FILES_DIR`, a separate Docker volume (`private_files:/app/private`), never under `/static`, never a public URL.
- Each upload is checked by its first bytes (JPEG/PNG/WEBP, and M4A/AAC/MP3/WEBM/OGG/WAV), capped at 5 MB per photo and 8 MB per clip, with at most 60 photos per test. Keys can't step outside the store.
- Only an admin who can decide tests (`providers.verify`) can open a file (`GET /admin/skill-tests/evidence/{id}`, sent `no-store`), and every opening is logged. Admins with only `providers.view` see the attempt and its signs, but not the faces, ID, transcripts or recordings: the review detail replaces them with `{restricted: true}`.

### 3.5 How long they're kept
- **Daily** (`delete_expired_skill_test_files`), files are deleted **30 days after the decision** (`SKILL_EVIDENCE_KEEP_DAYS`), or after the test ended when there was nothing to decide.
- If turned down, they're kept **180 days** (`SKILL_EVIDENCE_KEEP_REJECTED_DAYS`), for an appeal.
- Never deleted while a review waits, a test is open, or the provider's account is on **legal hold** (it may be the evidence).
- The file and the transcript go; the score and the time stay with the attempt.
- **Removing an account** deletes them straight away (`AccountRepository.private_files_of`, cleared in `anonymise`).

### 3.6 Still to come
- A practical check (a photo or video task, such as a multimeter reading on an outlet).
- A badge that real work confirms or withdraws: bad ratings or disputes in a skill reopen its test.
- A live retest (a short video call with a Dodorez reviewer) after a test is turned down for cheating.
- Automated face matching (AWS Rekognition), only if the lawyer clears biometric consent: §16.

## 4. How each skill is checked, and tests in Spanish

Decided 2026-11-21: a written quiz is the wrong check for much manual work, it reads as an English-reading test, and every step before the first job loses providers. So the check now matches the skill's risk.

### 4.1 The three levels — `skills.check_level`
| Level | Before its jobs reach a provider |
|---|---|
| `NONE` — no test | A verified ID. Then the **first 3 rated jobs** in the skill decide: averaging under 3.5 on the customers' *skill* rating sets `provider_skills.test_required`, the skill locks, and the provider is told to pass the quick test. Passing it opens the skill again. |
| `TEST` — quick test | The 5-question test (§2). |
| `CAMERA` — camera test | §3. |

**Defaults (migration 118)** come from the catalogue's own ratings (`data/catalogue/`):
- `CAMERA` for the safety-critical licensed home trades: Plumber (gas lines included), Electrician, HVAC Technician, Roofer, Radon & Asbestos Testing, General Contractor, Kitchen and Bathroom Remodeler, Solar Installer, Pest Control, Home Inspector.
- `NONE` for the 157 professions the catalogue rates low-risk with no licence: cleaning, lawn and garden, moving help, assembly, errands, car detailing, tutoring and lessons, creative and online work.
- `TEST` for everything else. That includes licensed non-trade professions (barber, attorney, real-estate agent), whose licence is checked separately, and high-trust care (babysitter, caregiver), where the risk is trust (the background check), not knowledge.

### 4.2 What unlocks and locks a no-test skill (`app/services/skill_check_service.py`)
- **ID verified in the portal:** their no-test skills open. ID un-verified: the ones held only by ID (no test passed) close.
- **Adding a no-test skill** with a verified ID: open at once.
- **A customer's review** (`JobService.submit_review`): the weak-start check, once per provider and skill.
- **An admin changes a skill's level** (portal → Skill Tests → How skills are checked), applied straight away to everyone listing it:
  - **to `NONE`:** opens it for those with a verified ID (not those a weak start sent to the test);
  - **from `NONE` to a test:** closes it for those who never passed one, and they're told;
  - **between `TEST` and `CAMERA`:** nothing changes for anyone who has already passed.

  The portal asks before any change that opens or closes the skill for people.
- Eligibility answers `no_test` for these skills. The app's row shows "Verified · No test needed", or "Verify ID" until then. The dashboard prompt says "No test needed — verify your ID".

### 4.3 Tests in Spanish
- The rules screen has an **English / Español** switch. The choice is sent when the test starts (`lang`), fixed for that test, and a resumed test keeps it.
- Each bank is kept per language (`skill_questions.language`, `skill_spoken_questions.language`). The Spanish one is written by the model in US Spanish, in plain trade words, and grows the same way (§2.6).
- Spoken answers are transcribed in the test's language. They're scored with the same rubric, and the reason is written in English for the reviewer.
- Every word around the questions is translated (`components/skill-test/strings.ts`), from the rules to the consent screen to the result.
- Checked live: 40 Spanish multiple-choice and 10 Spanish spoken questions were generated, and a Spanish answer was scored sensibly (0.65).

## 5. Known limits
- Without a camera test, a second device next to the phone can't be seen. The 30 s clock and the bank's size are what limit it. With one, the photos may show it.
- `LEFT_APP` relies on an unmodified app reporting it. A modified app still can't beat the server's clock, the shuffled answers or the bank.
- On Android, the screen turning off counts as leaving the app. The rules say "stay in the app".
- Screenshot *detection* is iOS-only. Android blocks screenshots outright (FLAG_SECURE). Detecting one on Android 13 and older would need `READ_MEDIA_IMAGES`, a photo-library permission Google Play's policy restricts, so `app.json` blocks it (`blockedPermissions`).
- Apps older than this build show every answer as "Incorrect" (they expect `is_correct`). Testers need the new APK.
