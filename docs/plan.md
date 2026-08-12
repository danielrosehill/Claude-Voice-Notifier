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
5. Report back into the session what actually happened — whether it connected,
   whether a human or a machine answered, and the transcript.

Step 5 is for the operator's oversight, not for message-passing, and the
distinction matters enough to have its own section below.

Two properties make this different from ordinary voice-agent work:

- **The prompt is unique per call and disposable.** There is no assistant
  persona to tune over time. Every call has a brief that will never be reused.
- **The volume is one.** Nothing here is a throughput problem. Latency and
  naturalness matter; cost per minute is irrelevant at this scale.

## The tiers

The single most useful thing in this plan: the requirement is not one feature,
and the pieces have wildly different costs. Two are in scope; the third is
named so that it stays rejected.

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

The recipient can talk back, and the agent can respond. It answers questions
the brief anticipated, deflects the ones it did not, makes no commitments, and
**takes no messages**. This is the tier the framework survey is about, and the
main body of work.

"Conversational" here means the recipient does not have to listen in silence to
a recording — not that a dialogue is being opened. The agent's side of the
conversation is bounded by the brief in advance.

### Tier 2 — errand with a return value — **rejected, 2026-08-12**

The call would have a goal beyond delivery: confirm a time, get a yes/no, find
out whether Robin needs a lift on Thursday. The agent would come back with a
*structured* answer.

**This is out of scope, and not on a roadmap.** The product is a notifier — a
pager, conceptually — and a pager does not take messages. The agent has **no
return channel**: it cannot accept a reply for the operator, cannot promise to
pass anything on, and must decline when asked to.

The reasoning is worth keeping, because tier 2 is the thing that will get
proposed again:

- The value is real but the failure is silent. A misheard or subtly reframed
  reply arrives in the operator's session carrying the authority of a direct
  quote, and gets acted on. Nobody involved has any signal that it went wrong —
  the recipient believes the message was delivered, the operator believes they
  heard it, and only the consequences disagree.
- Every other risk in this design is *loud*. A bad notification is embarrassing
  immediately and gets corrected. A bad return value is quiet and compounds.
- There is a working alternative that costs the recipient one action: contact
  the operator directly. The relay saves a small amount of friction and buys a
  large amount of ambiguity about who said what.

If a return channel is ever wanted, it should be built as a different product
with a different name and its own consent conversation, not grown quietly out
of this one.

## The transcript is not a return channel, and the agent must not pretend otherwise

There is a tension in D10 worth naming, because the naive version of it is a
lie the agent would tell.

The operator receives a transcript of the call — they must, it is their call and
it is how a bad notification gets caught. So if Robin says "tell Alex Thursday
doesn't work" and the agent declines to take the message, Robin's words *still
reach Alex*, in the transcript. An agent that says "nothing you say to me goes
to Alex" would be stating something false.

The resolution is in what the refusal actually claims. It does not claim
Robin's words vanish. It claims the agent **will not carry them as a message**,
because it cannot guarantee they arrive, arrive intact, or arrive in time — and
so Robin should not rely on it and should contact Alex directly. That is true,
and it is the honest reason not to use a notifier as a mailbox.

Two rules follow, and they are the real content of D10:

1. **The refusal is about reliability, not secrecy.** Fixed wording, in the
   conduct layer. It never promises delivery and never claims non-delivery.
2. **On the operator's side, a transcript is oversight material, not
   instruction.** When Claude reports the call, anything the recipient said is
   reported as an observation — "Robin mentioned Thursday doesn't work" — and
   Claude must not act on it, schedule against it, or treat it as confirmed.
   Acting on it requires the operator to go and check with Robin directly.

Rule 2 is the one that will erode if it isn't written down, because acting on
it is exactly what a helpful assistant wants to do next.

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
| D10 | **No return channel.** The agent never accepts a message for the operator, and refuses with a fixed template when asked | The failure mode of a relayed reply is silent — see [tier 2](#tier-2--errand-with-a-return-value--rejected-2026-08-12). A fixed template rather than a model-composed refusal, because a helpful model will otherwise soften it into a promise |
| D11 | English only for now | Confirmed 2026-08-12. Removes the one open question that could have overturned D2 on voice quality. Revisit if the recipient list changes |

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
interruption handling, and how a synthesised voice sounds delivering a
household message.

Test the refusal path explicitly in this phase, not later: ask the agent to
pass a message back and check it declines cleanly rather than obliging. It is
the single behaviour most likely to be quietly wrong, because declining runs
against the model's grain.

Done when: the operator can hold a fifteen-second conversation with it, get a
clean refusal when asking it to relay something, and report whether it feels
acceptable to point at another person.

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

Phase 4 is the last phase. There is no phase 5 — tier 2 is rejected, not
scheduled.

## Settled

Both of these were open when the plan was first written and both were answered
on 2026-08-12. Recorded here because "we already decided this" is cheaper than
deciding it again.

- **Hebrew: not required for now.** English only. This was the one question
  capable of overturning the platform choice, since a household message in
  stilted Hebrew is worse than no call and voice quality would have outranked
  every other criterion. It doesn't apply, so D2 stands unchallenged. Revisit
  only if the recipient list changes.
- **One-way, not two-way.** The model is a pager. The agent may converse, but
  there is no return channel — see [D10](#decisions-taken) and the
  [transcript section](#the-transcript-is-not-a-return-channel-and-the-agent-must-not-pretend-otherwise).

## Open questions

Three left, none of them blocking design work.

1. **Which number does it call from?** A new Twilio number, or one of the two
   existing lines? A new number costs about a dollar a month and buys a clean,
   recognisable identity. *Blocks: Phase 1.*
2. **Who is on the allowlist at launch?** Suggest: the operator's own mobile
   only, for as long as it takes to stop being embarrassing.
3. **Recording.** Transcript-only, or audio too? Transcript is enough and is
   much easier to justify to the person on the other end.

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
