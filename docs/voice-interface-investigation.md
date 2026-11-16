# Hands-free voice ("Jarvis") for Claude Code: investigation

Date: 2026-11-16. Vocci facts were read through the Apify web-fetch connector (a few metered calls on your Apify account), because this sandbox blocks vocci.ai. Written from a Claude Code **cloud** session (started from the Fold). This session can't open
files on the MacBook. Everything below about the Mac comes from the Kosmos board, the Appolis connector, and
the Remote Control session list. Each claim is marked **VERIFIED** (seen in this session), **DOCUMENTED** (in
the Claude Code docs, not tested here) or **UNVERIFIED**.

---

## 1. What you already have (inspected)

### The plain-English explanation system: keep it as is
| Piece | Where it lives | What it does | Status |
|---|---|---|---|
| Law 3 "PLAIN TERMS AT THE END" | Appolis connector rulebook (served to every session) | Every answer ends with a plain-English summary that names anything unverified | VERIFIED |
| `in-plain-terms` skill (v2026-09-16.87113e4706f7) | Appolis house skill, installed into `~/.claude/skills/` | Fixes the shape: the last section is labelled exactly `**In plain terms:**` and covers what changed, what it means, what's left, and what wasn't verified | VERIFIED |
| `speak-summary` skill (v2026-10-31) | Tyler's skill. **Windows-only** (PowerShell) | A Stop hook finds the `In plain terms` section in the transcript, cleans it for speech, and reads it aloud (Edge voice or ElevenLabs). It never reads the whole reply | VERIFIED. You chose to remove it from the Mac (note_8748, Q1 = yes, Oct 31) |
| Kosmos Agora chat + decision forms | Kosmos board | "The voice": sessions post progress and questions there. A choice goes on a tickable decision form | VERIFIED |
| STATE notes | e.g. `note_8131` "Hockey Lab — CURRENT STATE" | A short current-status note that each session rewrites | VERIFIED |

**The design rule that follows:** the voice layer reads out the `In plain terms:` section, just as
`speak-summary` does. It never writes its own summaries. That label is the contract, so the voice can rely on
it.

Live proof, VERIFIED today: I read the newest reply of your Mac session **"HockeyLab Player Card Design"**
(`session_013yPfwNgB8XchbaSimnd5f1`) from this cloud session. It ends:

> **In plain terms:** Status: Hockey Lab now saves the win chance for your goalies every half hour before their
> games… The Mac still needs to be awake and plugged in for this to run… Not done: not tested on a real game
> night yet, nothing committed to git.

### Hardware and software
| Have | Relevant because | Status |
|---|---|---|
| MacBook Pro (`ravens-macbook-pro`), Claude desktop app, Claude Code CLI 2.1.321 in the Hockey session | Runs the sessions; 2.1.321 is new enough for channels, permission relay and cross-session messaging | VERIFIED (version from session metadata) |
| Desktop sessions bridged to Remote Control (tag `remote-control-sdk`, ~30 sessions listed) | They can be listed, read, and (by the docs) messaged from a Claude cloud session | VERIFIED list/read; send not yet tested |
| Tailscale on the Mac, Funnel → port 8749 (Hockey Lab shared door) | A ready, private, encrypted path from the phone to the Mac | VERIFIED on board. Phone side was set up for an **iPhone** (todo_8115). The Fold needs the Tailscale Android app |
| launchd jobs, Node v26, Keychain-stored secrets, Resend email alerts | A Mac "bridge" service can run the same way the Hockey server does | VERIFIED on board |
| Local Ollama `qwen3:30b` (Ask Hockey Lab) | Free local model, but slow (35–48 s per answer). Too slow for live voice | VERIFIED on board |
| ElevenLabs account (connected here) | Voice agents (speech in/out, turn-taking, tools/webhooks, phone numbers) | VERIFIED connected; **0 agents, 0 phone numbers** today |
| Wispr Flow (connected here) | Dictation; not a two-way voice agent | VERIFIED connected |
| Galaxy Fold + Bluetooth headphones | Mic/speaker. Headset buttons can answer/launch calls and the assistant | Standard Android behaviour; not tested with your headphones |

---

## 2. Can something send to and read from an *existing* desktop session?

