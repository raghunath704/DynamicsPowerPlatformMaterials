

# Orchestration & topic engine

## 1. Generative orchestration vs. classic orchestration — what changes at runtime, and what breaks on migration?

**Classic orchestration** is a deterministic NLU pipeline. The user utterance is scored against trigger phrases across all topics, the highest-confidence intent above threshold wins, exactly **one topic** is invoked per turn, and control stays in that topic until it ends or redirects. Entities are extracted by the recognizer and mapped into topic input variables.

**Generative orchestration** replaces the intent classifier with an LLM planner. At each turn the planner is handed the agent's **instructions**, plus the *names and descriptions* of every topic, tool (connector action, flow, prompt, MCP tool), connected agent, and knowledge source. It then builds a plan — which may chain **multiple** tools/topics in a single turn, pass outputs of one as inputs to the next, ask the user only for the slots it genuinely can't infer, and compose a natural-language response from the results.

**What breaks on migration:**
- Topics with rich trigger phrases but **blank descriptions** silently stop firing — routing now depends on the description.
- Designs built on `Redirect` chains and "Go to step" get bypassed, because the planner can re-enter or skip.
- Topic-scoped variable assumptions break — one turn may span several topics.
- Deterministic scripted output becomes paraphrased unless you pin it (message nodes still render, but surrounding text is generated).
- Your regression tests are invalid: the same input can produce different plans across runs, so you need semantic/behavioral assertions, not string equality.

I always migrate incrementally: turn it on in a dev copy, rewrite descriptions first, then re-test the top 20 intents.

---

## 2. What does the orchestrator actually "see" when selecting a tool, and how do you prevent mis-selection?

The planner's context window contains: the **agent instructions**, the **display name + description** of every enabled topic/tool/agent/knowledge source, each tool's **input parameter names, types, and descriptions**, the **output description**, and the recent conversation history. It does *not* see your node graph, your Power Fx, or your trigger phrases.

That means the description **is** the API contract. My rules:

- **Verb-first, outcome-oriented:** "Creates a new expense report for the signed-in user" — not "Expense stuff".
- **State the boundary explicitly.** Overlap is resolved by exclusion clauses: `Use this ONLY for expenses already submitted. For draft expenses, use Get Draft Expenses instead.`
- **Describe every input as if to a junior dev**, including format: `employeeId — the 6-digit numeric employee ID, not the email address.`
- **Describe the output**, because the planner uses it to decide whether it can chain into the next tool.
- **Prune ruthlessly.** Twenty half-described tools perform far worse than eight well-described ones. If two tools are genuinely near-duplicates, merge them and branch internally.

When mis-selection happens, I read the **orchestration trace / activity map** in the test pane to see which tool was chosen and why, then fix the description — not the logic.

---

## 3. Trigger phrases vs. topic description in generative mode

In **classic** mode, trigger phrases are the training utterances for the intent recognizer — they're everything. In **generative** mode they are effectively **ignored for routing**; the planner selects on the topic's **description**.

The classic trap: a team spends weeks curating 30 trigger phrases per topic, flips the agent to generative orchestration to get knowledge and multi-tool chaining, and the agent suddenly "forgets" half its topics — because descriptions were left empty or default. Nothing errors; the topics just never get planned.

Practical guidance I give teams: treat trigger phrases as **documentation of user language** and mine them to write the description. A good pattern is: one sentence on what it does, one sentence on when to use it, one on when *not* to. Keep trigger phrases populated anyway — they're your fallback if you ever revert to classic, and some system/trigger behaviors (and event-triggered topics) still rely on them.

---

## 4. Multi-turn slot filling across chained actions — and stopping duplicate prompts

The planner maintains a working set of known values from: existing variable state (global/topic), entities extracted from the current and prior utterances, outputs of tools already executed this turn, and authenticated user context. When a tool needs an input, the orchestrator first tries to **fill from context**; only if it can't will it generate a clarifying question.

The re-prompt problem almost always comes from one of four causes:

1. **Wrong variable scope** — the value lives in a topic variable that died when the topic ended. Fix: promote to a **global variable**.
2. **Input configured as "the agent should ask"** rather than "fill from conversation / pass programmatically." Each topic/tool input has this setting; set it deliberately.
3. **No description on the input**, so the planner can't match the known value to the slot semantically.
4. **Type mismatch** — you captured a string but the tool wants a record/choice, so it discards it and asks again.

For values that should persist across sessions (preferred language, cost centre), I store them in **Dataverse keyed on `System.User.Id`** and rehydrate them in `Conversation Start`, rather than relying on conversation state.

---

## 5. Conversational boosting vs. Fallback — and replacing them cleanly

**Conversational boosting** is a system topic containing a **Generative answers** node. In classic orchestration, when no custom topic matches with sufficient confidence, boosting fires: it queries your configured knowledge sources and returns a grounded, cited answer.

**Fallback** (`System Fallback`) fires when nothing matched **and** boosting is disabled, or boosting found no grounded answer and you've routed the "no answer" path there. It's your last-resort handler — typically "I didn't get that" → escalate or re-prompt, with a `System.Fallback.RepeatedCount` guard to escalate after N consecutive misses.

To replace boosting cleanly, don't delete it — **edit it**. Open Conversational boosting and either point the Generative answers node at a narrower data source, or replace it with a `Redirect` to your own topic that does custom retrieval (e.g., an HTTP call to Azure AI Search) and then feeds results into a Generative answers node using **"Data from a variable"** as the content source. That preserves the system routing hook while giving you full control.

