# Call recording on our own media server — design

**Status (2026-10-16):**
- **Stages 1–3 are done and live:** infrastructure, server and admin panel.
- **Stage 4 code is done in both apps, but not yet built or tested on devices.**
  - Built: the call audio sits behind `MediaTransport` / `CallMedia` (`services/call/`), and `chat.tsx` no longer touches WebRTC.
  - Not built yet: the ringing and signalling state machine still lives in `chat.tsx`. Moving it into its own call engine is the next refactor.
- **Recording is still off** (`CALL_RECORDING_ENABLED=false`) until the consent wording is approved.
**Decisions (founder, 2026-10-16):** self-hosted only, no third-party call service.
Purpose is quality, safety and disputes, not AI training. Recordings are kept
90 days, or longer while the job has an open dispute or support ticket. A
consent notice is shown on every call; the wording goes to a lawyer before
recording is switched on for real users.

## Why the call path changes

Today the two phones send audio directly to each other (WebRTC, peer to peer),
encrypted end to end. Even a TURN relay only forwards encrypted packets. Nothing
we run can hear a call, so nothing can record one. To record, audio has to go
through a server we own: a **media server (SFU)** that both phones connect to.
It forwards each person's audio to the other, and our recorder joins the room
and writes one mixed audio file.

## Components and what each one owns (single responsibility)

| Component | Where | Owns | Knows nothing about |
|---|---|---|---|
| **LiveKit server** (open source, self-hosted) | call server `dodorez-media` (EC2 t3.large, Elastic IP 43.205.75.86) | Moving audio between the two phones; reconnection; NAT relay (built-in TURN) | Jobs, users, permissions |
| **Recorder** (LiveKit Egress, self-hosted) | same machine | Writing a room's mixed audio to `recordings/calls/<job>/<call>.ogg` | Who may listen |
| **Caddy** on the call server | same machine | HTTPS; only lets our main server fetch a recording (shared key) | Everything else |
| **Call signalling** (existing) | main API, `call_signaling.py` + `call_registry.py` | Ringing, busy, answer, end, call log, push | Audio and recording |
| **Media client** (new) | main API, `app/services/media_server.py` | Access tokens for a room; start/stop the recorder; fetching a file | Calls, jobs, policy |
| **Call recordings** (new) | main API: `CallRecordingService`, `CallRecordingRepository`, table `call_recordings` | A recording's lifecycle and who may hear it | How audio moves |
| **Retention policy** (new, pure) | `app/domain/recording_retention.py` | Whether a recording must be kept or may be deleted | Storage, I/O |
| **Call engine** (new, apps) | `services/call/` in each app | One call's lifecycle as a state machine; ringing UI hooks | Screens |
| **Media transport** (new interface, apps) | `services/call/mediaTransport.ts` | Connecting audio for an answered call; mute; speaker; reconnect events | Ringing, jobs |
| **LiveKit transport** (new, apps) | `services/call/livekitTransport.ts` | The transport, using our media server | Everything above it |

The call engine depends on the `MediaTransport` interface, not on LiveKit
(dependency inversion), so the screens never import WebRTC or LiveKit, and the
transport can be swapped or faked in tests.

## Lifecycles

### A call (call engine, apps)

```
idle ──dial──▶ calling ──callee rings──▶ ringing_remote
  ▲               │                          │
  │            no answer / cancel            │ answered
  │               ▼                          ▼
  └────────── ended ◀── hang up / drop ── connecting ──media up──▶ connected
                                             ▲                       │
                                             └──── reconnecting ◀────┘ (network blip)
incoming:  idle ──invite──▶ ringing ──accept──▶ connecting ──▶ connected
                              └── decline / timeout ──▶ ended
```

* Ringing, accept, decline and end stay on our job WebSocket and push (unchanged).
* **Only after answer** does either phone connect media: it asks our API for
  a room token for that call, then joins room `call-<call_id>`.
* `reconnecting` is the transport's own reconnection. If it can't recover
  within the grace period, the engine ends the call and sends `call_end`.
* `ended` always runs one teardown: transport disconnect, ringtones off,
  notifications cleared, call screen released. It's idempotent.

### A recording (server)

```
           call answered                   call ended
(none) ──────────────────▶ recording ──────────────────▶ processing ──file confirmed──▶ ready
                              │                              │                           │
                       recorder failed                  no file after retries      retention says
                              ▼                              ▼                     "may delete"
                           failed                         failed                       ▼
                                                                                    deleted
```