| Method | Send in | Read reply | Keeps the session's context | Works on desktop-app sessions | Verdict |
|---|---|---|---|---|---|
| **Remote Control** (Claude app on the Fold) | yes | yes | yes | yes (`/rc` in the Code tab) | DOCUMENTED + in use. It's the thing you want to avoid holding open |
| **Claude cloud session → `send_message` / `list_events`** (the tools this session has) | yes (tool exists; session reports `cross_session_inbound: available`) | **VERIFIED** (read the Hockey reply above) | yes: it's the same session | yes: those are your desktop sessions | **Best supported route.** Send not yet tested; needs your OK |
| Cross-session messaging (local Unix socket on the Mac) | yes, from another process on the Mac | no reply API; read it via a Stop hook | yes | DOCUMENTED for macOS v2.1.224+ | Supported, but the message format isn't fully documented. Mac-only |
| `claude -p --resume <id>` from Terminal | yes | yes (JSON) | yes | it's the same transcript | **Not safe while the desktop app has the session open**: the docs warn two writers can lose messages. Use `--fork-session` (makes a copy) |
| **Channels** (custom MCP server, `--channels`) | yes (push) | yes (reply tool) | yes | **Docs only show terminal launch.** Desktop Code tab not documented | Research preview. Custom channels need `--dangerously-load-development-channels`. Would mean running *new* terminal sessions, not your existing desktop ones |
| Hooks (Stop / Notification / PermissionRequest) | no | **yes**: a Stop hook can capture `In plain terms` (the speak-summary pattern) | n/a | Hook events fire in the desktop app (DOCUMENTED). Settings-file hooks in desktop: UNVERIFIED | The building block for "tell me when it's done" |
| Dispatch (Claude app → Cowork → Code session on the Mac) | yes (new task) | push notification | starts **new** sessions | yes | Good for "task for later"; not for an existing session |

**Bottom line:** controlling an existing session is possible with supported tools. The path goes through
Anthropic's Remote Control bridge: a Claude cloud session uses `send_message` / `list_events`. Read is verified.
Send is the next test.

---

## 3. How the Fold and headphones would work (voice layer)

Nothing from Anthropic gives a hands-free voice line into a Code session. The Claude app's voice mode is for
chat, not Code. So the voice layer is the custom part.

**Recommended: an ElevenLabs voice agent ("Jarvis") with a few tools that call a small "router".**

```
 headphones ─► Fold ─► ElevenLabs agent ("Jarvis")      ← speech in/out, interruptions, turn-taking
   (call or app)            │  tools: list_projects, get_status, ask_session, send_instruction, answer_pending
                            ▼
                     Router  (custom, small)
             ┌──────────────┴──────────────┐
   Option A: Claude cloud "hub" session      Option B: Mac bridge over Tailscale
   uses send_message / list_events            uses cross-session socket + Stop hook
   (supported; works with Mac asleep?  NO —   (supported pieces; Mac-only;
    target session must be connected)          needs Tailscale Funnel + auth)
                            ▼
            Your existing Mac session (e.g. HockeyLab)
            replies ending "In plain terms: …"  ─► router extracts that section ─► Jarvis speaks it
```

- **How you reach Jarvis hands-free:** an ElevenLabs agent can have a **phone number**. "Hey Google, call
  Jarvis" or one headset-button redial, and you're in a normal call through your Bluetooth headphones. You
  don't need to keep an app open. A phone number has a monthly cost (Twilio/SIP), and the voice minutes are
  billed by ElevenLabs.
- **Fast answers:** "check progress" doesn't need to wake the session. The router reads the session's last
  `In plain terms` section and the project's Kosmos STATE note, and Jarvis answers in seconds.
- **Slow answers:** a real question to the session can take 30 s to many minutes. Jarvis says "I've asked the
  Hockey session; I'll call or ping you when it answers," or holds the line for short ones.
- **Privacy:** whatever Jarvis reads aloud goes through ElevenLabs. Keep secrets out of the summaries (law 3 and
  the speak-summary rules already forbid reading keys or logs).

---

## 4. Yes/no decisions and permission prompts

There are three different kinds of "asking". They behave differently:

1. **A plain question in the reply** ("Want me to make that change?"). A voice "yes, go ahead" is just the next
   message → works through any send route. **This should be the default for remote work.**
2. **Kosmos decision forms.** Your current method. The form records *who* ticked a box (`kind: person` vs `ai`).
   A box ticked by Jarvis would show as AI. Don't let the voice tick forms on your behalf unless you decide that
   counts. Better: Jarvis reads the form aloud and posts your spoken answer into the Agora chat as a quote.
3. **Tool permission prompts** ("Allow Bash: …?"). These **cannot** be answered by a cross-session message (the
   docs say plainly that messages "can't approve anything"). Supported ways to answer them remotely:
   - Remote Control app (the Claude app on the Fold), with phone push turned on under `/config` → "Push when
     actions required". DOCUMENTED.
   - Channel permission relay (terminal sessions launched with `--channels`, v2.1.234+). DOCUMENTED.
   - A `PermissionRequest` hook (command or HTTP) that asks your phone and waits. Hooks wait up to 600 s by
     default, but the docs don't state outright that the session blocks until you answer. UNVERIFIED; test first.
   - Recommendation: keep away-from-home sessions on a permission mode or allowlist that covers routine safe
     tools, so they rarely hit prompts. When one does, Jarvis says "the Hockey session is waiting for a
     permission on your phone" and you tap once in the Claude app.

---

## 5. Recorded "task for later" (Vocci webhook) vs live conversation

See §7 for what the ring can and can't do. The routing design doesn't depend on the ring:

- **Recorded task:** a `session.completed` webhook (transcript + summary) arrives at the router. The router finds
  "Claude task: <project> — <instruction>", matches the project name to a session by title (and the Kosmos
  project), and sends the instruction. If there's no confident match, it files a Kosmos to-do and asks you once.
  When the session's next `In plain terms` arrives, it pings you (Remote Control push, email via Resend, or a
  Jarvis call-back).