In **generative orchestration**, knowledge is a first-class tool the planner selects directly, so boosting matters much less — but Fallback still guards the "planner produced nothing" case.

---

## 6. Precedence when a topic, a system topic, and knowledge all match

**Classic orchestration** resolves in this order:
1. **Active topic / interruption rules** — if you're mid-topic, the current topic's question node consumes the input unless interruption is allowed and another topic scores high enough.
2. **Event/system triggers** — `Conversation Start`, `Escalate`, `End of Conversation`, `On Error`, `Sign in` fire on their events, not on NLU.
3. **Custom topic NLU match** above the confidence threshold — highest score wins. Custom topics outrank system topics on ordinary utterances.
4. **Conversational boosting** (knowledge) if nothing matched.
5. **Fallback.**

So knowledge is explicitly *last* in classic — a weak topic match still beats a great grounded answer, which is why over-broad trigger phrases starve your knowledge base.

**Generative orchestration** flattens this: topics, tools and knowledge are all candidates the planner ranks semantically in one pass, and it can use several. Practically that means knowledge competes on equal footing, and precedence is controlled by **description quality and agent instructions** ("Always prefer the Order Status tool over knowledge for order questions"), not by an engine-level hierarchy.

---

# Variables, Power Fx, logic

## 7. Global vs. Topic vs. System vs. Environment variables

| | Scope | Lifetime | Notes |
|---|---|---|---|
| **Topic** (`Topic.x`) | Single topic instance | Dies when the topic ends | Default scope. Must be explicitly declared as input/output to cross boundaries |
| **Global** (`Global.x`) | Whole agent | Whole conversation session | Can be marked *"External sources can set values"* (query string / Direct Line) and *"Persist across sessions"* for authenticated users |
| **System** (`System.x`) | Runtime-provided | Read-only | Channel, user, conversation, activity, locale, auth token |
| **Environment** (`Env.x`) | Solution/environment | Deploy-time config, not conversation state | Used for URLs, site IDs, feature switches — the correct ALM mechanism |

**Redirect vs. call:** `Redirect to another topic` transfers control and *can* pass declared **inputs**; when the child topic ends it returns declared **outputs** to the caller. Topic variables that aren't declared as inputs/outputs are **not** visible across the boundary — that's the number one source of "my variable is blank" bugs. Global variables survive both.

Rule of thumb I use: topic variables for scratch, globals for conversation-scoped user state, Dataverse for cross-session state, environment variables for anything that differs between dev/test/prod.

---

## 8. System variables for channel, user, locale, conversation — and `User.Id` vs `Conversation.Id`

Key ones I use daily:

- `System.Activity.ChannelId` — `msteams`, `directline`, `webchat`, `msdyn365copilot`, etc. Essential for channel-conditional rendering (adaptive cards behave differently in Teams vs. web).
- `System.User.Id` — stable identity of the user **within the channel**. For Teams + authentication it maps to the Entra identity; unauthenticated web chat gives an anonymous per-session ID.
- `System.User.DisplayName`, `System.User.PrincipalName`, `System.User.Email` — populated only when authenticated.
- `System.User.IsLoggedIn` and `System.User.AccessToken` — auth state and bearer token (manual auth).
- `System.Conversation.Id` — unique per conversation session.
- `System.UserLanguage` / `System.Activity.Locale` — for multilingual routing.
- `System.LastMessage.Text` / `System.Activity.Text` — raw utterance, useful for logging and for passing to a prompt.

**`User.Id` vs `Conversation.Id`:** use `User.Id` as the key for anything that must survive across sessions — preferences, consent, cached profile. Use `Conversation.Id` as the correlation key for logging/telemetry and for anything scoped to this session only (a cart, a draft). Critical caveat: **only trust `User.Id` for identity when the user is authenticated** — on an anonymous web channel it's spoofable/ephemeral and must never be used as a security boundary.

---

## 9. Parsing a JSON HTTP response into a typed table with Power Fx

`ParseJSON()` returns an **Untyped Object**, so nothing is usable until you cast it. Given an HTTP response stored in `Topic.RawResponse`:

```powerappsfl
// 1. Parse to untyped
Set(Topic.Parsed, ParseJSON(Topic.RawResponse))

// 2. Project into a typed table
Set(
  Topic.Orders,
  ForAll(
    Table(Topic.Parsed.value),
    {
      OrderId:   Text(ThisRecord.Value.orderId),
      Customer:  Text(ThisRecord.Value.customer.name),
      Amount:    Value(ThisRecord.Value.amount),
      Placed:    DateValue(Text(ThisRecord.Value.placedOn)),
      IsRush:    Boolean(ThisRecord.Value.rush)
    }
  )
)
```

Points I'd make in the interview:
- `Table(untypedArray)` converts an untyped array into a table of untyped records; inside `ForAll` you reach the element via `ThisRecord.Value`.
- **Every** field needs an explicit cast — `Text()`, `Value()`, `Boolean()`, `DateValue()`. Skipping the cast is the classic error.
- Indexing a single element: `Index(Table(Topic.Parsed.value), 1).Value.orderId`.
- Missing keys return *blank*, not an error, so guard with `IsBlank()` / `Coalesce()`.
- Declare the target variable's type in the variable properties (Table with a defined schema) so downstream nodes and adaptive cards can bind to it.
- If the payload is deep or the shape varies, I do the shaping in an **agent flow / Power Automate** with `Parse JSON` and a schema instead — far more maintainable than nested Power Fx.

---

