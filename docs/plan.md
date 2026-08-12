# Plan

Opened 2026-08-12. This is the working plan; it is expected to change as the
spike in Phase 1 makes contact with reality.

## The requirement, stated precisely

An operator, mid-session with Claude, says something like:

> "Call Sam and let them know the electrician finished, the fuse box is done,
> and the power will be off for about ten minutes at four."

Claude must then:

1. Work out who "Sam" is and whether calling them is permitted.
2. Compose a **brief** — the specific, one-time content and boundaries of this
   call — and show it to the operator for confirmation.
3. Place a real phone call to Sam.
4. Speak as an identified AI assistant acting on Alex's behalf, deliver the
   message, handle a small amount of back-and-forth, and end the call.
5. Report back into the session what actually happened, including anything Sam
   said in reply.

Two properties make this different from ordinary voice-agent work:

- **The prompt is unique per call and disposable.** There is no assistant
  persona to tune over time. Every call has a brief that will never be reused.
- **The volume is one.** Nothing here is a throughput problem. Latency and
  naturalness matter; cost per minute is irrelevant at this scale.

## The three tiers

The single most useful thing in this plan: the requirement is not one feature,
it is three, and they have wildly different costs.

### Tier 0 — announcement (no voice agent)

One-way. Claude calls, speaks a message, optionally offers "press 1 if you got
that", hangs up. Implemented entirely as TwiML `<Say>` returned to a
`CreateCall`, using tooling that already exists in `Twilio-Manager` and the
`twilio-ops` plugin.

No LLM in the call path at all — Claude writes the words *before* dialling, and
the call itself is a dumb pipe. Which means it is cheap, predictable, and
impossible for it to say something unplanned.

**This covers a real fraction of the actual use case** ("tell them X") and is
buildable in an afternoon. It should ship first and independently, and it is
the fallback whenever the conversational tier fails.

### Tier 1 — conversational, bounded

The recipient can talk back. The agent answers questions that the brief
anticipated, deflects the ones it did not, and makes no commitments. This is
the tier the framework survey is about, and the main body of work.

### Tier 2 — errand with a return value

The call has a goal beyond delivery: confirm a time, get a yes/no, find out
whether Robin needs a lift on Thursday. The agent must come back with a
*structured* answer, not just a transcript. Adds tool-calling and an outcome
schema on top of tier 1.

This is past what the word *notifier* promises, and that is intentional. Tier 2
is where the thing becomes genuinely useful and also where it becomes capable
of causing real trouble, because a misheard answer propagates silently into
decisions the operator then makes. Not in the first version, and it should be
renamed if it ever becomes the main event.

## Decisions taken

