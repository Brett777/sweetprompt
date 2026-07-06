# SweetPrompt

**Compile a rough prompt into a precise, codebase-grounded Claude Code prompt, without executing it.**

SweetPrompt is a Claude Code skill that turns a rough, half-formed request into a precise, codebase-grounded prompt. You give it your first-draft ask (typos, vagueness, half-thoughts and all), and it reads your real codebase, works out what you actually mean and which files it touches, and returns a clean, specific, file-aware prompt that's ready to run. It never edits code or runs the task. **The rewritten prompt is the deliverable.**

Think of it as a cheap checkpoint between intent and execution: a wrong guess gets caught in a ten-second skim of a paragraph, not in a diff you have to unwind.

## Why run it first?

A vague prompt makes Claude Code guess, and wrong guesses cost you turns. It hunts for the wrong file, opens with a round of clarifying questions, or confidently builds the wrong thing, and you spend the next few messages steering it back. SweetPrompt front-loads that work: one quick, cheap pass that turns your idea into a precise spec so Claude gets it right on the first pass instead of the third.

- **Right files, first try.** It greps your actual code and cites only paths it has verified, so Claude doesn't burn a turn hunting for "the login thing."
- **No clarifying-question ping-pong.** Ambiguities get resolved up front (or surfaced as explicit assumptions you can confirm) instead of interrupting you mid-task.
- **Cheap pass, real savings.** The rewrite runs on fast, inexpensive Sonnet; the turns it saves are on your main model, where a wrong edit is slow and annoying to undo.
- **Scoped on purpose.** Built-in acceptance criteria and out-of-scope notes keep Claude from over-building or quietly missing the point.

The aim is fewer wasted turns and correction loops per task: the difference between a drawn-out back-and-forth and a clean **prompt → `go`**.

## When to use it (and when to skip it)

SweetPrompt earns its turn when a wrong first guess would be expensive:

- Vague or half-formed asks ("fix the export thing")
- Cross-cutting changes, or code you don't know well
- Anything where the wrong file or quiet scope creep means real cleanup

Skip it when your prompt already names its files and states its scope; sweetening a clear prompt just adds a turn. It's a checkpoint, not a toll booth.

## Install

Once accepted into the community marketplace:

```
/plugin marketplace add anthropics/claude-plugins-community
/plugin install sweetprompt@claude-community
```

Or test this repo locally. Clone it, then in Claude Code:

```
/plugin marketplace add /path/to/sweetprompt
/plugin install sweetprompt@sweetprompt-marketplace
```

## Usage

```
/sweetprompt:sweetprompt make teh login thing retry when it fails idk where it is
```

(The command is doubled because plugin skills are namespaced `plugin:skill`; here the plugin and the skill are both `sweetprompt`.)

You get back a polished, ready-to-run prompt plus a short note on what changed:

```
──✦  Sweetened Prompt  ────────────────────────────

**Goal** — Make `signIn()` in `src/auth/login.ts` resilient to flaky networks.

**Change** — Wrap the `fetch` call to retry up to 3× on network/5xx errors
with exponential backoff. Do **not** retry on `401`/`403`.

**Done when**
- [ ] A flaky network recovers within 3 attempts
- [ ] Auth failures (`401`/`403`) still surface immediately

────────────────────────────────────────────────────
  Grounded ·  `src/auth/login.ts` · `signIn()` at :42
```

Reply `go` to run it, or tell it what to adjust; the Sweetened Prompt is
already in context, so there's nothing to copy.

The `Grounded ·` line is your pre-flight check: it lists only the files and
symbols SweetPrompt actually verified in your code. Wrong file? Say so; any
adjustment re-emits the complete updated prompt, so the latest block is
always exactly what `go` will run.

## Examples

Rough in → sharp out, across the kinds of work where a vague prompt costs the most:

- **Avoid an open-ended refactor.** `clean up the auth code it's a mess` → a *bounded* prompt ("extract the duplicated token-refresh logic in `src/auth/*.ts` into one helper; keep public signatures unchanged; don't touch session storage") so Claude makes one safe, reviewable change instead of a sprawling rewrite you have to unwind.
- **Land on the right file, first try.** `the export button is broken fix it` → a prompt that names the real handler and the function it calls, with a repro and a "Done when…" line, skipping the turns Claude would otherwise spend hunting.
- **Keep Claude on a short leash.** `add a null check to getUser, don't change anything else` → a minimal, single-file prompt with a hard out-of-scope guard, so one line doesn't become a three-module refactor.
- **Match your codebase's conventions.** `add pagination to the users list` → a prompt grounded in the component that renders users today and the existing fetch pattern to mirror, so you don't end up reviewing a mismatched approach.
- **De-risk a cross-cutting rename.** `rename the User model to Account everywhere` → the verified touchpoints enumerated (model, migrations, imports, serializers), with what *not* to rename flagged.
- **Turn a messy ticket into a spec.** Paste a rambling issue → goal, files, steps, acceptance criteria, and open assumptions: one clean **prompt → `go`** instead of a back-and-forth.

## How it works

- Runs on **Sonnet** at **medium effort**: fast and cheap; reverts to your session model when you run the result, so execution happens on your normal model.
- **Read-only** (`Read`, `Grep`, `Glob`, `Agent`): it structurally cannot edit your code.
- **Explicit invocation only** (`/sweetprompt:sweetprompt`): never auto-triggers, so it won't hijack a prompt you meant to execute.
- Grounds file references against your real code and only cites paths it has verified; fans out to read-only `Explore` agents only for large, multi-subsystem repos.

## SweetPrompt vs plan mode

Claude Code's plan mode also explores before acting, so when do you want which?

- **Plan mode** runs on your session model and produces an implementation plan. Reach for it on large, multi-step work where the ***how*** needs review.
- **SweetPrompt** runs a cheap, fast pass and produces a better *prompt*. Reach for it when what needs review is the ***what***: intent, scope, and target files. It's lighter, and the artifact is portable: paste it into a ticket, share it with a teammate, or run it in a fresh session.

They compose: sweeten first, then run the result in plan mode for big changes.

## What makes it different

Most prompt tools rewrite your **words**: cleaner grammar, tighter structure, a tidier template. SweetPrompt reads your **code**. That's the line that matters:

- **Grounded, not generic.** It greps your real repo and cites only files and symbols it has verified, so the prompt points at `signIn()` in `src/auth/login.ts`, not a plausible-sounding guess. A rewrite that never opens your codebase can sharpen phrasing, but it can't tell you *where* the work actually lives.
- **You get the prompt, not a running task.** SweetPrompt stops at the rewrite and hands you a portable artifact: read it, paste it into a ticket, share it, or run it when you're ready. Because it's read-only, it can't touch your code while it thinks; rewrite-and-run tools trade that control for convenience.

## Permissions & safety

SweetPrompt is built to be safe to install and hard to let loose:

- **Read-only by design.** Its tools are `Read`, `Grep`, `Glob`, and `Agent`. `Agent` is used only to spawn read-only `Explore` subagents. It searches and reads your code; it does not edit files or run shell commands, and producing the rewritten prompt is its terminal action.
- **No background surface.** No MCP servers, no hooks, no bundled executables, no network calls, no install scripts. It adds ~100 tokens to a session and nothing else.
- **Explicit invocation only.** It never auto-triggers; it runs only when you type `/sweetprompt:sweetprompt`, so it can't hijack a prompt you meant to execute.
- **Text is the only output.** The deliverable is the rewritten prompt. Running it is a separate step you initiate, on your normal session model.

## Privacy

SweetPrompt collects nothing. It has no telemetry or analytics and makes no network calls of its own; it stores nothing and sends nothing to any third party. Like any Claude Code skill, the model running it processes your prompt and the code it reads as part of your normal Claude Code session, under Anthropic's existing [Privacy Policy](https://www.anthropic.com/legal/privacy); SweetPrompt adds no data collection or transmission of its own.

## License

MIT. See [LICENSE](LICENSE).