## 10. Power Fx limits in Copilot Studio, and workarounds

Copilot Studio supports the **expression (non-behavioral)** subset of Power Fx. Available: string functions (`Concatenate`, `Left`, `Substitute`, `Split`, `Match`/`IsMatch` regex), math, logic (`If`, `Switch`, `And`/`Or`), date/time (`Now`, `DateAdd`, `DateDiff`, `Text` formatting), table/record shaping (`ForAll`, `Filter`, `Sort`, `AddColumns`, `First`, `Index`, `CountRows`), `ParseJSON`, `JSON`, `Coalesce`, `IsBlank`.

**Not available:**
- **Behavioral functions** — `Patch`, `Collect`, `Remove`, `Notify`, `Navigate`, `SubmitForm`. There's no data-write surface in a topic.
- **Direct data source access** — you can't `Filter(SharePointList, …)`; connectors are only reachable as actions.
- **UI/control functions** and anything referencing controls (`Self`, `Parent`).
- **`Set()`** as a function — you use the *Set a variable value* node instead.
- `User()`, `Defaults()`, and app-lifecycle functions.

**Workarounds:** anything imperative or data-touching goes into a **Power Automate cloud flow or agent flow**, which is called as an action and returns typed outputs. For heavy transformation, I push it server-side (flow, Azure Function, or the API itself) rather than fighting Power Fx — topic-level Power Fx should stay declarative and shallow, because it's untestable and unversioned in isolation.

---

## 11. Retry loop with a max-attempts guard, and why `Go to step` is risky

There's no loop node, so the canonical pattern is:

1. **Set variable** `Topic.Attempts = 0` before the loop.
2. Add a **label-able node** that starts the retry block (e.g., the *Call an action* node) — this is the jump target.
3. Call the action / HTTP request.
4. **Condition** on success → exit branch.
5. On failure: **Set variable** `Topic.Attempts = Topic.Attempts + 1`.
6. **Condition**: `Topic.Attempts < 3` → **Go to step** (jump back to step 3). Else → error branch: friendly message + escalate/log.

Why `Go to step` is risky:
- **No compiler guarantee of termination.** Forget the increment or invert the condition and you get an infinite loop that burns messages/credits and hangs the conversation until the channel times out.
- It creates **unreadable topic graphs** — the YAML shows a `goto` and the visual designer draws a jump line that's easy to miss in review.
- It **doesn't compose with generative orchestration**: the planner can decide to leave the topic mid-loop, so your guard state can be abandoned.
- No backoff — you can't wait between attempts inside a topic, so you hammer a failing endpoint.

**Preferred approach:** put the retry inside a **cloud flow**, which gives you the built-in *Retry Policy* (exponential, fixed interval, configurable count) on each connector/HTTP action, plus `Do until` with a timeout, plus proper `Delay` for `Retry-After`. The topic then calls the flow once and just handles success/failure. Retries are an infrastructure concern; keep them out of the conversation graph.

---

# Knowledge & grounding

## 12. Comparing SharePoint / Dataverse / Azure AI Search / public website / file upload

**SharePoint** — Queried live via Microsoft Graph / the M365 semantic index; no separate index build, so content is near-fresh. **Security trimming is honored** against the signed-in user. Best for existing document estates. Constraints: requires end-user authentication, supported file types only (Office docs, PDF, etc.), per-file size ceilings, and quality drops badly on scanned PDFs and huge documents. Citations link back to the file.

**Dataverse** — Queries selected tables/columns directly; respects Dataverse **security roles and row-level security** when the user is authenticated. Ideal for *structured* lookups ("what's the status of case 12345") where you want grounded natural-language answers over records without writing a query. Quality depends heavily on selecting the right columns and adding synonyms/glossary terms. Not good for long-form prose.

**Azure AI Search** — You own the index, so you control chunking, embeddings, hybrid/vector search, filters, and scale. Best for large or non-M365 corpora, and the only option when you need custom relevance tuning. **No automatic security trimming** — you must implement filters (e.g., pass a group claim into the query) yourself. Adds Azure cost and an operational pipeline.

**Public website** — Crawled/indexed through Bing, scoped to a limited number of URLs/domains. Only *publicly reachable* content; anything behind auth, a paywall, or heavy JS rendering is invisible. Freshness follows the public index, so recent page changes can lag. Zero security model. Great for product/marketing content, bad for policy documents.

**File upload** — Files are stored with the agent (in Dataverse) and indexed by Copilot Studio. Fastest path to a demo and fine for a small, stable set of documents. Downsides: no security trimming (anyone who can use the agent sees the content), documented file-count/size limits, and **no source of truth** — updating means re-uploading, and it won't transport cleanly through ALM.

*(Exact limits move release to release — I check the current Learn quotas page rather than quoting numbers from memory.)*

My default decision rule: structured data → Dataverse; existing M365 documents with permissions → SharePoint (+ user auth); large/bespoke/external corpus or custom relevance → Azure AI Search; public marketing content → website; throwaway pilots only → file upload.

---

## 13. How SharePoint security trimming actually works — and where it breaks

Trimming is enforced by **Microsoft Graph on behalf of the signed-in end user**. Copilot Studio doesn't hold a copy of the content or maintain a shadow ACL; it issues the search in the user's context, so Graph returns only items that user can already open. Permissions are therefore always current — revoke access in SharePoint and the next turn reflects it, with no re-index delay.

