# This technique in general is a game changer when it comes to use Perplexity as a knowledge base for you or your agents.

## Why it matters

By default Perplexity answers with wall of text.
`You` (using it as a gui app) or `your agent` (using it through api or MCP) don't know what was a `fact` (fact official, fact 3rd party) taken from the source and what was `made up` by Perplexity (inference).

My experience is also that it stops most of its hallucinations because it just flags the info as `UNKNOWN`.

# THE FIX

Minimal version if you already have a system prompt:

```
State ASSUMPTIONS, MISSING, SCOPE before answering. Tag: [FACT:official], [FACT:3rd-party], [INFERENCE], [UNKNOWN]. Cite dates (YYYY-MM-DD). Flag sources >6mo as [STALE]. Challenge question assumptions. Precision > hedging.
```

Full version (1499/1500 characters used)

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

## The difference

With both of these your answers will be properly tagged and will have completely different meaning than without it.

Without these Perplexity invents some information in answers (for example when asked about Claude Code prices - it made them up).

With this system prompt Perplexity will:
- use tagging [FACT:official], [FACT:3rd-party], [FACT:analyst], [INFERENCE], [UNKNOWN]
- add [ASSUMPTIONS] [MISSING] [SCOPE] in the beginning of the body of the answer

## What each section does

### "Summarize my question in one line"

This is because I often copy&paste answers from Perplexity into Claude Code. The problem was I asked Perplexity something and it came up with an answer.
But the answer didn't have the question in it so when pasted into Claude Code it didn't have proper context.
This short summarization gives your AI agent proper context of the answer.

### "Cite source dates (YYYY-MM-DD)"

This is important. For example the tool will extract the date of the article it bases it answer on.
It helps to distinguish current valuable information from old and obsolete one.

### "PRIORITY: Precision > Diplomacy > Completeness. One truth > ten maybes"

It directs the model to give precise answers. No "maybe" or "should".

### "Tag claims: [FACT:official], [FACT:3rd-party], [FACT:analyst], [INFERENCE], [UNKNOWN]"

This is critical for treating the response as research data feed to other LLM. Without it it wouldn't have the ability to differ facts from inference (thinking/hallucinations).

### "Distinguish marketing claims from contractual/technical specs. Flag marketing figures"

In general deep research did that properly, but quick answers had many hallucinations and did not distinguish facts or opinions properly.
This gives better answers.

### "BEFORE answering, state ASSUMPTIONS / MISSING / SCOPE"

This is the core discipline. Before any answer, the model has to commit in writing to:

- **ASSUMPTIONS**: This is where silent assumptions come to the surface - things you didn't tell it. If any assumption is wrong, you know immediately.
- **MISSING**: What the model would need to answer more precisely, but you didn't provide. This is an anti-hallucination mechanism - instead of inventing the missing piece, the model lists it.
- **SCOPE**: Another "dimension" of assumptions and missing - the broader scope or planned answer.

The three together form a "answer contract" before the answer starts.

### "If MISSING has 2+ items, answer conditionally"

This is the rule that keeps answers useful when inputs are incomplete. Instead of picking one interpretation and answering for that, the model branches: "If you're on Windows, do X. If you're on Linux, do Y." The user picks the branch that matches their real situation. Prevents the "I assumed X and answered for X, but you're on Y" failure.

### "Challenge my assumptions"

Allows the model to push back. This rule grants explicit permission to say "you're asking about X, but the real constraint is Y, and here's why."

### "Reveal context I may lack"

It's a nudge toward proactive information sharing. If the model knows things you don't it will tell you.

### "Verify source recency: flag >6 months as [STALE]"

The model will try to tell you when the information is old.

### "When answering software/games/APIs: check version, changelog, deprecation status"

This is to reinforce the model to further test the source data for potential deprecation.

### "State conditions precisely: X causes Y when Z. Never unqualified 'might'"

Attacks some words directly. "Might work" is a useless answer - you can't act on it.
Either say "X causes Y under conditions Z, otherwise not" or say "[UNKNOWN]."
The unconditional "might" is the worst of both worlds: it commits to nothing but sounds like an answer.

### "State criticism directly. No hedging without conditions"

Models are trained to be polite, which often means softening criticism.
We don't want answers like "You might want to consider that X could potentially be an issue".
We need clear statements. So we ask the model to be critical.
My observation is that the less polite the model is the more honest it is.

