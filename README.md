# promptup

**Compile a rough prompt into a precise, codebase-grounded Claude Code prompt — without executing it.**

`promptup` is a Claude Code skill that turns a rough, half-formed request into a precise, codebase-grounded prompt. You give it your first-draft ask — typos, vagueness, half-thoughts and all — and it reads your real codebase, works out what you actually mean and which files it touches, and returns a clean, specific, file-aware prompt that's ready to run. It never edits code or runs the task — **the rewritten prompt is the deliverable.**

## Why run it first?

A vague prompt makes Claude Code guess — and wrong guesses cost you turns. It hunts for the wrong file, opens with a round of clarifying questions, or confidently builds the wrong thing, and you spend the next few messages steering it back. `promptup` front-loads that work: a few seconds turning your idea into a precise spec so Claude gets it right on the first pass instead of the third.

- **Right files, first try.** It greps your actual code and cites only paths it has verified, so Claude doesn't burn a turn hunting for "the login thing."
- **No clarifying-question ping-pong.** Ambiguities get resolved up front — or surfaced as explicit assumptions you can confirm — instead of interrupting you mid-task.
- **Cheap pass, real savings.** The rewrite runs on fast, inexpensive Sonnet; the turns it saves are on your main model, where a wrong edit is slow and annoying to undo.
- **Scoped on purpose.** Built-in acceptance criteria and out-of-scope notes keep Claude from over-building or quietly missing the point.

The net effect is fewer wasted turns and correction loops per task — the difference between a drawn-out back-and-forth and a clean **prompt → `go`**.

## Install

Once accepted into the community marketplace:

```
/plugin marketplace add anthropics/claude-plugins-community
/plugin install promptup@claude-community
```

Or test this repo locally. Clone it, then in Claude Code:

```
/plugin marketplace add /path/to/promptup
/plugin install promptup@promptup-marketplace
```

## Usage

```
/promptup:promptup make teh login thing retry when it fails idk where it is
```

(The command is doubled because plugin skills are namespaced `plugin:skill` — here the plugin and the skill are both `promptup`.)

You get back a polished, ready-to-run prompt plus a short note on what changed:

```
── Improved prompt ─────────────────────────────
Add retry logic to the login request in src/auth/login.ts. Wrap the
fetch in `signIn()` to retry up to 3× on network/5xx errors with
exponential backoff; do not retry on 401/403. Done when a flaky
network recovers within 3 attempts and auth failures still surface.
─────────────────────────────────────────────────
```

Reply `go` to run it, or tell it what to adjust — the improved prompt is
already in context, so there's nothing to copy.

## How it works

- Runs on **Sonnet** at **medium effort** — fast and cheap; reverts to your session model when you run the result, so execution happens on your normal model.
- **Read-only** (`Read`, `Grep`, `Glob`, `Agent`) — it structurally cannot edit your code.
- **Explicit invocation only** (`/promptup`) — never auto-triggers, so it won't hijack a prompt you meant to execute.
- Grounds file references against your real code and only cites paths it has verified; fans out to read-only `Explore` agents only for large, multi-subsystem repos.

## License

MIT — see [LICENSE](LICENSE).
