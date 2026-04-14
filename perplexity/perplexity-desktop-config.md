# Paste following prompt to Settings -> Personalization or use as a system message in API requests

```
Summarize my question in one line. Cite source dates (YYYY-MM-DD).
PRIORITY: Precision > Diplomacy > Completeness. One truth > ten "maybe"s.
Tag claims: [FACT:official], [FACT:3rd-party], [FACT:analyst], [INFERENCE], [UNKNOWN].
Distinguish marketing claims from contractual/technical specs. Flag marketing figures.
BEFORE answering, ALWAYS state:
- ASSUMPTIONS: what you assumed about my context (version, platform, modded/vanilla, OS, environment)
- MISSING: what you would need for a precise answer but I did not provide
- SCOPE: current version, latest stable, or something else?
If MISSING has 2+ items, answer conditionally: "If X then Y, if A then B".
Challenge my assumptions - find logic gaps proactively.
Reveal context I may lack about the topic.
Verify source recency: if search result older than 6 months, flag [STALE].
When answering software/games/APIs: check version, changelog, deprecation status.
State conditions precisely: "X causes Y when Z" - never unqualified "might".
State criticism directly. No hedging without conditions.
Answer my specific situation, not general theory, unless I ask for general.
BAD: "Gas wants to condense"
GOOD: "Phase equilibrium - no net condensation without pressure/temp change"
If your search results conflict, show both claims with dates - don't pick one silently.
When you cannot ask me clarifying questions, list them under MISSING and answer each scenario.
For PowerShell/CLI commands: verify against current docs, not training memory. md>pdf.
```

## What each section does

The template has ten distinct rules. Each one catches a specific failure mode. Here's why each one matters.

### "Summarize my question in one line"

Forces the model to prove it understood the question before committing to an answer. If the one-line summary is wrong, you spot it immediately and correct before reading the full response. Without this, you sometimes get a perfectly-written answer to the wrong question. This is also useful if you paste your response to other model. Without it it wouldn't know the context of the answer.

### "PRIORITY: Precision > Diplomacy > Completeness. One truth > ten maybes"

This is the most important line. It sets the model's utility function. Default LLM training optimizes for helpfulness and completeness - the model would rather give you ten hedged possibilities than one committed truth with conditions. This line inverts that ordering. The model will now refuse to answer rather than guess, and will answer short rather than pad.

### "Tag claims: [FACT:official], [FACT:3rd-party], [FACT:analyst], [INFERENCE], [UNKNOWN]"

Same as the compact version but with `[FACT:analyst]` added. That tag matters when researching market data, pricing trends, or industry stats where the "source" is an analyst firm (Gartner, Forrester, IDC). Analyst reports are not primary sources but they're also not random third parties. Giving them their own tag lets you weight them correctly.

This tagging technique is critical for treating the response as research data feed to other LLM. Without it it wouldn't have the ability to differ facts from inference (thinking/hallucinations).

### "Distinguish marketing claims from contractual/technical specs. Flag marketing figures"

This catches a specific class of failure that gets people in trouble with software vendors. A product page says "99.99% uptime" - that's a marketing figure. The actual SLA document says "99.5% uptime, measured monthly, excluding scheduled maintenance" - that's the technical spec. Both are "from the vendor's website." Only one is binding. The tag forces the model to say which is which.

### "BEFORE answering, state ASSUMPTIONS / MISSING / SCOPE"

This is the core discipline. Before any answer, the model has to commit in writing to:

- **ASSUMPTIONS**: what it filled in on your behalf. This is where silent assumptions come to the surface - assumed OS, assumed version, assumed you're on the free tier, assumed you mean the latest release. If any assumption is wrong, you know immediately.
- **MISSING**: what the model would need to answer precisely, but you didn't provide. This is an anti-hallucination mechanism - instead of inventing the missing piece, the model lists it.
- **SCOPE**: a commitment to what version/time window this answer covers. Protects against "the answer was right in 2022" problems.

The three together form a "answer contract" before the answer starts.

### "If MISSING has 2+ items, answer conditionally"

This is the rule that keeps answers useful when inputs are incomplete. Instead of picking one interpretation and answering for that, the model branches: "If you're on Windows, do X. If you're on Linux, do Y." The user picks the branch that matches their real situation. Prevents the "I assumed X and answered for X, but you're on Y" failure.

### "Challenge my assumptions. Reveal context I may lack"

Licenses the model to push back. Standard LLM behavior is to answer the question exactly as asked, even if the question is malformed. This rule grants explicit permission to say "you're asking about X, but the real constraint is Y, and here's why."

The second half - "reveal context I may lack" - is a nudge toward proactive information sharing. The model knows things you don't, and without this rule it defaults to only answering the literal question. With this rule, it surfaces relevant adjacent facts.

### "Verify source recency: flag >6 months as [STALE]"

API docs, pricing pages, and software behavior change quickly. A perfectly correct 2024 answer might be dead wrong in 2026. The `[STALE]` tag is a forcing function for the model to actually check the date on its source, not just the content.

For fast-moving topics (API pricing, security advisories, library versions), 6 months is generous. For stable topics (physics, history, established standards), 6 months is too strict. Adjust to your domain.

### "For software/games/APIs: check version, changelog, deprecation status"

Specific protection against the most common failure mode for technical questions. The model might know how a library worked in v1.x but not that the API was removed in v2.0. This rule forces version-awareness into the answer.

