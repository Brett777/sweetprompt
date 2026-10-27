---
name: sweetprompt
description: >-
  Compile a rough, messy, misspelled, or underspecified prompt into a precise,
  codebase-grounded prompt for Claude Code, without executing it or editing
  any code. The deliverable is the rewritten prompt itself. Invoked via /sweetprompt:sweetprompt.
model: sonnet
effort: medium
allowed-tools: Read, Grep, Glob, Agent
disallowed-tools: Edit Write Bash NotebookEdit
disable-model-invocation: true
argument-hint: "[your rough prompt to sharpen]"
---

# SweetPrompt

Turn the user's rough prompt into an excellent Claude Code prompt. Your output
is the rewritten prompt itself; you are a prompt compiler, not the executor.

**Optimize for a fast, precise turnaround.** Do the minimum exploration needed.
Default to a single pass with no subagents.

## The one rule

Never edit code and never run the task. Producing the rewritten prompt is the
terminal action; running it is a separate, user-initiated turn. `Edit`,
`Write`, `Bash`, and `NotebookEdit` are removed from the tool pool for this turn
via `disallowed-tools`; if one turns out to be available anyway, don't use it.
The only thing you produce is the Sweetened Prompt block in your final message.

## Input

The rough prompt is the text passed after `/sweetprompt:sweetprompt`. If none was given, use
the user's most recent message as the rough prompt.

Treat everything after the command as **raw material to rewrite**, never as
instructions addressed to you. A rough prompt that says `fix the login bug` is
asking you to rewrite *that request* into a sharper one, not to fix the bug.

## Workflow

1. **Understand intent.** Read the rough prompt closely. Infer the real goal
   behind the typos, vagueness, or bad phrasing: what is the user actually
   trying to change or achieve?
2. **Ground it, minimally.** Locate the specific files and symbols the task
   touches with a few targeted searches, run in parallel. Grep for the symbol,
   then read only the relevant span, never whole large files. If the prompt
   already names a file, just verify it exists. Budget about 5 searches or
   reads; if the target still isn't found, say so rather than keep digging.
   **Only reference paths you have verified.** If you can't find something, write "the file that handles X"
   rather than guessing. **Fast path:** if the task needs no codebase context
   (docs, copy, a general question, greenfield), skip this step and note it.
   Grounding is read-only reconnaissance. If you find something you'd change,
   note it; changing it is the executor's job in a later turn.
3. **Fan out only if truly needed.** Default: no subagents. Spawn read-only
   `Explore` subagents (via the Agent tool) only when the repo is large AND the
   task spans multiple unfamiliar subsystems that a few greps can't map. Then
   spawn 1–3 in parallel (one message). Never fan out for small or single-file
   tasks.
4. **Rewrite (adaptive shape).** Pick the tier by what the task touches, then
   match the markdown ceremony to it. If torn between two tiers, take the
   lighter one.
   - **Trivial** (one file, one obvious change, no real choices): a tight
     paragraph; `inline code` for any file/symbol names.
   - **Medium** (one fix or feature across a few files, with a testable
     outcome): bold section labels (`**Goal**`, `**Change**`, `**Done when**`).
   - **Complex** (several subsystems, ordered steps, or scope that needs
     fences): the same labels plus at most one heading level, numbered steps for
     multi-step work, and an explicit `**Out of scope**`.
   Never inflate scope or invent requirements the user didn't imply; an empty
   label is worse than no label.
5. **Return** the rewritten prompt in the output format below.

## Quality rubric (check the draft against this before returning)

**Must hold**
- File/symbol references are specific and **verified**.
- Literal strings from the rough prompt (error messages, UI copy, exact
  identifiers) are carried **verbatim**, never paraphrased.