- **Difference from live:** one-way, after the fact, with no back-and-forth. Delay = recording finished +
  transcription + summary + session time. It suits "do this later", not "what still needs doing?".

---

## 6. Failure cases

| Situation | What happens | Mitigation |
|---|---|---|
| Mac asleep | Bridge sessions drop. Nothing runs. Remote Control reconnects automatically on wake (DOCUMENTED) | Plugged in + "prevent sleep on power adapter" or `caffeinate`. Your board already notes "Mac asleep = link down" |
| Desktop app quit / session closed | The session can't receive. What happens to a message sent to a disconnected session is UNVERIFIED | Router checks `connection_status` first and tells you "Hockey session is offline". It can queue the task as a Kosmos to-do |
| Network drop | Remote Control retries until back (server mode gives up after ~10 min) | Router retries; Jarvis says so plainly |
| Session needs input while you're away | Plain question → Jarvis reads it. Tool prompt → phone push | §4 |
| Session context near full | The Hockey session is at 330k of 400k tokens | Your session-handoff skill already exists. The router should prefer the newest session per project |
| Two writers | `-p --resume` on a session the app has open can lose messages | Don't use it on live desktop sessions |

---

## 7. Vocci ring: verified from vendor docs (docs.vocci.ai, support.vocci.ai, vocci.ai)

| Question | Answer | Source |
|---|---|---|
| Double-tap? | Yes, but it's fixed: **double-click = start/stop recording**. Single click = highlight. Long-press = "Voice Demand" to Vocci's own AI (max 60 s) | support article 15754703436047 |
| Remap the button / expose it to your app? | **Not documented anywhere.** No SDK, MCP or webhook for button events | docs + support |
| Live mic for a two-way conversation? | **No.** No headset/HFP/LE-Audio mode. It records to the ring, then syncs over BLE to the Vocci app only | docs, specs |
| Speaker / feedback | No speaker. Haptics (1 pulse start, 2 stop, 3 error) and an LED. Haptics can't be controlled from outside | support |
| MCP for Claude Code | Yes: `claude mcp add --transport http vocci https://mcp.vocci.ai/mcp` (OAuth). Read-only search/fetch of transcripts, summaries, audio links, plus to-do create/complete. **Nothing real-time** | docs.vocci.ai/developers/mcp |
| Webhook | `session.completed` (transcript + summary Markdown, ≤1 MB, HMAC-signed) and `audio.uploaded`. Public HTTPS, answer in 5 s, retries for 24 h. **Ring voice-command (long-press) recordings don't send `audio.uploaded`**. Unclear whether they send `session.completed` | docs.vocci.ai/developers/webhooks |
| Latency | Not published. It's after phone sync + upload + transcription (~1 min per 15 min of audio). Auto-upload and auto-transcription are **off by default** | support |
| Android / Fold | Android app on Google Play (v1.1.39). **Fold not stated.** One phone at a time | Play, support |
| Price | $249 sale / $299 list. Free plan: 300 transcription min/month. MCP + webhooks on every plan | product page |

**Verdict:** fine for the **"task for later"** path (double-tap → speak "Claude task: Hockey Lab — …" →
double-tap → webhook). **Not usable for the live conversation.** For live, the trigger is a separate
programmable button and the mic/speaker are your headphones:
- **Flic 2** button: click/double-click/hold, an official Android SDK, and it can launch apps or a call. Strongest
  option.
- A Bluetooth media-key ring plus the Key Mapper app (only if the ring sends real key codes; reportedly stops
  with the screen off).
- Your headphones' own call/assistant button (redial "Jarvis").
- Galaxy Ring double-pinch: **not** remappable (photo/alarm only).

---

## 8. Proposed first test (one session, no ring, no money spent)

Target: **"HockeyLab Player Card Design"** (connected, idle).

1. **Send test (needs your OK):** from this cloud session, `send_message` a read-only request: "Don't change
   anything. In two or three sentences, what's still left to do on Hockey Lab? End with In plain terms." Then
   `list_events` and pull out the `In plain terms` section. **Proves:** an existing desktop session can be
   driven remotely with its context kept, and nothing on the Mac has to change.
2. **Voice test:** I create one private ElevenLabs test agent ("Jarvis test") with your OK and a voice you
   pick. It's only a test link (`agents_get_link`) that you open on the Fold with headphones. No phone number.
   It reads the plain-terms reply from step 1 aloud. **Proves:** speech in your headphones from the real
   summary. Spends a few ElevenLabs credits.
3. **Then decide** between Option A (cloud hub) and Option B (Mac bridge over Tailscale) for the always-on
   router. That's the first real custom build.

What is custom development: the router, the ElevenLabs agent tools, the project → session name matching, the
Vocci webhook receiver, and a Mac Stop hook that captures `In plain terms` (a port of speak-summary's capture,
without the Windows parts).
