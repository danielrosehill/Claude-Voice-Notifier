# Architecture

Two layers that can be chosen independently: **the line** (how a call gets
originated and what identity it carries) and **the brain** (what decides what
is said). Getting them confused is the main source of muddle in this problem —
"a SIP credential for Claude" is a line-layer answer to a brain-layer question.

---

## The line

### Outbound Twilio calls do not need SIP

Worth stating flatly, because the instinct to reach for SIP is strong and here
it is wrong. To originate a call, you `POST` to the Twilio Calls resource with
`To`, `From`, and either a `Url` pointing at TwiML or the TwiML inline. Twilio
dials, and when the far end answers it executes the TwiML. That is the whole
mechanism. No SIP registration, no credential list, no trunk.

The existing gateway already exposes this as
`twilio-residence__TwilioApiV2010--CreateCall`, so the line layer for tier 0 is
arguably already built.

SIP becomes relevant in exactly one case: if the brain layer is a **SIP-native
voice platform** — OpenAI's Realtime SIP connector, an ElevenLabs SIP trunk,
LiveKit SIP — then Twilio Elastic SIP Trunking bridges the PSTN call into it,
and *that* is where a credential lives. It is a consequence of a brain-layer
decision, not a prerequisite.

### What the instinct was actually after: a caller identity

The useful part of "there should be a SIP credential for Claude" is the word
*for Claude*. The recipient's phone should be able to tell them, before they
answer, that this is the assistant and not the person.

That is solved with a **dedicated Twilio number**, not a credential:

- The household saves it once as *"Claude (Alex's assistant)"*. From then on
  every notification call arrives pre-identified, on the lock screen, without
  anyone having to listen to a disclosure first.
- A call from the house line is ambiguous in the worst possible way — Sam picks
  up expecting Alex and gets a synthesised voice. A distinct number removes
  that entirely.
- It gives a clean kill switch. Releasing one number stops all of it, without
  touching the lines that carry real calls.
- It keeps the logs, recordings and billing for machine-placed calls separate
  from the human lines already managed in `Twilio-Manager`.

Cost is roughly a dollar a month. This is the cheapest safety feature in the
whole design.

### Reaching a person, or not

`CreateCall` supports answering-machine detection via `MachineDetection`, which
reports whether a human or a machine answered before the TwiML runs. This
matters more here than in most applications: a notification delivered to a
voicemail box is fine and often preferable, but a *conversational* agent
talking to a voicemail greeting is a comedy. The design should branch — human
gets the agent, machine gets a tier-0 announcement, no answer gets a retry
policy and then a report back to the operator.

---

## The brain

### Recommended: ConversationRelay with Claude as the LLM

`<Connect><ConversationRelay url="wss://…">` hands the live call to a WebSocket
server you operate. Twilio performs speech-to-text on the caller, and
text-to-speech on whatever text you send back, and handles barge-in and
interruption. Twilio does **not** supply a WebSocket server and does **not**
integrate with any LLM — the conversational logic is entirely yours, running
outside Twilio.

That last sentence is the reason this is the recommendation. The per-call
system prompt is not a feature that has to be supported; it is a string your own
code builds. There is no override allowlist to enable, no set of fields blessed
as variable, no template language to fight.

The mechanism for getting per-call context in is **custom parameters**: key/value
pairs attached in the TwiML at dial time, which arrive in the `setup` message on
the WebSocket when the session opens. Read them, assemble the prompt, start the
conversation.

`welcomeGreeting` is the other essential piece: text spoken immediately on
session start, before any input. On an *outbound* call this is not a nicety —
the recipient said "hello?" and is waiting. The greeting is where the AI
disclosure goes.

### Call lifecycle

