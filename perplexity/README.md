# Make Perplexity Admit What It Doesn't Know

## A 60-word system prompt that stops Perplexity from confidently making things up

**TL;DR**: Perplexity with web search hallucinates URLs, dates, and specific numbers at a shocking rate - and does it with full confidence. One short system message forces it to tag every claim by source quality and flag unknowns. Tested empirically on Perplexity API (sonar-pro). The before/after is night and day. The pattern should generalize to any LLM that accepts a system message, but this repo only documents the Perplexity test.

---

## The problem

Ask Perplexity a question about a technical API and you get back a clean, detailed answer. Specific RPM limits. Exact prices per tier. Precise cache TTLs. Links to documentation pages. Dates of recent updates.

It reads like a Stack Overflow answer from a staff engineer.

The issue: most of it is made up.

I ran a test. Asked Perplexity five specific questions about its own API - rate limits, caching behavior, pricing multipliers, domain filter support. The first call gave me:

- Exact rate limits per tier ("sonar: 60 RPM / 1M tokens per day, sonar-pro: 120 / 5M...")
- Cache behavior with specific TTLs ("10 minutes standard, 1 hour for Deep Research, $0 on cache hit")
- Pricing multipliers to three decimals ("minimal 1x, low 1.5x, medium 2.5x, high 4x")
- A documentation URL (`docs.perplexity.ai/docs/caching`) with an update date ("Feb 28, 2026")
- Another URL for the usage endpoint

All of it sounded plausible. None of it survived verification. The URLs don't exist. The rate limits aren't documented. The cache behavior isn't documented. The pricing multipliers are fiction.

This is not a Perplexity-specific failure. This is what LLMs do when asked for information they don't have: they pattern-match "how an answer like this should look" and make one up. The confidence is a byproduct of the training objective, not a signal of accuracy.

## The fix

One system message. Sixty words. Paste it at the start of every call to your LLM:

```
State ASSUMPTIONS, MISSING, SCOPE before answering. Tag: [FACT:official], [FACT:3rd-party], [INFERENCE], [UNKNOWN]. Cite dates (YYYY-MM-DD). Flag sources >6mo as [STALE]. Challenge question assumptions. Precision > hedging.
```

That's it. Now re-run the same question.

Here's what I got back for the same five questions on Perplexity, with this prompt in the system message:

> **1. RATE LIMITS**
> [UNKNOWN] No search results specify default RPM or tokens/day per tier. One changelog notes increased default to 50 RPM for "sonar online models" (post-2025-07) [STALE]. No /usage API endpoint documented.
>
> **2. REQUEST CACHING**
> [UNKNOWN] No evidence API caches identical requests, cache TTL, $0 hits, or key including reasoning_effort/stream. Premise challenged: no caching mentioned in API guides.
>
> **3. REASONING_EFFORT COSTS**
> [UNKNOWN] No concrete pricing differences or multipliers documented for any model. Costs shown per-request only.
>
> **4. SEARCH_DOMAIN_FILTER on sonar-deep-research**
> [UNKNOWN] No mention of SEARCH_DOMAIN_FILTER in raw API request body for any model. Filter unsupported or undocumented.
>
> **5. WILDCARD SUPPORT**
> [UNKNOWN] No evidence of any filter format.

Same model. Same question. Same web access. Completely different answer.

The second version is boring. It refuses to tell me specific numbers. It says "no evidence" over and over.

The second version is also the truthful one.

## Why it works

The prompt does four things that matter:

**1. It forces a taxonomy of claims.** Every statement has to land in one of four buckets. The model can't hide an `[INFERENCE]` inside a sentence that reads like a `[FACT:official]` - it has to commit to a tag. The act of tagging creates a moment of self-check: "do I actually have a source for this, or am I pattern-matching?"

**2. It makes "I don't know" a valid answer.** Default LLM output pressure is toward completeness. Empty answers feel like failures. The `[UNKNOWN]` tag gives the model explicit permission to say nothing - with dignity, in a structured way. Without that tag, the model fills the void with invention.

**3. It pre-commits to citing dates.** When the model has to print `YYYY-MM-DD`, it has to actually think about whether its source has a date. Undated claims become conspicuous. The `[STALE]` flag for anything older than six months forces a second judgment call on recency.

**4. It grants permission to push back on the question itself.** "Challenge question assumptions" lets the model say "your premise is wrong" instead of answering a broken question with a fake answer. In my test this caught something real - Perplexity pointed out that `sonar-deep-research` might not be a documented model name, which would change my entire cost model. That's a finding I'd never get from a standard answer.

## The semantics of each tag

You need to understand what each tag means or the output is worthless:

| Tag | Meaning | What to do with it |
|---|---|---|
| `[FACT:official]` | Statement backed by the authoritative source (docs, spec, API reference) | Use it. No extra verification needed for non-critical work. |
| `[FACT:3rd-party]` | Statement backed by a reputable but not-authoritative source (blog, Stack Overflow, GitHub issue) | Use it but note the source in your notes. Lower trust weight. |
| `[INFERENCE]` | The model is reasoning from facts, not citing a fact | Treat as hypothesis, not fact. Verify before acting. |
| `[UNKNOWN]` | No source found. The model doesn't know. | Don't act on it. Either research further or accept the gap. |
| `[STALE]` | Source older than six months | Doubly suspicious for fast-moving topics (APIs, pricing, policy). Re-verify. |

The tags are not decoration. They're a contract between you and the model about epistemic honesty.

## How to use it with Perplexity API

Perplexity's chat completion endpoint takes a standard OpenAI-style messages array. Put the system message first:

```python
import requests

messages = [
    {
        "role": "system",
        "content": (
            "State ASSUMPTIONS, MISSING, SCOPE before answering. "
            "Tag: [FACT:official], [FACT:3rd-party], [INFERENCE], [UNKNOWN]. "
            "Cite dates (YYYY-MM-DD). Flag sources >6mo as [STALE]. "
            "Challenge question assumptions. Precision > hedging."
        )
    },
    {
        "role": "user",
        "content": "Your actual question here"
    }
]

response = requests.post(
    "https://api.perplexity.ai/chat/completions",
    headers={"Authorization": f"Bearer {api_key}"},
    json={
        "model": "sonar-pro",
        "messages": messages,
        "search_context_size": "high",       # more sources = better grounding
        "search_recency_filter": "year",      # technical questions rarely need older data
        "search_domain_filter": [             # restrict to trusted domains when possible
            "docs.perplexity.ai",
            "learn.microsoft.com"
        ]
    }
)
```

Three parameters beyond the system message matter for technical queries:

- `search_context_size: "high"` - retrieves more web context per call. More expensive, much better grounding.
- `search_recency_filter: "year"` - blocks old blog posts and dead tutorials from polluting the answer.
- `search_domain_filter` - lock the model to specific documentation domains when you know what authoritative sources look like.

If you're using Perplexity through an MCP wrapper (like Claude Code), check whether your wrapper exposes these parameters. If it doesn't, push to get them added.

## Does this work with other LLMs?

Not tested here. The empirical before/after in this repo is Perplexity API only (sonar-pro). The pattern is model-agnostic in principle - any LLM that accepts a system message can be told to tag its claims - but I have not run the same A/B on Claude, GPT, or Gemini, so I am not claiming it. If you reproduce it on another model, I would like to hear how it went.

The only adaptation for closed-book use (no web access): drop the `[STALE]` instruction - it does not apply when there are no dated sources to flag.

A trimmed version for closed-book use:

```
Before answering, state ASSUMPTIONS and SCOPE. Tag every claim: [FACT] for things you're certain of, [INFERENCE] for reasoning, [UNKNOWN] for gaps. Challenge the premise if the question is malformed. Precision > hedging.
```

This works particularly well for:
- Technical research (API behavior, library docs)
- Legal/compliance questions
- Medical information
- Anything where a confident wrong answer is worse than no answer

It works less well for:
- Creative writing (tags break flow)
- Brainstorming (you want loose output)
- Casual chat (overkill)

## What changes in your workflow

Once you see the tagged output, you start treating LLM answers differently. You scan for the tag, not the prose. A paragraph full of `[UNKNOWN]` tells you to stop asking and go look somewhere else. A paragraph full of `[FACT:official]` tells you to trust and proceed. A paragraph with a mix tells you what to verify.

This is the part I didn't expect: the tags don't just make the model more honest. They make you more honest with yourself about what you actually know. You stop copying answers into your notes verbatim. You copy only the `[FACT:*]` parts and flag the `[INFERENCE]` parts for later verification.

It changes research from "ask AI, paste result" to "ask AI, audit result, keep the audit-surviving parts."

## The case for adopting this immediately

Most people pay for LLM tokens thinking they're buying information. They're actually buying pattern-matched-plausibility-shaped-like-information. The difference only matters when you act on the output.

Acting on untagged LLM output in technical or high-stakes contexts is dangerous. I almost wrote "perplexity_research has a 10-minute cache" into documentation I was going to share, based on the first un-tagged answer. That would have been a lie I spread downstream, with authoritative confidence, because my source sounded authoritative.

The fix takes 60 words in a system message. Paste it once. Every call after that is less dangerous.

## One last thing

If you're building an agent or a workflow that calls LLMs programmatically, put this in your default system message. Make it impossible to call your LLM without it. Treat untagged LLM output the way you treat unsanitized user input: as something that needs to be cleaned before it enters your system.

The cost is that some answers become shorter and more boring. The benefit is that they become trustworthy.

That trade is very, very worth it.

---

## Credit and sources

This technique came out of an empirical debugging session with Perplexity API through a Claude Code MCP wrapper (April 2026). The pattern was refined by a Claude Code user testing identical queries with and without the system message injection. The specific tag taxonomy (`FACT:official` / `FACT:3rd-party` / `INFERENCE` / `UNKNOWN` / `STALE`) is original to this workflow.

If you try it and find it works or breaks, please share what you ran.

## Copy-paste templates

**For Perplexity API (web-grounded, full template):**
```
State ASSUMPTIONS, MISSING, SCOPE before answering. Tag: [FACT:official], [FACT:3rd-party], [INFERENCE], [UNKNOWN]. Cite dates (YYYY-MM-DD). Flag sources >6mo as [STALE]. Challenge question assumptions. Precision > hedging.
```

**For closed-book LLMs (no web access):**
```
Before answering, state ASSUMPTIONS and SCOPE. Tag every claim: [FACT] for things you're certain of, [INFERENCE] for reasoning, [UNKNOWN] for gaps. Challenge the premise if the question is malformed. Precision > hedging.
```

**For agents/subagents that return structured output:**
```
Respond in this format:
## Assumptions: <what you assumed>
## Scope: <what this answer covers and does not cover>
## Answer: <your answer, every claim tagged [FACT:official], [FACT:3rd-party], [INFERENCE], or [UNKNOWN]>
## Unknowns: <things you could not answer>
Challenge the question if the premise is wrong. Precision > hedging.
```

## Extended version for production use

The 60-word template above is the compact version. If you want a production-grade version with more specific rules - marketing-vs-spec distinction, conditional branching when inputs are incomplete, explicit handling of conflicting sources, domain-specific rules for software/API research - see `perplexity-desktop-config.md` in this repo. That file has the full template that has been running in a real technical research workflow, with explanations for each rule and what failure mode it catches.

Use the compact version for general work. Use the extended version when you need strict discipline for deep technical research.