### "State conditions precisely: X causes Y when Z. Never unqualified 'might'"

Attacks weasel words directly. "Might work" is a useless answer - you can't act on it. Either say "X causes Y under conditions Z, otherwise not" or say "[UNKNOWN]." The unconditional "might" is the worst of both worlds: it commits to nothing but sounds like an answer.

### "State criticism directly. No hedging without conditions"

Models are trained to be polite, which often means softening criticism. "You might want to consider that X could potentially be an issue" is unusable. "X is broken because of Y" is useful. This rule turns the politeness down so you can actually act on the feedback.

### "Answer my specific situation, not general theory"

Another anti-padding rule. Models often answer with the general case first, then optionally mention your specific case. This inverts the order and removes the general case unless you explicitly asked for it.

### "If search results conflict, show both claims with dates - don't pick one silently"

Critical for research. When sources disagree, the safe default is to disclose the disagreement, not resolve it silently. The model picks silently based on training-time patterns, not on which source is actually more authoritative for your context. Forcing explicit disagreement lets you make the call yourself.

### "Cannot ask clarifying questions - list under MISSING and answer each scenario"

This is a workaround for non-interactive contexts. In a chat you can go back and forth. In an API call you can't. If the question is ambiguous, the model would normally need to ask "which did you mean?" - but it can't, because the API is one-shot. This rule tells the model to list the clarification it would have asked, and then answer both branches.

### "PowerShell/CLI commands: verify against current docs, not training memory. md>pdf"

PowerShell in particular evolves fast and has multiple versions (5.1 vs 7.x) with different syntax. Cmd flags get deprecated, module APIs change. The rule "verify against current docs" forces the model to actually search rather than rely on pattern-matched memory of what a PowerShell script should look like.

"md>pdf" means "Markdown output is better than PDF output" - a stylistic preference for how the model formats its responses. Adjust or remove based on your preference.

## How to customize

This template is opinionated toward technical research. If your use case is different, these are the parts you might want to change:

- **Drop** "For PowerShell/CLI commands..." if you don't use PowerShell
- **Drop** the BAD/GOOD physics example if you don't care about physical systems
- **Add** domain-specific rules - for legal research, add "distinguish dicta from holdings." For medical, add "distinguish trial phase from approved indication." For finance, add "distinguish GAAP from non-GAAP figures."
- **Change** the 6-month staleness threshold to match your domain's velocity
- **Replace** "Precision > Diplomacy > Completeness" with a different priority ordering if that doesn't match your values

The structure - summarize question, state ASSUMPTIONS/MISSING/SCOPE, tag everything, challenge assumptions - is the core. Everything else is adaptation.

## Expected output format

With this system prompt, a Perplexity answer looks something like this:

```
Question summary: How does Microsoft Graph API handle $filter on nested navigation properties?

ASSUMPTIONS: You're on v1.0 stable (not beta), Exchange Online (not on-prem), Application permissions.
MISSING: Your specific property - from/sender/toRecipients behave differently. Your scale - works differently below vs above 10K messages. Whether you use $orderby in the same query.
SCOPE: Graph API v1.0 as of 2026-03 (most recent docs I found).

Conditional answers:

If the property is `from/emailAddress/address`:
- [FACT:official] Only `eq` operator supported. (Source dated 2025-09, [STALE])
- [FACT:3rd-party] `contains` / `startswith` / `endswith` return HTTP 400. Stack Overflow confirmation 2024-11 [STALE].
- [UNKNOWN] Whether `in` operator works - no documented examples either way.

If you combine with $orderby on a different field:
- [FACT:official] Documented rule: $orderby properties must appear in $filter in the same order, before other filter properties.
- [INFERENCE] Violating this causes HTTP 400 with code `InefficientFilter`. I did not find explicit confirmation of the exact error code, but the ordering rule implies this failure mode.

MISSING from my answer:
- Whether sender/toRecipients behave identically to from (probably but not confirmed)
- Whether the restriction changes for large mailboxes
- Whether beta endpoint has relaxed rules

Challenge: You asked about "nested navigation properties" generically. If you actually mean collection properties like toRecipients (which are arrays, not single objects), the answer changes - you need `any` / `all` lambda operators, which have their own quirks. Clarify if needed.
```

This is longer than a default answer. It is also much harder to act on wrong, because every commitment is tagged and every gap is visible.

## When this template hurts you

The template is bad for:

- **Casual chat** - it adds heavy structure to questions that don't need it
- **Creative writing** - tagging breaks narrative flow
- **Brainstorming** - you want loose exploratory output, not committed claims
- **Quick lookups** ("What's the capital of Portugal") - overkill

For those, disable the custom prompt or use a lighter variant.

The template is good for:

- **Technical research** (APIs, libraries, tools)
- **Debugging sessions** where a confident wrong answer burns hours
- **Vendor research** (comparing tools, checking pricing, reading docs)
- **Due diligence** work
- **Any agent workflow** where the LLM output feeds downstream automation

## The tradeoff

You trade output verbosity and some polish for honesty and usability. Answers get longer in token count, but shorter in what you actually act on - because instead of accepting the full prose, you scan for `[FACT:*]` tags and skip the `[UNKNOWN]` sections.

Practical result: you spend less time chasing down confident lies, more time on real work.

## See also

- Main guide: `README.md` in this repo
- Copy-paste templates: bottom of README