### "Answer my specific situation, not general theory"

Another anti-padding rule. Models often answer with the general case first, then optionally mention your specific case.
This inverts the order and removes the general case unless you explicitly asked for it.
Or it shoud say "[UNKNOWN]".

### "If search results conflict, show both claims with dates - don't pick one silently"

Critical for research. When sources disagree, the default is now to disclose the disagreement, not to pick one silently.
Forcing explicit disagreement lets you make the call yourself.

### "When you cannot ask clarifying questions - list under MISSING and answer each scenario"

This is a workaround for non-interactive contexts. In a chat you can go back and forth. In an API call you can't.
If the question is ambiguous, the model would normally need to ask "which did you mean?" - but it can't, because the API is one-shot.
This rule tells the model to list the clarification it would have asked, and then answer both branches.

### "PowerShell/CLI commands: verify against current docs, not training memory."

Just to avoid inference and hallucinations. It forces the model to lookup answers in situations it would respond with data from training.
Perplexity is by design forced to lookup information before answering but that changes what it looks up.

### md>pdf

A filler to tell the model that we want to download markdown files not pdfs.

## Expected output format

With this system prompt, a Perplexity answer looks something like this (when you ask it about Microsoft 365 licensing):

```
ASSUMPTIONS: Exchange Online shared mailbox, properly converted from user → shared, standard Microsoft 365 cloud-only, no holds/compliance features in play unless stated.  
MISSING: 1) Is Litigation/Retention/Legal Hold or eDiscovery in use on this mailbox? 2) Is Online Archive enabled? 3) Is this a converted user mailbox or “native” shared?  
SCOPE: Latest Microsoft 365 / Exchange Online behavior as of 2026-03.

In one line: If you remove EP2 from an 80 GB shared mailbox, quota drops to 50 GB, mailbox stays shared but becomes over‑quota and effectively stops normal mail flow. [techcommunity.microsoft](https://techcommunity.microsoft.com/blog/exchange/correcting-shared-mailbox-provisioning-and-sizing/607991)

## What exactly happens at 80 GB when you remove EP2

When you unassign the EP2 (Exchange Online Plan 2) license from that shared mailbox, Exchange enforces the unlicensed shared mailbox quota of 50 GB instead of the licensed 100 GB. [FACT:official] The mailbox data is not deleted; it remains at 80 GB, but because actual size exceeds the now‑effective 50 GB quota, the mailbox is treated as over‑quota. [FACT:official] [blog.edbmails](https://blog.edbmails.com/correcting-shared-mailbox-provisioning-and-sizing.html)

In an over‑quota state, send and receive operations for that mailbox fail: incoming messages start bouncing with “mailbox is full” (5.2.2) and the mailbox cannot send new mail until size is reduced below quota or a suitable license is re‑applied. [FACT:official] This is a hard behavior, not “maybe”: an 80 GB mailbox with only a 50 GB quota is systematically blocked for normal traffic until you either add a license (EP2/E3/etc.) or delete enough content to go under 50 GB. [FACT:official] [techcommunity.microsoft](https://techcommunity.microsoft.com/blog/exchange/correcting-shared-mailbox-provisioning-and-sizing/607991)

## Shared status and delegation after license removal

The mailbox remains a shared mailbox object in Exchange Online; converting to shared is independent from assigning or removing the EP2 license. [FACT:official] Existing Full Access / Send As / Send on Behalf permissions and group membership still exist, so users can in principle still “have it shared,” open it, and read historical content, subject to the over‑quota restrictions and client behavior. [FACT:3rd-party] [learn.microsoft](https://learn.microsoft.com/en-us/microsoft-365/admin/email/about-shared-mailboxes?view=o365-worldwide)

The important nuance is that “can be shared” (delegation model) is not the same as “can continue to function as a production mailbox.” [INFERENCE] Technically it stays a shared mailbox with permissions intact, but practically it becomes a read‑only / blocked mailbox for new traffic until you either re‑license it or bring it under 50 GB. [INFERENCE] [blog.edbmails](https://blog.edbmails.com/correcting-shared-mailbox-provisioning-and-sizing.html)

The biggest missing piece is whether this mailbox has any compliance holds or archives enabled; are you using Litigation Hold / retention policies or Online Archive on this EP2 mailbox?
```
