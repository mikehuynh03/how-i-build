# how-i-build

The workflow I use to ship products with AI agents doing the implementation and me doing the thinking. Idea to live URL in days, with written plans, fresh-context execution, and proof before anything is called done.

Built on Claude Code and the [superpowers](https://github.com/obra/superpowers) plugin by Jesse Vincent, which provides the process skills named below.

## The workflow

### 1. Brainstorm out loud
Bounce a half-formed idea around in ChatGPT voice mode. Immediate audio feedback makes it faster to think out loud and get to a clear idea than typing ever is.

### 2. Plan with Claude
Feed the voice conversation into Claude Code (Opus) to turn the messy thinking into a real spec and plan. The superpowers `writing-plans` skill structures it: exact files, bite-sized tasks, test steps, no placeholders.

### 3. Review the plan myself
The manual step that makes everything downstream work. Go through the plan, answer its open questions, add the concrete details that solidify it, and turn it into a handoff guide that carries cleanly into a new chat.

### 4. Execute in a fresh context
Spin up a fresh conversation so the planning chat's context doesn't bloat the build. The `subagent-driven-development` skill implements task by task, each with its own tests, in a clean context window.

### 5. Review and polish
The `verification-before-completion` skill forces proof (run the command, show the output) before any "done" claim. Then patch the small details that slipped through.

**Seeing what's being built:** when text isn't enough, the [lavish](https://github.com/kunchenguid/lavish-axi) `/lavish` skill renders the agent's plan or output as an interactive browser preview I can annotate and send feedback on directly, instead of reading walls of prose.

**Unsupervised variant:** when I don't want to supervise and care less about tokens, `/goal` on ultracode or xhigh effort runs the loop autonomously.

## The stack

Decided once so no project re-litigates it (the decision-fatigue tax is real):

| Layer | Default | Exception |
| --- | --- | --- |
| Frontend | Next.js (App Router, latest) | |
| Hosting | Vercel | |
| Database + backend | [Convex](https://convex.dev) (DB, functions, realtime) | Neon Postgres for simple à-la-carte apps |
| Auth | [Better Auth](https://better-auth.com) | Clerk for simple à-la-carte apps |
| Caching / rate limiting | Upstash Redis | |
| AI | AI SDK v6 via Vercel AI Gateway, or OpenRouter when I want to run across multiple models | |
| Runtime / tooling | Bun, with npm / pnpm depending on the project | |

The default is the integrated stack my most mature web project actually shipped on. Exceptions exist, but they are named exceptions, not per-project debates.

## Worked example

[example/plan-phase-3-gesture-interactions.md](example/plan-phase-3-gesture-interactions.md) is a real phase plan from gesture-canvas, exactly as executed: gesture vocabulary v2 (two-finger connect, fist delete, thumbs-up confirm). Every task in it maps to commits like `fix(gesture): track connect source/target continuously + show target ring` in the repo's history. The plan is the source of truth; the commits are the receipt.

## Skills I keep installed

The ones marked `skills/` are vendored in this repo (copy a folder into `~/.claude/skills/`); the rest link out to their source.

| Skill | What it does | By |
| --- | --- | --- |
| [superpowers](https://github.com/obra/superpowers) | The process backbone: `brainstorming`, `writing-plans`, `subagent-driven-development`, and `verification-before-completion` are the skills each workflow stage above runs on | [Jesse Vincent](https://github.com/obra) |
| [tidy-commit](skills/tidy-commit) | Turns a messy working tree into clean, atomic commits that mirror what actually changed | me |
| [ui-ux-pro-max](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill) | UI/UX design intelligence: styles, palettes, font pairings, UX guidelines across 10 stacks | nextlevelbuilder |
| [web-design-guidelines](skills/web-design-guidelines) | Audits UI code against [Web Interface Guidelines](https://github.com/vercel-labs/web-interface-guidelines) | [Vercel](https://github.com/vercel-labs) |
