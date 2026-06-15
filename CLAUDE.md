# kt-cheat-sheets

A collection of **succinct, copy-paste-friendly cheat sheets** explaining tech tools and concepts.

## Purpose

Each cheat sheet exists to get a reader productive fast. Optimise for **brevity and concision above all** — no filler, no prose where a code block will do. If a sentence isn't earning its place, cut it.

## Audience

Assume a competent engineer who is new to *this specific* tool. Don't explain general programming. Don't pad with motivation. Get to the point.

## Every cheat sheet must answer (up top, in 1–2 lines each)

1. **What it is** — one plain sentence.
2. **Why people use it** — the core problem it solves.
3. **What it's typically used for** — common real-world use cases.

## For tool-specific sheets (e.g. Docker, git, kubectl)

After the intro, include:

- **Most common commands** — the 80% you reach for daily, as a quick reference.
- **Simple example usages** — the most basic, correct way to do the common thing.
- **Advanced examples** — grounded in real practice. Where possible, anchor to how a known company/project uses it (e.g. "how Stripe uses idempotency keys"), kept just as brief.
- **Practical recipes** — a section of self-contained, copy-paste-ready blocks for real tasks the reader will actually want ("delete all images", "remove containers older than 2 days", "free up disk"). Each block: a one-line description of what it does, then the exact command. Favour the real day-to-day chores, including destructive cleanup ones — but clearly flag anything destructive.

Maximise copy-paste-ready code. Prefer a code block over a paragraph.

## Format rules

- Markdown, one file per topic: `topic-name.md` (kebab-case). Tool/concept sheets live in `cheatsheets/`; cross-cutting convention guides (architecture, error handling, testing, validation) live in `practices/`.
- Lead with the 3 intro answers, then commands, then examples (simple → advanced).
- Code blocks always tagged with a language.
- Real, runnable snippets — no pseudocode unless unavoidable.
- Comments inside snippets only when they clarify something non-obvious.
- Cite the source for "how X uses this" claims (link or name) so they're verifiable.
- Don't invent references. If you can't ground an advanced example in a real source, present it as a generic best-practice example instead.

## Anti-goals

- No tutorials that read like blog posts.
- No exhaustive flag dumps — curate to what's actually used.
- No marketing language.