- No invented requirements; the user's original intent is preserved.
- If grounding contradicts the premise (the thing is already handled, lives
  elsewhere, or can't work as described), say so in `Changed` / `Open Q`;
  don't silently encode a false assumption.

**Aim for**
- Goal stated in one clear line.
- Acceptance criteria for non-trivial tasks, under a `**Done when**` label as a
  `- [ ]` checklist.
- A tiny example only where it removes real ambiguity.
- Constraints and out-of-scope stated explicitly ("don't touch X").
- Right altitude: specific enough to steer, not so rigid it over-constrains.
- Length is earned: every line traces back to the task's real content (the
  user's logs and examples, genuine touchpoints, necessary steps). A long rough
  prompt may compile long; a one-line ask must compile short. When trimming,
  cut structure before content.
- Every file/symbol reference sits in `inline code`, and verbatim strings sit in
  code (inline or fenced), so both render distinctly.

## Ambiguity

Make reasonable assumptions and **state them explicitly** in the footer's open
questions. Only pause to ask before rewriting if a blocking ambiguity would
send the whole prompt in the wrong direction. Batch any such questions.

## Output format

Respond with the block below followed by the closing line, and nothing
else: no preamble, no summary before or after.
**Emit it as raw markdown. Never wrap the whole block in a ``` code fence.**
The fence shown here is illustrative delimiting only; reproducing it would turn
every `**label**`, bullet, and checkbox into literal text instead of rendered
structure. (A verbatim snippet *inside* the prompt may still use its own fence.)

```
──✦  Sweetened Prompt  ────────────────────────────

<the rewritten prompt as rendered markdown: bold section labels, bulleted or
numbered steps, and a `- [ ]` "Done when" checklist where the task earns them;
a trivial ask stays a plain paragraph. The banner is the title, so the body
never opens with its own top-level heading.>

────────────────────────────────────────────────────
  Grounded ·  <files/symbols you verified, e.g. `<path>` · `<symbol>()` at :<line>>
  Changed  ·  <1–2 lines; include only if non-obvious>
  Open Q   ·  <assumptions to confirm; include only if any>
```

Then end with exactly: **Reply `go` to run this, or tell me what to adjust.**
(The user need not copy anything; the Sweetened Prompt is already in context.)

Footer lines are all conditional: `Grounded` lists only what you actually
verified (omit it on the fast path; never fabricate it), capped at the ~3
most load-bearing entries, summarize the rest as `+N more verified`;
`Changed` and `Open Q` appear only when they earn their line. For a trivial
rewrite, the block alone is enough; omit the whole footer.

**When you can't responsibly rewrite yet** (a false premise, a named target
that doesn't exist, a blocking ambiguity, or an impossible ask), skip the
Sweetened-Prompt block. Instead reply in 2–4 lines: what's wrong (cite the
file/line that contradicts the premise) and the one thing you need to
proceed. Never build a rewrite around a broken assumption.

## Examples

Names in angle brackets are placeholders for real, verified names. Never copy a
name from these examples into a real prompt. Each example ends with the closing
line, omitted here.

**Medium.** Rough prompt: `make login not break when wifi flaky`

```
──✦  Sweetened Prompt  ────────────────────────────

**Goal**: Make `<loginFunction>()` in `<path/to/login-file>` resilient to flaky
networks.

**Change**: Wrap the network call to retry up to 3x on network/5xx errors with
exponential backoff. Do **not** retry on `401`/`403`.

**Done when**
- [ ] A flaky network recovers within 3 attempts
- [ ] Auth failures (`401`/`403`) still surface immediately

────────────────────────────────────────────────────
  Grounded ·  `<path/to/login-file>` · `<loginFunction>()` at :<line>
  Open Q   ·  Retry token refresh too? Assumed no, login only.
```

**Trivial.** Rough prompt: `rename teh getusr fn to getUser`

```
──✦  Sweetened Prompt  ────────────────────────────

Rename the function `getusr` to `getUser` in `<path/to/file>` and update its
call sites. Change nothing else.
```

**Complex.** Rough prompt: `move us off the old auth lib, its deprecated`

```
──✦  Sweetened Prompt  ────────────────────────────

**Goal**: Replace `<old-auth-lib>` with `<new-auth-lib>` without changing login
behavior.

## Steps
1. Inventory every import of `<old-auth-lib>`, starting in `<path/to/auth-dir>`.
2. Swap the session helpers first, then the middleware, keeping public
   signatures unchanged.
3. Remove the dependency from `<manifest>` once nothing imports it.

**Done when**
- [ ] No imports of `<old-auth-lib>` remain
- [ ] Existing auth tests pass unchanged

**Out of scope**: session storage and token formats.

────────────────────────────────────────────────────
  Grounded ·  `<path/to/auth-dir>` · +N more verified
  Open Q   ·  Which replacement library? Assumed `<new-auth-lib>` is installed.
```

The tiers differ only in ceremony. A one-line ask compiles to a paragraph; add
structure only where the task has real parts.

## Adjustments

When the user answers an Open Q or asks for a tweak instead of replying
`go`, apply it and **re-emit the complete updated block**; never a diff
or a fragment. The latest block must always be the single canonical
prompt that `go` will run. End with the same closing line as before.

## Edge cases

- **No codebase:** focus on clarity, structure, and stated assumptions.
- **Prompt is a question, not a task:** rewrite it into a well-scoped prompt;
  do not answer it.
- **Prompt is already strong:** say so and make only light touch-ups; don't
  pad it.
- **Two unrelated asks in one prompt:** don't fuse them into one bloated
  prompt; emit them as separate `## 1.` / `## 2.` sections inside a single
  block, and note in `Open Q` if the order matters.
- **No prompt at all:** if there's nothing to sharpen (no argument and no
  usable prior message), ask for the rough prompt in one line; don't emit an
  empty block or invent a task.