**Prerequisites:** the agent must use **Authenticate with Microsoft** or **manual Entra authentication with the required Graph scopes**, and the user must actually be signed in for that turn. On an anonymous channel, SharePoint knowledge simply returns nothing — which users report as "the bot is broken" and makers misdiagnose as an indexing problem. First thing I check is `System.User.IsLoggedIn`.

**Where it gets dangerous:**
- **Autonomous / triggered runs** have no interactive user. The agent executes under a **maker-configured connection or service identity**, so trimming is evaluated against *that* identity. If that account is over-privileged (a global admin's connection, say), the agent can surface content to a requester who should never see it — a real data-exfiltration path. Mitigation: dedicated least-privilege service principal, tightly scoped site, and an explicit authorization check before returning content.
- **Sharing the agent broadly** doesn't grant content access, but it does mean any grounding you do with a *shared* connection is not user-scoped.
- **Mixed sources**: an unsecured file-upload or website source sitting alongside SharePoint can leak the same content without trimming. Audit all sources, not just the SharePoint one.

---

## 14. Tenant graph grounding vs. adding SharePoint as a knowledge source

**Tenant graph grounding** lets the agent ground on the **Microsoft 365 semantic index across the tenant** — the user's mail, chats, meetings, and files as surfaced by the same index Microsoft 365 Copilot uses — rather than on a hand-picked set of sites. It's enabled at the agent level and, because it rides the M365 Copilot substrate, it generally requires the **end users to be Microsoft 365 Copilot licensed** and is enforced with full user-level permission trimming.

**Choose tenant graph grounding when:** you want a genuinely personal assistant ("what did my manager say about the Q3 budget?"), the relevant content is spread across the tenant and can't be enumerated as sites, and your users already have M365 Copilot.

**Choose explicit SharePoint knowledge when:** you need a **bounded, auditable scope** (this policy library and nothing else), your users are *not* M365 Copilot licensed, you need deterministic citations from a known corpus, or the agent is externally facing. Bounded scope also gives dramatically better precision — the semantic index over an entire tenant introduces noise for narrow use cases.

I usually pair them: tenant grounding for the "help me with my work" surface, scoped SharePoint for the "what does policy say" surface, with instructions telling the planner which to prefer per question type.

---

## 15. Diagnosing hallucinated answers from a website source

I work through it in this order:

1. **Confirm it's actually ungrounded.** Check whether the response carries **citations**. No citation = the model answered from parametric knowledge, which is a configuration problem, not a model problem.
2. **Turn off general knowledge.** In the Generative answers node / agent generative AI settings, disable *"Allow the AI to use its own general knowledge."* This is the single biggest cause. With it on, the model happily fills gaps.
3. **Raise the content moderation level to High.** Moderation directly controls how strictly responses must stay anchored to retrieved content; High = fewer, more conservative, more grounded answers. Low = chattier and more prone to drift. I'll trade recall for precision on anything policy- or compliance-adjacent.
4. **Verify the content is actually retrievable.** Is the page public? Is it JS-rendered (crawler sees an empty shell)? Is it within the URL scope you configured? I test by searching a distinctive phrase from the page — if retrieval returns nothing, no amount of prompt tuning helps.
5. **Tighten source scope.** Broad domains pull in blog posts and archived pages that contradict current policy. Narrow to specific paths, and remove sources that duplicate/conflict.
6. **Add explicit instructions**: *"Answer only from the provided sources. If the sources do not contain the answer, say you don't have that information and offer to connect the user to support. Never infer pricing, dates, or eligibility."*
7. **Fix the content, not the prompt.** If the source page is genuinely ambiguous, the fastest durable fix is editing the source or moving that topic to a curated file/Azure AI Search index with clean chunking.

Then I re-run the failing utterances as a saved test set so the fix is verified, not assumed.

---

## 16. Forcing citations and suppressing ungrounded responses

**Citations:** the Generative answers node has settings for including in-line citations / source links; enable them and make sure the source type supports link-back (website and SharePoint do well; "data from a variable" won't produce a real link unless you construct one). I also add an instruction like *"Always include the source link for any factual claim."* On channels that strip rich formatting, I render citations myself into an adaptive card from the node's output.

**Suppressing ungrounded answers** — layered:
1. Disable *"Allow the AI to use its own general knowledge."*
2. Set **moderation to High**.
3. Use the Generative answers node's **"no answer found" branch**. The node exposes a path for when nothing grounded is returned — route that to a deterministic message, a deflection topic, or escalation rather than letting it fall through.
4. Add a post-check: if the answer variable is blank **or** the citation collection is empty, don't send the generated text at all — send the fallback. `If(IsBlank(Topic.Answer) || CountRows(Topic.Citations) = 0, …)`.
5. Instruction-level backstop: *"If you cannot cite a source, respond exactly: 'I don't have that information.'"*

The combination matters — any single control on its own leaks.

---

# Actions, connectors, flows, MCP

## 17. Agent flows vs. Power Automate cloud flows

**Agent flows** are authored inside Copilot Studio and are purpose-built for agent work: deterministic, step-by-step business logic that can include **AI-powered nodes** (invoke a prompt, extract/classify with a model) alongside connector actions. They're designed for low-latency, in-conversation execution, are managed with the agent, and are metered against **Copilot Credits / the agent's billing model**. They're the right choice for "do this reliable sequence as part of answering the user."

**Cloud flows** are full Power Automate: the entire trigger catalogue (scheduled, Dataverse events, HTTP request, email arrival), the full connector estate including on-prem gateway, robust error handling, scopes, `Do until`, retry policies, and child flows. Called from an agent via the **"When an agent calls a flow"** (formerly Power Virtual Agents) trigger and a matching **Respond to the agent** action. Billed under **Power Automate licensing** (premium connectors need premium licensing).

**Differences that matter in practice:**
- **Latency/timeout:** a flow called synchronously from a conversation must respond within a short window (roughly a couple of minutes; the practical UX ceiling is far lower). Long-running work must be made asynchronous — kick it off, return a ticket, notify later.
- **Error surfacing:** a failed cloud flow does **not** automatically produce a nice message. You must catch failures with `Configure run after` / scopes and return a structured `Success`/`ErrorMessage` output, then branch on it in the topic. Otherwise the user gets a generic error and you debug in the flow run history.
- **Typing:** cloud flow agent-trigger inputs/outputs are limited to primitives and simple arrays; agent flows handle richer shapes more naturally.
- **ALM:** both should live in the same solution as the agent, with connection references.

---

## 18. Connection references vs. end-user connections

**Connection references** are solution components that decouple a flow/action from a specific **connection** (credential). The flow binds to the reference; the reference is bound to an actual connection **per environment**. This is what makes ALM possible — you export a solution with a connection reference, and in the target environment an admin points it at a prod connection without editing the flow. Without them you get "flows arrive turned off with broken connections" on every deployment.

**Who the connection runs as** is the security question. For a connector action in Copilot Studio you choose between:

- **Agent author / maker authentication** — every user's request executes under the **maker's credentials**. Simple, works anonymously, but it's impersonation: the agent can read and write anything the maker can, and the audit log shows the maker, not the user. Only acceptable for genuinely shared, non-sensitive data (a public product catalogue) with authorization enforced inside the action.
- **End-user authentication** — each user is prompted in-conversation to create their own connection/consent. Actions then execute as that user, so permissions and audit trails are correct. Costs a consent step, requires the agent to be authenticated, and won't work on anonymous channels or in autonomous runs.

My default is end-user authentication for anything touching user-specific or regulated data, maker connections only for read-only public data, and a **dedicated least-privilege service account** (never a personal admin account) whenever a shared connection is unavoidable — because a maker leaving the company otherwise takes production down.

---

## 19. Exposing an MCP server as a tool

**Model Context Protocol** lets you publish a server that advertises **tools** (and resources/prompts) which the agent discovers dynamically. In Copilot Studio you add the MCP server as a tool — in practice via a **custom connector definition that points at the MCP endpoint** (streamable HTTP / SSE transport), after which Copilot Studio calls the server's discovery methods and surfaces each advertised tool to the orchestrator with its name, description, and JSON-Schema input contract.

**Why it matters:** the tool list is **dynamic**. Add a tool to your MCP server and it appears in the agent without re-authoring topics or re-publishing a connector — a big deal versus maintaining N hand-built connector actions. It also gives you one integration surface reusable across agents and other MCP hosts.

**Auth patterns supported:** no auth (dev only), **API key**, and **OAuth 2.0 / Microsoft Entra ID**, including flowing the end user's identity so the server can act on their behalf. Entra + end-user auth is the pattern I'd push for anything internal.

**Things I'd raise as risks:** MCP tools are only as safe as their descriptions — a malicious or sloppy server can inject instructions into the planner's context (tool-description injection), so I only register servers we control or have reviewed; MCP endpoints must be covered by DLP/network policy; and dynamic tool lists make regression testing harder, so I pin and monitor the advertised tool set.

---

## 20. Handling 401 / 429-with-Retry-After / 500 from an HTTP node

**In the topic**, the HTTP request node lets you capture the **response body, status code, and headers**, and lets you choose whether an error is thrown or handled. My pattern:

1. Configure the node to **not** fail the topic on error, so I get the status code into a variable.
2. **Switch on the status class:**
   - **401/403** — the token is expired or scopes are wrong. Don't retry blindly. Clear auth state and trigger a re-sign-in (`Sign out` + redirect to the sign-in flow), or if it's a maker connection, log it and tell the user the integration needs attention. Message: *"I need you to sign in again to do that."*
   - **429** — read `Retry-After`. Topics can't delay, so this is the case I hand to a **cloud flow**, which can `Delay` for the returned interval and use a retry policy. If it's persistent, back off to a queued/async pattern.
   - **5xx** — transient; retry 2–3 times with backoff **in a flow**, then fail gracefully.
   - **404** — usually a data problem, not an outage: tell the user the record wasn't found and offer alternatives.
3. **Never surface the raw payload.** Raw bodies leak internal hostnames, stack traces, and sometimes tokens. I map to a friendly message plus a **correlation ID** I generate (`System.Conversation.Id` + timestamp) and show that to the user: *"Something went wrong on our side. Reference: ABC-123."*
4. **Log the detail out-of-band** — a fire-and-forget flow writing status, endpoint, correlation ID, `System.User.Id`, and the truncated error to Dataverse/Application Insights, so support can look up ABC-123.
5. Add a topic-level **`On Error`** system topic as the catch-all so unhandled failures still produce a controlled message and escalation path.

---

## 21. Passing complex objects into and out of a flow

The **"When an agent calls a flow"** trigger has historically only accepted **primitive types** (text, number, boolean, date) and simple arrays — no nested records. So for anything structured:

**Into the flow:** serialize in the topic and deserialize in the flow.
```powerappsfl
Set(Topic.PayloadJson, JSON(Topic.OrderRecord, JSONFormat.IndentFour))
```
Pass `Topic.PayloadJson` as a **text** input, then in the flow use **Parse JSON** with an explicit schema (generate it from a sample payload). From there every field is a first-class dynamic value.

**Out of the flow:** the `Respond to the agent` action returns primitives, so return a **JSON string** plus a couple of flat control fields — I always return `Success` (boolean) and `ErrorMessage` (text) alongside `ResultJson` (text), because that gives the topic a clean branch without parsing. In the topic:
```powerappsfl
Set(Topic.Result, ParseJSON(Topic.ResultJson))
```
then cast fields as in Q9, or `ForAll(Table(...))` for collections.

**Caveats to mention:** watch the size — very large JSON strings hurt latency and can hit payload limits, so page or return IDs rather than whole datasets; keep the schema versioned (add a `schemaVersion` field) because flow and topic deploy independently; and note that **agent flows** and newer runtime versions handle records/tables more directly, so check current capability before defaulting to the string-serialization workaround — but the JSON pattern remains the portable, always-works answer.

---

# Multi-agent & extensibility

## 22. Connected agents — context passing, results, and loop prevention

**Connected agents** let one agent register another as a callable tool. The parent's orchestrator sees the child by its **name and description**, exactly like any other tool, and delegates when the description matches the user's need.

**What's passed:** not the whole transcript. The parent sends the **task/instruction it generated** plus any **declared inputs**, along with enough conversational context for the child to act. The child then runs its **own orchestration** — its own instructions, topics, tools, and knowledge — and returns a **result/summary** (plus declared outputs) that the parent folds into its response. This is the key architectural benefit: the child's tools and knowledge don't pollute the parent's planner context, which keeps tool selection accurate as the solution grows.

**Identity and security:** the child runs with its own configuration; authentication/connections don't magically inherit, so I verify that end-user identity actually flows where I need it, and that a child with broad data access isn't callable from a weakly-secured parent.

**Preventing runaway delegation:**
- **Hub-and-spoke topology**, never mutual/circular connections. Agent A → B → A is the loop you must design out.
- **Sharply exclusive descriptions** so two children aren't both plausible.
- **Cap the depth** — I keep it to one, at most two, levels; deep chains destroy latency and make failures untraceable.
- **Latency/credit budget awareness** — each hop is another orchestration round trip and another metered set of calls.
- **Correlation logging** across parent and child using a shared conversation/correlation ID, otherwise debugging is guesswork.

---

## 23. Declarative agent vs. Copilot Studio agent vs. publishing into M365 Copilot

**Declarative agent (M365 Copilot):** a lightweight, manifest-defined agent that runs **on the Microsoft 365 Copilot orchestrator and model**. You supply instructions, knowledge scoping (SharePoint/Graph), starter prompts, and optionally API plugins/actions. Built in Copilot Studio's agent builder or VS Code/Teams Toolkit, shipped as an M365 app. **Requires users to be M365 Copilot licensed.** No custom conversation flow, no deterministic topics, no channels outside M365. Choose it when the job is "shape Copilot's behavior over our content" and users are already in Copilot chat.

**Full Copilot Studio agent:** your own orchestration, deterministic topics, Power Fx, flows, custom auth, multi-channel publishing (Teams, web, Direct Line, voice/telephony, Facebook, custom apps), its own analytics, its own billing (credits/messages). Choose it when you need **guaranteed deterministic paths** (compliance-scripted disclosures, transactional flows), **non-M365 audiences** (customer-facing web/IVR), or **users without M365 Copilot licenses**.

**Publishing a Copilot Studio agent into M365 Copilot** (as an agent/action available in Copilot chat): the right answer when the *logic* must be deterministic and custom, but the *entry point* should be where users already work. You get Copilot chat's surface plus your own topics/flows behind it.

My decision heuristic: licence coverage and determinism. No M365 Copilot licences, or hard compliance scripting, or external users → full Copilot Studio agent. Licensed internal users, content-shaping only → declarative. Licensed users needing custom transactions → Copilot Studio agent surfaced in M365 Copilot.

---

## 24. Autonomous agents and event triggers — identity and security implications

**Autonomous agents** act without a user in the conversation. They're started by **triggers**: a Dataverse row created/updated, a scheduled recurrence, an email arriving in a monitored mailbox, a Teams message, or an external event via a connector. The agent then runs its instructions and tools to completion and typically writes results somewhere (updates a record, sends a reply, posts to Teams).

**Identity is the crux.** There is no interactive user, so:
- Everything runs under **maker-configured connections / a service identity**, not the requester's.
- `System.User.*` and end-user authentication are effectively unavailable, so **knowledge security trimming is evaluated against the service identity** — a serious over-exposure vector (see Q13).
- Any action the agent can take, it can take **at machine speed and volume** with no human in the loop.

**How I'd govern it:**
- **Dedicated least-privilege service principal / service account** per agent, scoped to exactly the tables, sites, and mailboxes required — never a personal or admin account.
- **Environment isolation**: autonomous agents in their own environment with their own DLP policy, so the connector set is constrained.
- **DLP policies** classifying connectors, blocking anything that could exfiltrate (e.g., generic HTTP, personal storage connectors) unless explicitly justified.
- **Guardrails in the agent**: idempotency keys so a retriggered event doesn't double-post; volume caps; a "confidence low → create a task for a human instead of acting" branch.
- **Full auditability**: every action written to Dataverse/App Insights with the trigger ID, plus Purview audit for connector activity.
- **Rollback story** — how do we undo 500 wrong record updates? If there isn't one, the agent gets human approval steps.

---

# Authentication & security

## 25. No auth vs. Authenticate with Microsoft vs. manual authentication

**No authentication** — anonymous. Anyone with the link/embed talks to it. `System.User.*` is anonymous/ephemeral, no `AccessToken`, **SharePoint/Dataverse knowledge returns nothing**, and user-context connectors can't work. Valid only for public FAQ/deflection bots.

**Authenticate with Microsoft** — Copilot Studio manages the Entra app registration for you. Zero-config, gives you a signed-in Entra identity, `System.User.IsLoggedIn`, display name/email, working SharePoint knowledge trimming, and **SSO in Teams and M365 out of the box**. The trade-off: you don't control scopes, so you **don't get an arbitrary delegated Graph/API token** — it's for identity and first-party grounding, not for calling your own APIs on the user's behalf.

**Authenticate manually (Entra ID or generic OAuth 2.0)** — you supply your **own app registration**: client ID, client secret, token/authorization endpoints, and the **scopes** you need. This is the only option that populates a usable **`System.User.AccessToken`** for downstream API calls, and the only way to integrate a non-Microsoft IdP (Okta, Auth0, Google) via generic OAuth 2.

**Manual Entra setup checklist:**
- App registration → **Web** platform → **redirect URI `https://token.botframework.com/.auth/web/redirect`** (use the correct regional token service host, e.g. the `europe.` variant, if your environment is in that geography).
- **Client secret** (track expiry — expired secrets are a top-3 production outage cause; use a calendar reminder and, where supported, certificate credentials).
- **API permissions**: delegated scopes for what you'll call (`User.Read`, `Mail.Read`, or `api://<your-api>/access_as_user`), with **admin consent** granted to avoid per-user consent prompts.
- In Copilot Studio: paste client ID/secret, token exchange URL (for SSO), and the **space-separated scope list**.
- Re-publish; auth settings are **per environment** and do not transport in a solution.

---

## 26. Teams SSO — and what breaks on a custom website via Direct Line

**Teams SSO** removes the sign-in card so the user is silently authenticated with their Teams identity. Setup:

1. **Manual authentication** with your own Entra app.
2. Expose an API on that app: **Application ID URI** in the form `api://<region-or-host>/botid-<your-bot-app-id>`, with a scope such as `access_as_user`.
3. Add the **Teams client application IDs** as **authorized client applications** (pre-authorized) on that scope — `1fec8e78-bce4-4aaf-ab1b-5451cc387264` (Teams desktop/mobile) and `5e3ce6c0-2b1f-4285-8d4b-75ee78787346` (Teams web).
4. In Copilot Studio, enable SSO and supply the **token exchange URL** (the Application ID URI + scope).
5. Grant **admin consent** for the delegated permissions.
6. Re-publish the agent and re-install the Teams app.

**What breaks on a custom website via Direct Line:** Teams SSO simply doesn't apply — there's no Teams host to broker the token. On the web the user gets an **OAuth sign-in card and a popup/redirect flow**. Common failures: third-party cookie blocking and popup blockers killing the flow; embedding in an iframe where the OAuth provider refuses to frame; and losing the conversation on redirect if you don't persist the Direct Line conversation ID.

The production pattern: generate the Direct Line token **server-side** (never expose the Direct Line secret in the browser), keep the secret in Key Vault, use the **Web channel security / "secure site" enhanced authentication** options so anonymous users can't hijack the endpoint, and if the site already authenticates the user, pass their identity into the conversation (e.g., via a signed token / user ID on token generation) rather than making them sign in twice.

---

## 27. Calling Microsoft Graph on behalf of the signed-in user

1. **Configure manual authentication** with your own Entra app.
2. **Request delegated Graph scopes** on that app (`User.Read`, `Mail.Read`, `Calendars.ReadWrite`, …) and grant **admin consent**. In Copilot Studio, list those scopes in the auth configuration so the issued token has the right **audience (`https://graph.microsoft.com`)**.
3. The user signs in (via the `Sign in` node or automatic sign-in at conversation start). Check `System.User.IsLoggedIn`.
4. Call Graph from an **HTTP request node**:
   - URL: `https://graph.microsoft.com/v1.0/me/messages?$top=5`
   - Header: `Authorization: Bearer {System.User.AccessToken}`
   - Parse the response with `ParseJSON` (Q9).

**The nuance interviewers look for:** an access token has **one audience**. If your app's token is minted for *your own* API (`api://…/access_as_user`) — which is what you configure for Teams SSO — that token is **not valid for Graph**. In that case you need the **on-behalf-of flow**: send the user token to your own API (or a flow/Azure Function), which exchanges it with Entra for a Graph token, calls Graph, and returns the result. Copilot Studio itself doesn't perform OBO for you.

**Consent** is handled at first sign-in; pre-granting admin consent avoids per-user prompts and is essential for a smooth rollout. Also: never log the token, never put it in an adaptive card, and remember `System.User.AccessToken` is empty for anonymous users and in autonomous runs.

---

## 28. DLP, connector classification, CMK, and data residency for regulated data

**Data Loss Prevention** is the primary control. Power Platform DLP policies sort connectors into **Business**, **Non-Business**, and **Blocked** groups; a flow or agent action **may not combine connectors from different groups**. So the classic exfiltration path (Dataverse → Twitter/personal OneDrive) is structurally prevented rather than policy-documented. I'd put SharePoint/Dataverse/Office 365 in Business, block generic HTTP and consumer storage/social connectors outright in regulated environments, and use **connector action-level rules** to allow, say, HTTP GET to an approved host while blocking POST. Newer **agent-aware DLP/governance controls** extend this to knowledge sources, autonomous agents, and which models/plugins can be used — I'd check the current admin center capabilities rather than assume.

**Environment strategy** is inseparable from DLP: separate dev/test/prod environments, prod locked down, **environment groups + rules** to enforce policy at scale, and no personal "developer" environments touching regulated data.

**Encryption and residency:**
- **Customer-managed keys (CMK)** on the environment's Dataverse database, keys in Azure Key Vault under your control, with the understood consequences (key revocation makes the environment inaccessible; restore/support paths change).
- **Data residency / geo** — choose the environment region deliberately; know which processing may still occur outside the geo (some AI features and cross-geo grounding), and check the current **data movement / EU Data Boundary** commitments for the specific features you enable.
- **Tenant isolation** to stop cross-tenant connector use, **Customer Lockbox** for support access approval.

**Operational:** Purview audit logging for agent and connector activity, conversation transcripts retention policy (transcripts land in Dataverse and *contain user PII* — set retention and access controls deliberately), least-privilege security roles, and Direct Line secrets in Key Vault with rotation.

---

# ALM, testing, observability

## 29. Full ALM walkthrough — and what does *not* transport

**Setup:** create the agent **inside a solution** from the start (solution-aware). Retrofitting an agent into a solution later is possible but painful. Use a **dedicated publisher and prefix**. Everything related goes in the same solution: agent, topics, cloud/agent flows, custom connectors, Dataverse tables, environment variables, connection references, AI prompts.

**Pipeline:** Dev (unmanaged) → export → **import as managed** into Test → then Prod, ideally via **Power Platform Pipelines** or Azure DevOps/GitHub Actions with the Power Platform Build Tools. Managed in downstream environments is non-negotiable: it prevents drift and makes uninstall/rollback possible.

**Parameterize everything environment-specific:**
- **Environment variables** for SharePoint site URLs, API base URLs, Dataverse record GUIDs, feature toggles. Set values per environment at import time — never hard-code a URL in a topic.
- **Connection references** for every connector so the target environment binds its own credentials.

**What does *not* transport (the part interviewers actually probe):**
- **Published state.** The agent arrives unpublished — you must **publish in the target environment**. Every deployment.
- **Connections.** References transport; the actual credential doesn't. Someone must authorize connections in the target, or flows/actions arrive broken/off.
- **Authentication configuration.** Entra client ID/secret, scopes, and auth mode are environment-scoped and set post-import.
- **Channel configuration.** Teams app publication/approval, Direct Line secrets, web channel security settings, telephony config — all re-done per environment.
- **Knowledge indexes and, for some source types, the content.** Website sources re-crawl; uploaded files and index state don't reliably carry; Azure AI Search/SharePoint endpoints must be repointed via environment variables.
- **Analytics/telemetry history, conversation transcripts, and test datasets.**
- **Sharing, security roles, and Dataverse data** (data is not a solution component — move it with a data migration tool or configuration-data package).
- **Solution-aware ≠ everything**: anything created outside the solution (a flow made in "My flows", a connector in the default environment) is invisible to the export.

**Post-deploy checklist I run:** connections authorized → environment variable values set → agent published → auth configured → channels enabled → smoke test suite green.

---

## 30. Testing and monitoring at scale

**Copilot Studio Kit** (the Power CAT open-source accelerator) is the main answer for scale. It's a Dataverse-based solution that lets you define **test sets** — an utterance, the expected topic/response, and expected behavior — and run them in **bulk against a deployed agent**, rather than typing into the test pane. It supports **AI-assisted evaluation** of generative answers (semantic similarity / LLM-as-judge rather than string match, which is essential once responses are generated), regression comparison across versions, and results stored in Dataverse so you can trend them. I wire it into the release pipeline as a quality gate: a generative-answer regression should block promotion the same way a failing unit test does.

**Test pane vs. real channel.** The pane is fast but misleading: it runs as *you* (so auth and permission trimming look better than reality), it renders adaptive cards differently from Teams and web, it doesn't reproduce channel-specific text truncation or attachment behavior, and it hides real network latency. Anything auth-, card-, or latency-sensitive gets tested in the actual channel before sign-off.

**Observability.** Enable **Application Insights** on the agent for conversation traces, custom events, exceptions, and latency — then you can query with KQL, alert on error-rate spikes, and correlate a user's complaint to a conversation ID. **Conversation transcripts** land in Dataverse for detailed replay (with a retention/PII policy attached). I also emit my own correlation ID and structured events from flows so an end-to-end trace spans agent → flow → API.

**Metrics I actually act on**, from the analytics dashboards:
- **Resolution rate** and **escalation rate**, broken down **per topic** — the aggregate number tells you nothing; the per-topic breakdown tells you which three topics to fix.
- **Abandonment rate** — where users drop mid-conversation, usually a too-long form or a failing action.
- **CSAT** (the end-of-conversation survey) correlated with topic, to separate "bad answer" from "correct answer the user didn't like."
- **Unrecognized/unmatched utterances** (session details / transcript mining) — the single best backlog for new topics and knowledge gaps.
- **Generative answer citation rate** — a drop signals a knowledge source that stopped indexing.
- **Latency p95** on flow-backed topics, and **credit/message consumption** trends for cost control.

The discipline I'd emphasize: agents drift because *content* and *user language* change, not because code changed. So monitoring and a scheduled test run are ongoing operations, not a launch-week activity.

---

Want me to save this as a markdown study file, or drill into any of these with follow-up "second-round" probing questions?