* **Start:** on `call_answer` accepted by the registry. The server creates the
  row (`recording`) and asks the recorder to record room `call-<call_id>`.
  Failure to start never affects the call; the row becomes `failed`.
* **Stop:** on `call_end` (any side, or the server's timers). The recorder is
  told to stop; the row becomes `processing` with the call's duration.
* **Finalise:** a background task asks the recorder for the egress result
  until it's complete, then stores the file path and size → `ready`.
* **Retention (daily):** `ready` rows older than 90 days are deleted unless the
  job has an open dispute or an open support ticket (pure policy function).
  The call server deletes the file; the row becomes `deleted` (kept for audit).

## Access

* Phones never get recordings. They get a **room token** only for a call they
  are the caller or callee of, valid only for that room, for a short time.
* Admins listen through the main API: `GET /admin/recordings/{id}/audio`,
  permission `recordings.listen`. Every play is written to
  `recording_access_log` (who, which recording, when).
* The call server serves files only to requests with the shared key; the key
  lives in both servers' `.env`, never in the apps or the admin panel.

## Consent

Every call screen shows "This call is recorded for quality and safety" before
it connects. A spoken notice isn't built yet (a question for the lawyer).
Recording is gated by a server setting, `CALL_RECORDING_ENABLED`, off until the
notice and privacy policy wording are approved.

## Rollout

1. **Infrastructure** — call server, recorder, private storage. Done; proven
   with a recorded test call.
2. **Server** — media client, room tokens, recording lifecycle, retention,
   admin endpoints, permission, audit log; tests for every transition.
3. **Admin panel** — recordings with a player on the job, dispute and ticket pages.
4. **Apps** — call engine + transport interface + LiveKit transport, replacing
   peer-to-peer audio (`react-native-webrtc` → `@livekit/react-native-webrtc`;
   the two can't coexist). Needs a prebuild and device testing on Android,
   iPhone and the Samsung. Both apps must update together: an old app can't
   join a media-server call.

## Built (stage 4, apps)

- **`services/call/mediaTransport.ts`** is the interface. **`livekitTransport.ts`** is the only file that knows LiveKit. **`callMedia.ts`** coordinates the call's pass, the transport and the phone's audio session.
  - The pass is fetched while the phone is ringing, so the recording notice shows before anyone answers.
  - A generation counter stops a connect that's still under way when the call ends, so no orphaned room is left with the mic and recorder on.
- **`chat.tsx` (both apps):**
  - The offer carries `media: "sfu"` and no SDP. Both phones join the room only after the call is answered.
  - The transport's `reconnecting`, `reconnected` and `disconnected` events replace the old ICE-restart logic, with the same 30 s grace period.
  - Mute and speaker go through `CallMedia`. InCallManager now only handles ringing.
- **Echo:** LiveKit's Android audio module would turn the hardware echo canceller back on. `patches/@livekit+react-native+2.12.0.patch` keeps WebRTC's software canceller instead. It replaces `plugins/with-webrtc-audio.js`, which LiveKit's setup would have overwritten anyway.
- **iOS:** CallKit audio activation uses `@livekit/react-native-webrtc`'s `RTCAudioSession` (same API). This needs testing on an iPhone: LiveKit's `AudioSession` and CallKit both manage the audio session.
- **Packages (exact versions):** `@livekit/react-native` 2.12.0, `@livekit/react-native-webrtc` 144.1.2 (replaces `react-native-webrtc`), `livekit-client` 2.22.3, `@livekit/react-native-expo-plugin` 1.0.2. **A prebuild is needed.** An old app version can't join a media-server call, so both apps must update together.

## Failure handling

| Failure | Effect |
|---|---|
| Media server down | Calls can't connect audio; the app shows "Call failed" and ends the call (no silent call). Health check alarms. |
| Recorder fails to start | Call continues unrecorded; row `failed`, logged. |
| Recorder never finishes | Finalise task gives up after its retries; row `failed`. |
| Main API restarts mid-call | The room and recording carry on; `call_end` from either phone still stops it; the room also closes itself 60 s after both leave. |
| Call server disk fills | Retention deletes expired files daily. A disk usage alarm isn't set up yet (follow-up). |
