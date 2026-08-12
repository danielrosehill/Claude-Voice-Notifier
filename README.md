# Claude-Voice-Notifier

**A planning repository.** What if "Claude, could you let Sam know the boiler's
fixed?" ended with the phone ringing in Sam's pocket?

Claude already runs errands across the channels it has hands on: it sends the
email, it uploads the file to Drive, it files the invoice. Voice is the missing
channel — and it is the one the people around you actually answer.

This repo works out how to add it: Claude places a real outbound phone call to a
person you nominate, and delivers one specific, one-time message on your behalf.

Status: **planning, opened 2026-08-12. No code yet.** The plan is
[`docs/plan.md`](docs/plan.md).

> All names, numbers and scenarios in this repo are fictional placeholders.
> "Alex" is the operator, "Sam" is their partner, "Robin" is a parent-in-law.
> See [`examples/README.md`](examples/README.md).

---

## Notifier, deliberately

The name is a scope boundary, not just a label. The job is to **deliver a
notification by voice**: Claude has something the operator wants a specific
person to know, and the telephone is the channel most likely to actually reach
them.

The right mental model is a **pager**, not an assistant. It goes one way. It
reaches you, it tells you the thing, and that is the end of the transaction.

Concretely, and this is the load-bearing constraint of the whole design:

> **There is no return channel.** The agent may talk — it can answer a question
> the message obviously raises, from answers written into the brief in advance.
> But it cannot carry anything back. If the recipient says "tell Alex that
> Thursday doesn't work", the agent declines, in as many words: it is a one-way
> notification, it has no ability to deliver a message, and they should contact
> Alex directly.

That refusal is a fixed template, not something the model composes in the
moment. See [the conduct layer](docs/architecture.md#prompt-composition).

Refusing to take messages sounds like a limitation and is actually the feature.
The failure mode that would make this product harmful is a misheard reply
arriving in the operator's session with the authority of a direct quote, and
being acted on. Removing the return channel removes that failure mode entirely,
rather than trying to engineer around it.

## The shape of the thing

It splits cleanly into two problems that can be solved independently.

**1. The line.** Claude needs a way to originate a call, and a caller identity
the recipient recognises. Twilio, already in use here.

**2. The brain.** A voice agent whose instructions are composed *per call*. Not
a bot with a fixed job description — a stable persona ("you are Claude, calling
on behalf of Alex, you are an AI and you say so") wrapped around a brief written
fresh for this one call and never used again:

> *Tell Robin the plumber came at two, replaced the valve under the sink, and
> that the water is back on. If Robin asks about the cost, say Alex will send
> the invoice by email this evening. Do not agree to any follow-up visit.*

The second problem is the interesting one, and it is the axis on which the
platform choice turns. Most hosted voice-agent products are built for the
opposite case — one assistant, thousands of calls, a few variables swapped in.
This needs the inverse: **one call, a wholly new prompt, every time.** Some
platforms allow full prompt replacement per call, some only template fills.
That distinction drives the whole survey in
[`docs/framework-survey.md`](docs/framework-survey.md).

## What the plan concludes

Short version, so you don't have to read four documents to get the punchline:

- **There are two tiers, and tier 0 is worth shipping on its own.** A spoken
  notification with no conversation at all needs no voice agent — just TwiML
  `<Say>` and the Twilio tooling already in place. It is buildable in an
  afternoon, it is impossible for it to say anything unplanned, and for a
  *notifier* it may honestly be most of the product. Don't let the interesting
  problem block the easy win.
- **English only, and no return channel.** Both settled 2026-08-12. Hebrew is
  not required for now, which removes the one criterion that could have
  overturned the platform choice. The return channel is not deferred, it is
  rejected — see above.
- **For the conversational tier, Twilio ConversationRelay with Claude as the
  LLM.** Twilio does speech-to-text, text-to-speech and interruption handling;
  you supply the model over a WebSocket you control, which means the system
  prompt is just a string your own code assembles at call setup. No vendor
  override feature to enable, no allowlist of which fields may vary. And it
  makes "you are Claude" literally true rather than a costume.
- **"A SIP credential for Claude" is the right instinct via the wrong
  mechanism.** Outbound Twilio calls need no SIP at all. What the instinct is
  really reaching for is a *distinct caller identity* — a dedicated number the
  household saves as "Claude (Alex's assistant)", so the call announces itself
  before it is answered. SIP only re-enters if a SIP-native voice platform is
  chosen instead. See [`docs/architecture.md`](docs/architecture.md).
- **The etiquette constraints are load-bearing, not a footer.** These calls go
  to real people who did not opt in. Identify as an AI in the first sentence,
  every call. Allowlist the recipients. Confirm the brief with the operator
  before dialling. [`docs/etiquette-and-safety.md`](docs/etiquette-and-safety.md)
  treats this as a design input.

## Layout

```
docs/
  plan.md                    # phases, decisions, open questions — start here
  architecture.md            # the two layers, prompt composition, call lifecycle
  framework-survey.md        # how each platform does per-call instructions
  etiquette-and-safety.md    # disclosure, allowlists, failure modes
examples/
  briefs/                    # what a call brief looks like, fictional
```

When this stops being a planning repo it becomes a Claude Code plugin, and
grows `.claude-plugin/`, `skills/` and `scripts/`. Not before — there is
nothing to put in them yet.

## Sibling repos

This is the outbound half of a set that already exists:

| Repo | Direction | What it is |
|---|---|---|
| `Twilio-Manager` | inbound | Version-controlled call flows for the live lines |
| `Twilio-Ops-Plugin` | inbound | Claude Code plugin for deploying and cutting over those lines |
| `Agent-IVR` | inbound | Idea repo: a phone menu branch for machine callers |
| **`Claude-Voice-Notifier`** | **outbound** | **This — Claude places the call** |

`Agent-IVR` asks what happens when an agent *calls a business*. This asks what
happens when an agent *calls a person you know*. They are close enough to share
prior art and far enough apart in etiquette to stay separate.

## Licence

MIT.
