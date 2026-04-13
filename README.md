# AI Cookbooks

Practical, empirically-tested techniques for working with AI models.
Each subfolder is a self-contained cookbook covering one topic: a specific
workflow, a prompt pattern, or a production-grade configuration.

## Cookbooks

- **[perplexity/](./perplexity/)** - Make Perplexity (and other web-grounded LLMs)
  admit what they don't know. A 60-word system prompt that forces every claim
  into a tagged taxonomy (`[FACT:official]` / `[FACT:3rd-party]` / `[INFERENCE]`
  / `[UNKNOWN]` / `[STALE]`), with an empirical before/after case study and a
  production-grade extended template.

## License

MIT - see [LICENSE](./LICENSE).

## Contributing

Cookbooks live as subfolders with their own `README.md`. Prefer concrete,
testable techniques over general advice. Empirical before/after beats theory.