| # | Decision | Why |
|---|---|---|
| D1 | Twilio for transport | Already in use, already version-controlled, ops plugin exists, both lines are known-good |
| D2 | Twilio ConversationRelay for tier 1, not a hosted voice-agent platform | Per-call prompts are the whole requirement; ConversationRelay gives unlimited per-call prompt freedom because the prompt is built by our own code. See [survey](framework-survey.md) |
| D3 | Claude as the in-call LLM | Makes the persona honest; official Twilio + Anthropic integration path exists; tool-calling documented |
| D4 | Dedicated caller identity, not the house line | Recipients must be able to recognise a Claude call before answering. See [architecture](architecture.md#the-line) |
| D5 | No SIP credential in the first design | Outbound Twilio calls don't need one. Revisit only if D2 is overturned in favour of a SIP-native platform |
| D6 | Confirm-before-dial is mandatory, not a setting | An outbound call to a third party is irreversible and outward-facing |
| D7 | Recipient allowlist, seeded manually | Claude must not be able to dial an arbitrary number it inferred from context |
| D8 | Ship tier 0 first and keep it as the permanent fallback | Highest value per unit of work in the whole plan |
| D9 | Vapi considered as a middleware layer and not chosen | Two reasons: the operator reports mixed results with it in prior use, and its model is campaign-shaped — a saved assistant plus variable fills. Its transient-assistant mode *does* meet the per-call-prompt requirement, so it stays the first fallback if ConversationRelay disappoints in Phase 2 |

## Phases

### Phase 0 — plan *(this)*

Capture the requirement, survey the platforms, settle the architecture and the
etiquette rules. Done when this repo reads correctly to someone who was not in
the conversation.

### Phase 1 — tier 0, shipped

A script that takes a message string and a recipient key, and places an
announcement call. Reuses the existing Twilio account and serverless setup.

Done when: Claude can be asked to call a nominated number and speak a sentence,
and it works reliably, including to voicemail.

### Phase 2 — tier 1 spike

A WebSocket server bridging ConversationRelay to Claude, with the system prompt
built from custom parameters carried in the TwiML. Deliberately crude: one
hardcoded brief, calls to the operator's own mobile only, no allowlist, no
persistence. The purpose is to find out what is actually hard — latency,
interruption handling, how a synthesised voice sounds delivering a household
message, whether Hebrew works.

Done when: the operator can hold a fifteen-second conversation with it and
report whether it feels acceptable to point at another person.

### Phase 3 — brief schema and the guardrails

Formalise the brief (see [`examples/briefs/`](../examples/briefs)), the
allowlist, the confirm step, the time-of-day window, the turn and duration
caps, and the voicemail path. This is the phase that makes it safe to point at
family rather than at yourself.

### Phase 4 — package as a Claude Code plugin

Skills, roughly:

- `place-a-call` — the main entry point; resolve recipient, compose brief,
  confirm, dial.
- `write-a-call-brief` — compose and critique a brief without dialling, for
  when the operator wants to see it first or reuse it.
- `call-outcome` — pull the transcript and outcome back into the session.
- `manage-call-recipients` — the allowlist.

Plus a `reference/` for the ConversationRelay message shapes and the Twilio
call parameters, because those are the things that will otherwise be
rediscovered every time.

### Phase 5 — tier 2, if it has earned it

Structured outcomes and tool use. Gated on tier 1 being something the household
does not find irritating.

## Open questions

These need answers from the operator, and two of them can change the design.

1. **Hebrew.** Do these calls need to be in Hebrew, or at least
   Hebrew-capable? This is a first-order question, not a polish item — it
   constrains TTS voice selection, STT accuracy, and possibly the platform
   choice itself. A household message delivered in stilted Hebrew is worse than
   no call. *Blocks: platform confirmation in Phase 2.*
2. **One-way or two-way, honestly?** If most real errands are "tell them X",
   tier 0 may be the entire product and phases 2–5 are optional. Worth
   answering before building a WebSocket server. *Blocks: whether Phase 2
   happens at all.*
3. **Which number does it call from?** A new Twilio number, or one of the two
   existing lines? A new number costs about a dollar a month and buys a clean,
   recognisable identity. *Blocks: Phase 1.*
4. **Who is on the allowlist at launch?** Suggest: the operator's own mobile
   only, for as long as it takes to stop being embarrassing.
5. **Recording.** Transcript-only, or audio too? Transcript is enough for the
   outcome loop and is much easier to justify to the person on the other end.

## Prior art and what it tells us

The near-neighbour repos (`Twilio-Manager`, `Agent-IVR`, `Twilio-Ops-Plugin`)
are all *inbound*. The outbound-agent-calls-a-human case is well served
commercially — it is the entire business of Vapi, Retell, Bland and ElevenLabs
Agents — but always in the shape of *one campaign, many recipients*. The
literature on *one recipient, one bespoke instruction, no campaign* is thin,
which is exactly why the per-call-prompt mechanism is the thing the survey
measures.

Nothing found so far does the specific thing this repo is about: a general
assistant, mid-task, deciding that the right way to discharge an instruction is
to telephone someone. If that turns out to exist, it belongs in this section.
