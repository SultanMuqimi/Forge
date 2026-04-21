# Contributing to Forge

Thanks for wanting to make Forge better. This document explains what we accept, how to propose changes, and the standards we hold contributions to.

Forge is opinionated on purpose. Most contributions add support for new stacks, new addons, or improve existing rule files based on real production lessons. PRs that try to soften the opinions (e.g., "let's add PHP support") will be respectfully closed.

---

## Ways to contribute

### 1. Report a bug or rough edge

Open an issue with:
- What you were trying to do
- What Forge (or the resulting Claude Code session) actually did
- The exact prompt or session output if possible

### 2. Improve an existing rule file

If a `CLAUDE.md` recommends something outdated, or you've shipped a project that exposed a real gap, open a PR. Include:
- A one-paragraph "why" — what real problem this fixes
- Before/after diff
- Any links to docs or post-mortems that justify the change

### 3. Add a new stack

We accept new stacks that meet **all** of these:
- At least 3 years of serious production use
- Active maintenance (commits in the last 6 months)
- Hireable talent pool
- Solves a use case the existing stacks handle poorly

To propose a stack, open an issue first describing the gap. If approved, submit a PR with:
- A new folder under `stacks/<stack-name>/`
- A `CLAUDE.md` (or `frontend-CLAUDE.md` + `backend-CLAUDE.md`) following the structure of existing stack files
- Updates to `docs/STACKS.md` explaining when Forge should pick it
- Updates to the interview prompt's stack table (Step 8) and fetch instructions (Step 9)

### 4. Add a new addon

Addons cover capabilities orthogonal to the base stack — payments, AI, real-time, multi-tenancy, etc. New addons should:
- Be needed across multiple stacks (otherwise it belongs in a single stack file)
- Have clear "when to load this" criteria
- Stack on top of the universal `CLAUDE.md` and the chosen stack — never replace them

### 5. Translate the interview prompt

The interview prompt currently auto-handles English and Arabic. We welcome translations to other languages. Open a PR with the translation as a separate file under a future `translations/` folder, plus an update to the language detection rule in the prompt.

### 6. Improve the docs

`STACKS.md`, `ERRORS.md`, `RESUME_TEMPLATE.md` — improvements always welcome. Real error fixes from real Claude Code sessions are especially valuable.

---

## What we won't accept

- **PHP, Laravel, or WordPress support.** Hard rule, won't change.
- **Microservices as a default.** Monolith is the default forever; microservices remain a senior-only deviation.
- **Adding more LLM provider abstractions.** Use the providers' SDKs directly. Wrappers age badly.
- **Stacks that primarily exist to demo a niche tool** — we serve people shipping products, not framework enthusiasts.
- **Bumping recommendations to bleeding-edge versions.** Forge prefers stable over new. If a major version is less than 6 months old, it doesn't get recommended.

---

## PR standards

- **One change per PR.** Don't bundle a new stack, a doc rewrite, and a typo fix into one PR.
- **Match existing tone.** Direct, opinionated, plain English. No marketing language. No emoji-stuffed headers.
- **Prefer concrete over abstract.** "Use Fastify because Express middleware is callback-based and Fastify's plugin system is faster" beats "Fastify is more modern."
- **Test the behavior change.** If you modify the interview prompt, paste it into a fresh Claude conversation and verify it still flows correctly across all 5 experience levels.
- **Update related files.** A new stack means: stack folder + STACKS.md + interview prompt table + interview prompt fetch instructions + README stack table.

---

## Local development

There's nothing to "run." This repo is a collection of markdown rule files plus an interview prompt. To test changes:

1. Edit the relevant file
2. Open [claude.ai](https://claude.ai), start a new conversation
3. Paste the (modified) interview prompt from the README
4. Walk through the interview as if you were a real user
5. When Forge generates a Claude Code prompt, paste it into Claude Code in a throwaway directory
6. Confirm the rule file fetches succeed and Claude Code follows them

If something breaks, fix it before opening the PR.

---

## Style guide for `CLAUDE.md` files

- **Imperative voice.** "Use Fastify." not "You should consider using Fastify."
- **Specific over general.** "Set max payload size to 64KB" beats "set a reasonable max payload size."
- **Reasons matter.** Where a rule isn't obvious, include a one-line "why."
- **No filler.** Cut every sentence that doesn't change behavior.
- **Code examples sparingly** — only when the pattern is genuinely non-obvious. The point is rules, not tutorials.
- **Keep universal rules in `/CLAUDE.md`.** Stack-specific files only contain what differs from the universal baseline.

---

## Code of conduct

Be direct and respectful. Disagreements are fine; condescension isn't. We're all trying to make this better for people building real things.

---

## Questions

Open an issue with the `question` label. Author Sultan Al-Muqimi ([@SultanMuqimi](https://github.com/SultanMuqimi)) reviews most PRs personally.
