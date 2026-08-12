# Etiquette and safety

This is a design input, not a compliance appendix. Every other document here
describes a system that rings a real person's phone — someone who did not sign
up for it, is not the customer, and cannot opt out. That constraint shapes the
architecture, so it is written down before the code exists.

The governing asymmetry: **the operator gets the convenience and the recipient
carries the cost.** Most rules below fall out of taking that seriously.

---

## Non-negotiable

**1. Identify as an AI in the first sentence, every call, without exception.**

Not on request. Not if asked. First. It goes in the `welcomeGreeting`, which is
spoken before the recipient can say anything, so it cannot be skipped by an
unlucky turn of conversation:

> "Hi Sam, this is Claude — I'm an AI assistant calling on behalf of Alex. Alex
> asked me to let you know…"

**2. Never imitate the operator.** No voice cloning, ever, for any recipient,
including the operator themselves. The persona layer states this and the voice
selection must not be a voice anyone in the household would mistake for a
person they know.

**3. Never claim to be a person.** If the recipient asks "is this a real
person?", the answer is no, plainly and immediately. If they seem confused
about who they are talking to, stop and re-identify without being asked.

**4. Confirm the brief with the operator before dialling.** Always. A phone
call to a third party is irreversible and outward-facing — once it has rung, it
cannot be recalled, and the operator's relationship absorbs any error. The
operator sees the brief and says go. There is no "quiet mode" for this and it
should not become a setting, because the entire failure mode of the product is
a call the operator did not fully intend.

**5. An allowlist, not a phone book.** Claude may call numbers that have been
explicitly added to a recipient list, and only those. It must never dial a
number it inferred from a document, an email signature, a contacts search, or
the conversation. The allowlist is edited by the operator, deliberately, out of
band.

---

## Design rules that follow

**A dedicated number** so the call is identified before it is answered — see
[architecture](architecture.md#what-the-instinct-was-actually-after-a-caller-identity).
Pre-identification on the lock screen is better than any disclosure spoken after
pickup, because it lets the recipient decline without the interaction happening
at all.

**Time-of-day window.** Nothing outside roughly 08:00–21:00 local to the
*recipient*, and no exception flag in v1. If it is urgent enough to justify
waking someone, it is urgent enough for the operator to make the call
themselves. This is also the correct default because the operator and the
recipients here are not always in the same time zone.

**Turn and duration caps.** Hard limits — a small number of exchanges and a
short wall-clock ceiling. A notifier that is still talking after two minutes
has failed at being a notifier, and an agent stuck in a loop on a live call is
both expensive and alarming.

**A graceful exit that always exists.** Any of: the recipient asks for the
operator, the conversation leaves the brief, the recipient sounds distressed,
the caps are hit. All route to the same close:

> "I'll let Alex know you asked, and Alex will get back to you. Thanks Sam —
> bye now."

Then hang up. The agent never tries to hold the call.

**No commitments.** The agent cannot agree to a time, accept an offer, confirm
an arrangement, or answer on the operator's behalf about anything the brief did
not pre-authorise. "I don't know, Alex will follow up" is always available and
always acceptable.

**Content the agent must never carry.** Health information, financial detail,
credentials, anything about a third party who is not on the call, and anything
the operator would not put in a text message. A synthesised voice reaching an
unverified pickup is the wrong channel for all of it — and note the agent
cannot verify *who actually answered*, only which handset rang.

**Voicemail is a different mode.** If answering-machine detection reports a
machine, drop to the tier-0 announcement and hang up. A conversational agent
should never be exposed to a voicemail greeting: it will try to converse with
it, and the recording that results is the single most embarrassing artefact
this system can produce.

---

## Recording

Design position: **transcript only, no stored audio**, in the first version.

The transcript is what the outcome loop needs — it goes back into the operator's
session so Claude can report what was said. Audio adds nothing to that and
changes the character of what is being asked of the recipient. Fewer things to
justify, fewer things to hold.

Whatever is retained, the recipient is told plainly if they ask, and the
retention period is short and stated in the code rather than implied by
whatever the storage bucket happens to do.

---

## Recipient consent, in practice

Before a person is added to the allowlist, the operator tells them — in person,
as a human — that this exists, what it will sound like, and that they can say
they'd rather not. That conversation is not a legal formality; it is the thing
that makes the first call land as useful rather than unsettling.

It is also worth being honest that some people will simply not want this, and
the correct response to that is to remove them from the list, not to tune the
greeting.

## The test to apply

Before adding any capability, ask the question in the recipient's voice:

> *Would Sam, hearing about this feature afterwards, feel it was reasonable —
> or feel that something had been done to them?*

Tier 0 and tier 1 pass. Some of the obvious tier 2 extensions do not, which is
most of why tier 2 is deferred.