```
operator says "let Sam know the boiler's fixed"
        │
        ├─ resolve recipient  ──→ allowlist check ──→ refuse if absent
        ├─ compose brief
        ├─ SHOW BRIEF TO OPERATOR ──→ wait for confirmation      ← mandatory
        │
        ├─ CreateCall(To: Sam, From: claude-number,
        │             Twiml: <Connect><ConversationRelay
        │                      url=wss://…
        │                      welcomeGreeting="Hi Sam, this is Claude,
        │                        an AI assistant calling for Alex…">
        │                      <Parameter name="brief_id" value="…"/>
        │             MachineDetection: DetectMessageEnd)
        │
        ├─ [machine answered]  ──→ speak the tier-0 message, hang up
        │
        └─ [human answered]
              ws: setup    → look up brief_id, build system prompt
              ws: prompt   → transcribed speech from Sam
                             → Claude (streamed) → ws: text → TTS
              ws: end      → persist transcript
                             → report back into the operator's session
```

### Prompt composition

Three layers, assembled at `setup`. Only the third is per-call.

**1. Persona — fixed, identical on every call.**

> You are Claude, an AI assistant. You are speaking on the telephone, on behalf
> of {{operator_name}}. You are not {{operator_name}} and must never imply that
> you are. You have already introduced yourself as an AI in the greeting; if the
> person seems confused about who they are speaking to, say plainly that you are
> an AI assistant, not a person.

**2. Conduct — fixed, the boundaries of a notifier.**

> Deliver the message below and then let the call end. You may answer a
> question the message obviously raises, if the brief gives you the answer. You
> may not commit {{operator_name}} to anything, agree to any arrangement,
> speculate, or improvise facts. If asked something the brief does not cover,
> say you don't know and that the person should contact {{operator_name}}
> directly. Keep every reply to one or two short sentences — this is a phone
> call, not an essay.
>
> **You cannot take a message.** This call goes one way. If the person asks you
> to tell {{operator_name}} something, to pass anything on, to relay an answer,
> or to have {{operator_name}} call them back, you must decline using the
> wording below. Do not agree, do not offer to try, and do not soften it into a
> maybe. Then close the call warmly.

**2b. The refusal — verbatim, not paraphrased.**

> "I'm sorry — I can't take a message. I'm a one-way notification, so there's
> no way for me to make sure that reaches {{operator_name}} properly, and I
> don't want to tell you it will. The best thing is to contact
> {{operator_name}} directly."

This is fixed wording rather than something the model composes, for one
specific reason: declining runs against the grain of a helpful model, and a
model writing its own refusal will reliably soften it into a promise — *"I'll
make sure Alex knows"* — which is exactly the behaviour the design forbids.
Pinning the words removes the drift.

Note what the refusal does **not** say. It does not claim the words vanish; the
operator does receive a transcript, and an agent asserting otherwise would be
lying. It claims only that the agent will not *carry* the message, because it
cannot guarantee delivery. That distinction is the honest one and it is
explained in full in [the plan](plan.md#the-transcript-is-not-a-return-channel-and-the-agent-must-not-pretend-otherwise).

**3. Brief — written fresh for this call, thrown away afterwards.**

The one-time payload: who is being called and their relationship to the
operator, the message, anticipated questions with their sanctioned answers,
explicit prohibitions, and how to close. Schema and worked examples in
[`../examples/briefs/`](../examples/briefs).

The value of the split is that layers 1 and 2 are reviewed once, carefully, and
then never touched — so the per-call surface Claude is composing on the fly is
only ever the content, never the guardrails.

### Voice and language

**English only.** Settled 2026-08-12.

This was the open question with the most power to overturn the platform choice:
had these calls needed Hebrew, voice quality would have outranked every other
criterion in this document, and a platform with better Hebrew would have won
regardless of how clean its per-call prompt story was. It doesn't apply, so the
recommendation stands on its merits.

ConversationRelay does let you select the TTS provider, the voice, and the STT
language per session, so adding a language later is configuration rather than
rearchitecture. But "configurable" and "good enough to send to a family member"
are different claims, and only a spike would distinguish them — so if the
recipient list ever grows to include someone who should be called in Hebrew,
treat that as reopening the platform question, not as changing a setting.

---

## Where the code will live

Deliberately not decided yet, but the shape is constrained: ConversationRelay
needs a **publicly reachable WebSocket endpoint** with a stable hostname and
TLS. The existing pattern in this estate — a container on residenceserver
behind a Cloudflare tunnel, as `Call-Archive-UI` already does — fits without
inventing anything. That is a Phase 2 decision, and it should not be made
before the spike proves the approach is worth hosting.
