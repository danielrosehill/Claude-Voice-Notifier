# Voice platforms: how each one lets you change the instructions per call

Surveyed 2026-08-12. Verification status is marked per claim — some of this is
read from vendor documentation, some is from search summaries that I could not
open the underlying page for. Treat the unverified rows as leads, not facts.

## The question the survey is actually asking

Nearly every hosted voice-agent product is built around a **saved assistant**:
you write one system prompt, tune it over weeks, and run thousands of calls
through it, substituting a name and an appointment time. The prompt is the
stable asset; the call is the disposable thing.

This project inverts that. The persona is stable, but **the instruction is
written fresh for one call and never used again**. So the discriminating
question is not "does it support dynamic variables" — almost everything does —
but:

> **Can the entire system prompt be replaced, per call, at dial time, without
> pre-registering what may vary?**

That splits the field into three groups.

| Group | Mechanism | Fit |
|---|---|---|
| **You own the prompt** | The prompt is a string in your own code | Perfect — nothing to work around |
| **Full replacement, gated** | Vendor permits prompt replacement, but you must pre-enable it or use a specific mode | Workable |
| **Template fill only** | You may substitute values into a saved prompt | Wrong shape — every possible instruction would have to fit one template |

---

## The options

### Twilio ConversationRelay + Claude — **recommended**

*Group: you own the prompt.*

TwiML `<Connect><ConversationRelay url="wss://…">` hands the audio to a
WebSocket server you run. Twilio does STT, TTS, and interruption handling.
Twilio does not supply the WebSocket server and does not integrate with any LLM
provider — conversational logic and the model call are entirely yours, outside
Twilio. *(Verified: Twilio ConversationRelay docs.)*

- **Per-call instructions:** unlimited, because there is no vendor-side prompt
  at all. Per-call context arrives as **custom parameters** set in the TwiML,
  delivered in the `setup` WebSocket message when the session opens. *(Verified:
  Twilio WebSocket messages docs.)*
- **Speaking first:** `welcomeGreeting` is spoken immediately on session start,
  before any input — documented as particularly useful for outbound calls.
  *(Verified.)*
- **Claude as the model:** Twilio publishes a three-part Anthropic integration
  series — basic integration, then token streaming and interruption handling,
  then tool calling. *(Verified: the three posts exist and are Anthropic-specific.)*
- **Known gotcha:** naive implementations feel laggy; buffer streamed tokens
  into larger chunks before sending to TTS. Reconnection is handled through the
  `<Connect>` action-URL callback, returning fresh
  `<Connect><ConversationRelay>` TwiML with the same `callSid`. *(Verified.)*

**Why it wins here:** it is the only option where the per-call-prompt
requirement is not a feature at all. It also keeps everything on Twilio, which
is already in use, already version-controlled, and already has an ops plugin.
And Claude genuinely being the model in the loop makes the persona honest rather
than a costume.

**Costs:** you operate a WebSocket server, and you own latency. There is no
dashboard, no built-in transcript viewer, no evaluation tooling.

---

### Vapi — considered, not chosen

*Group: full replacement, gated — and it is the good kind of gate.*

- **Per-call instructions:** a call to `/call` takes **either** `assistantId`
  (saved) **or** `assistant` (a *transient* assistant defined inline in the
  request). The transient path means the whole assistant, prompt included, is
  supplied per call — which does meet the requirement. Separately,
  `assistantOverrides.variableValues` fills `{{name}}`-style placeholders in a
  saved assistant. *(Verified: Vapi outbound-calling and dynamic-variables docs.)*
- **Gotcha:** for per-recipient overrides you must call the endpoint once per
  destination; the `customers` array does not carry per-customer overrides.
  *(Verified.)*
- **Gotcha:** transient assistants are also the documented way to pin a static
  version for scheduled calls — with `assistantId`, deleting the saved assistant
  makes the call fail. *(Verified.)*

