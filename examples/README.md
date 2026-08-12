# Examples

**Everything here is fictional.** No real name, number, address or event
appears in this repository. The cast is fixed so the examples read
consistently:

| Placeholder | Role |
|---|---|
| **Alex** | the operator — the person Claude is acting for |
| **Sam** | Alex's partner |
| **Robin** | Alex's parent-in-law |

Phone numbers use the reserved `+1 555 01xx` range, which is not routable.

## What a brief is

The **brief** is the one-time, per-call payload: everything that is specific to
*this* call and will never be reused. It is the third layer of the prompt —
layers one and two (persona and conduct) are fixed and live in the code, not
here. See [`../docs/architecture.md`](../docs/architecture.md#prompt-composition).

Claude composes the brief from what the operator said, shows it to the operator,
and only dials once it is confirmed.

## Fields

| Field | Meaning |
|---|---|
| `brief_id` | Opaque handle. This is what travels in the TwiML custom parameters — the brief itself never goes over the wire to Twilio |
| `tier` | `0` announcement (no LLM in the call path), `1` conversational |
| `recipient` | Must resolve to an allowlist entry. `relationship` is prompt material — it sets the register |
| `message` | The thing being conveyed. Written to be *spoken*, not read |
| `anticipated_questions` | Pre-authorised answers. Anything not here gets "I don't know — contact Alex directly". A question that is really a request to pass a message on gets `REFUSAL`, meaning the fixed no-message wording from the conduct layer |
| `prohibitions` | Explicit negative constraints for this call, on top of the standing ones in the conduct layer |
| `closing` | How to end, so the agent doesn't improvise a goodbye |
| `caps` | Hard turn and duration limits |
| `voicemail` | What to say if a machine answers. Always present — a tier 1 call degrades to a tier 0 announcement |

## Writing a good `message`

The recurring failure is briefs written as prose to be read rather than speech
to be heard. Rules of thumb:

- One idea per sentence, and few sentences. The recipient cannot re-read it.
- Front-load the point. "The boiler's fixed" before the explanation of how.
- Times and numbers spoken as a person would say them — "just after two", not
  "14:07".
- No structure that only works visually: no lists, no parentheses, no "firstly".
- If it needs more than about four sentences, it is an email, not a call.

## Files

- [`briefs/tier0-boiler-fixed.json`](briefs/tier0-boiler-fixed.json) — the
  simple case. One-way, no agent, no conversation.
- [`briefs/tier1-plumber-visit.json`](briefs/tier1-plumber-visit.json) — the
  case that needs the voice agent, and shows what `anticipated_questions` and
  `prohibitions` are for. Note the "can I speak to Alex?" entry: that is a
  message-taking request wearing a question's clothes, and it resolves to a
  refusal rather than an answer.

## The one thing a brief cannot authorise

A brief can widen what the agent may *say*. It can never grant a return
channel. No brief may instruct the agent to take a message, accept a reply,
promise a callback, or relay an answer — those are refused regardless of what
the brief says, because the prohibition lives in the fixed conduct layer above
it. See [`../docs/plan.md`](../docs/plan.md#decisions-taken), D10.