**Why not:** two reasons, one specific and one structural. The operator reports
mixed results with Vapi in prior use, which is the kind of evidence no docs page
outranks. And structurally it is a middleware layer bought to solve a problem
this project doesn't have — orchestrating STT/LLM/TTS across providers at
volume — while adding a vendor between Claude and the call for no gain, since
the transient-assistant prompt still has to be composed by our code anyway.

**But keep it warm.** If the Phase 2 ConversationRelay spike disappoints —
latency, Hebrew voice quality, or the WebSocket server proving more operational
burden than it is worth — Vapi's transient assistant is the first fallback, and
it would get to a working call faster than anything else on this list.

---

### ElevenLabs Agents

*Group: full replacement, gated — the more awkward kind.*

- **Per-call instructions:** overrides are **disabled by default for security**
  and must be toggled on per field in the agent's Security tab before they can
  be sent. The overridable set includes system prompt, first message, language,
  voice ID, LLM, tools, and knowledge base. In the agent JSON these live under
  `platform_settings.overrides.conversation_config_override`, with keys
  `first_message`, `prompt.prompt`, `prompt.tool_ids`, `prompt.knowledge_base`,
  `language`, `asr.keywords`. Values are then passed at conversation start as a
  `conversation_config_override` payload. *(Verified: ElevenLabs overrides doc.)*
- **Sharp edge:** sending an override for a field that was not enabled raises an
  error for most fields — but `asr.keywords` is a documented "soft disallow"
  that silently drops instead. Tool and knowledge-base overrides *replace*
  rather than merge. Omit fields you don't want changed; do not pass empty
  strings. *(Verified.)*
- **Their own advice:** use dynamic variables for injecting real-time data and
  reserve overrides for full replacement of prompts or first messages — which
  is precisely this use case, so overrides is the right mechanism here.
  *(Verified.)*
- **Not verified:** whether and how `conversation_config_override` is accepted
  on an *outbound* Twilio call request. The outbound-calling doc page 404'd at
  the URL tried; batch-calling and SIP-trunk pages exist and are the next place
  to look. **This is the one gap that would need closing before ElevenLabs could
  be chosen**, since an override mechanism that only works on inbound or on
  browser-initiated sessions would be useless here.

**In its favour:** best-in-class TTS, native SIP trunking, and it cut
Conversational AI pricing by roughly half in February 2026 *(unverified — from a
search summary)*. If Hebrew voice quality turns out to be the binding
constraint, this is the option that most deserves a second look.

---

### OpenAI Realtime API over SIP

*Group: you own the prompt.*

- **Per-call instructions:** `POST /v1/realtime/calls/{call_id}/accept` with a
  body carrying `type: "realtime"`, `model`, and `instructions` — a free-text
  system message built fresh on each webhook invocation, so it can vary per call
  on anything you can see in the incoming event. *(Verified: OpenAI Realtime SIP
  guide and Calls API reference.)*
- **Outbound is indirect.** The documented SIP flow is inbound: OpenAI fires a
  `realtime.call.incoming` webhook and you accept it. To dial out, you originate
  from the trunking provider and bridge into OpenAI, where it arrives as an
  incoming call you then accept with per-call instructions. Twilio publishes an
  Elastic SIP Trunking integration for exactly this. *(Verified.)*
- **Gotchas:** SIP sessions reject `threshold`, `prefix_padding_ms` and
  `silence_duration_ms` under `session.audio.input.turn_detection`. Instructions
  alone will not make the agent speak first — you need a separate
  `responseCreate` fired on accept, which for an outbound notification is
  essential. There is a `/reject` companion defaulting to SIP 603. *(Verified.)*

**Why not:** it is architecturally excellent and thematically wrong. Building a
"you are Claude" notifier on a competitor's realtime model is a strange choice
when Twilio publishes a documented Claude integration that reaches the same
place. It also reintroduces the SIP trunk that decision D5 removed. Worth
knowing about; not the default.

---

### LiveKit Agents / Pipecat

*Group: you own the prompt — completely.*

Self-operated frameworks. Instruction state is manipulated directly in your own
code, so per-call prompts are trivial. LiveKit Agents reached 1.0 in April 2025
with adaptive turn detection and native MCP tool support on the Python 1.5.x
line; Pipecat is a Python framework giving fully modular control of the
STT/LLM/TTS pipeline. Both do SIP. *(Unverified — from search summaries, not
read from source docs.)*

**Why not:** these are the right answer at volume, for a product, with a team.
The commonly cited heuristic is to use a managed platform below roughly 10k
minutes a month and move to LiveKit or Pipecat above it. This project's volume
is a handful of calls a week. Adopting a full media framework to make one phone
call a day is the wrong trade — ConversationRelay gets the same prompt freedom
while letting Twilio own the media pipeline.

---

### Retell, Bland

*Group: template fill (Retell), unassessed (Bland).*

Retell's per-call mechanism is understood to be `retell_llm_dynamic_variables`
— substitution into a saved prompt rather than replacement of it. **Unverified;
this was not read from Retell's documentation and should be checked before the
row is trusted.** If accurate, it is the wrong shape for this project: every
conceivable household notification would have to be expressible as fills into a
single fixed template. Retell's compensating virtue is a flatter, more
predictable per-minute price. Bland was not assessed.

---

## Summary

| Platform | Full per-call prompt replacement? | Speaks first on outbound? | Verified |
|---|---|---|---|
| **ConversationRelay + Claude** | **Yes — the prompt is yours** | `welcomeGreeting` | Docs read |
| Vapi (transient assistant) | Yes, via inline `assistant` | Assistant first-message | Docs read |
| ElevenLabs Agents | Yes, if pre-enabled per field | First message, overridable | Overrides read; **outbound path unverified** |
| OpenAI Realtime SIP | Yes — `instructions` on accept | Needs explicit `responseCreate` | Docs read |
| LiveKit / Pipecat | Yes — it's your code | Yours | Search summaries only |
| Retell | Believed template-fill only | Unknown | **Unverified** |

## What to do next

Close the two unverified rows that could change the decision — the ElevenLabs
outbound override path, and whether Retell permits full prompt replacement —
only *if* the Phase 2 ConversationRelay spike goes badly. If it goes well, the
survey has done its job and these stay open.

## Sources

- [Twilio: ConversationRelay](https://www.twilio.com/docs/voice/conversationrelay)
- [Twilio: ConversationRelay WebSocket messages](https://www.twilio.com/docs/voice/conversationrelay/websocket-messages)
- [Twilio: Integrate Claude with Twilio Voice using ConversationRelay](https://www.twilio.com/en-us/blog/integrate-anthropic-twilio-voice-using-conversationrelay)
- [Twilio: Token streaming and interruption handling with Anthropic](https://www.twilio.com/en-us/blog/anthropic-conversationrelay-token-streaming-interruptions-javascript)
- [Twilio: Function calling with Twilio Voice and Claude](https://www.twilio.com/en-us/blog/developers/tutorials/product/function-calling-twilio-voice-anthropic-claude-integration)
- [Twilio: OpenAI Realtime SIP connector with Elastic SIP Trunking](https://www.twilio.com/en-us/blog/developers/tutorials/product/openai-realtime-api-elastic-sip-trunking)
- [ElevenLabs: Agent overrides](https://elevenlabs.io/docs/agents-platform/customization/personalization/overrides)
- [Vapi: Outbound calling](https://docs.vapi.ai/calls/outbound-calling)
- [Vapi: Variables](https://docs.vapi.ai/assistants/dynamic-variables)
- [OpenAI: Realtime API with SIP](https://developers.openai.com/api/docs/guides/realtime-sip)
- [OpenAI: Realtime Calls API reference](https://platform.openai.com/docs/api-reference/realtime-calls/accept-call)
