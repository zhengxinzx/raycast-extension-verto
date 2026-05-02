# Verto — PRD

A Raycast extension that runs user-defined LLM prompts on the currently selected text in any macOS app.

## Problem Statement

When I'm writing in any macOS app — email, Slack, a Jira comment, a code review, a blog draft — I often want to apply a quick LLM operation to the text I just wrote or the text I'm reading. Polish my non-native English. Translate a snippet to or from Chinese. Make a curt message sound more polite for a colleague.

Today this round-trips through a chat UI: switch app, paste text, type the instruction, copy the result, switch back. Each switch breaks flow. Worse, the *instruction* (the prompt) gets retyped every time, even though I use the same handful of prompts again and again.

I want a tool where:

- Selected text becomes the input to a saved prompt with one keystroke.
- The set of prompts is mine — I can edit the wording, add my own, restore a built-in to its original wording if I tinkered too far.
- The result streams back inline so I can read as it generates and act on it (copy or paste) without leaving Raycast.

## Solution

Verto is a Raycast extension with a hybrid command model:

- **Three top-level commands** — *Polish Selection*, *Translate Selection*, *Make Selection Polite* — each bound to a specific built-in prompt and individually hotkey-able from Raycast preferences.
- **One generic command** — *Run Custom Prompt* — opens a picker over every prompt (built-in + user-defined) so the long tail is always one extra keystroke away.
- **One management command** — *Manage Prompts* — full CRUD: edit any prompt, duplicate, delete user-defined ones, restore built-ins to their factory wording, create new ones.

All prompts — the three built-ins and any user creations — live in a single `LocalStorage`-backed registry. The built-ins are seeded on first run from a static catalog that also remembers each one's *factory* template, which is what powers Restore Default.

Running any prompt:

1. Reads the selection from the frontmost app via `getSelectedText()`. If that fails (nothing selected, app blocks the macOS Accessibility API, or Raycast lacks permission), falls back to `Clipboard.readText()`. If both yield nothing, shows a fail-fast toast and exits.
2. Substitutes the captured text into the prompt's `{{text}}` placeholder.
3. Streams the LLM response into a Raycast `<Detail>` view. ⌘C copies the full result; ⌘V pastes into the previously-active app. ⌘R re-runs.

The LLM is OpenRouter (OpenAI-compatible). The user supplies their own API key as a Raycast extension preference. Model is a second extension preference (default `openai/gpt-4o-mini`) so the user can swap to any OpenRouter model ID without code changes.

## User Stories

1. As a non-native English speaker, I want to select text I've written in any app and run *Polish Selection*, so that I get a more native-sounding rewrite without leaving the app.
2. As a multilingual user, I want to select text and run *Translate Selection*, so that I get the translation streamed inline without opening a separate translator.
3. As someone navigating workplace tone, I want to select a draft message and run *Make Selection Polite*, so that I can soften the wording before sending it to a colleague.
4. As a Raycast user, I want each top-level command to appear in Raycast's root search, so that I can discover them by typing their name.
5. As a Raycast power user, I want to bind a hotkey to each top-level command in Raycast preferences, so that I can polish/translate/politen text in one keystroke.
6. As a user with prompts beyond the built-ins, I want a *Run Custom Prompt* command that lists every prompt I have, so that I can pick any of them without each needing its own top-level command.
7. As a user of the picker, I want the selection to be captured the moment I open the picker (not after I pick a prompt), so that I find out immediately if there's no input rather than after picking.
8. As a user, I want the picker to fall back to my clipboard when nothing is selected, so that I can still act on text I copied a moment ago.
9. As a user, I want a clear failure toast when there's neither a selection nor clipboard text, so that I know exactly what to do next.
10. As a user, I want LLM output to stream into a `<Detail>` view, so that I can read along as it generates instead of staring at a spinner.
11. As a user reading the result, I want to press ⌘C to copy the whole output to my clipboard, so that I can paste it wherever I want.
12. As a user reading the result, I want to press ⌘V to paste the output into the app I was in before opening Raycast, so that the result lands at my cursor without manual clipboard fiddling.
13. As a user who didn't quite get the result I wanted, I want to press ⌘R to re-run the same prompt against the same input, so that I can roll for a better answer without starting over.
14. As a user, I want the streaming view to show the prompt's name in its title, so that I always know which operation produced the visible result.
15. As a user with a slow or failing API call, I want any LLM error to surface clearly in the view (not as a silent spinner), so that I can recover or retry.
16. As a user, I want the streaming view to show a loading indicator while the model is producing tokens, so that I can tell the difference between "still generating" and "done".
17. As a user, I want a *Manage Prompts* command that lists every prompt, so that I have one place to see and edit my whole prompt library.
18. As a user editing a prompt, I want a form with a Name field and a multi-line Template field, so that I can clearly see and change both.
19. As a user creating a new prompt, I want the form to default to a sensible starter template containing `{{text}}`, so that I have an obvious example to riff on.
20. As a user editing a prompt that omits the `{{text}}` placeholder, I want the system to still work (by appending the selected text after the template), so that I'm not forced to learn template syntax for trivial prompts.
21. As a user, I want the form to require a non-empty name, so that every prompt is identifiable in the picker.
22. As a user creating two prompts with the same name, I want the system to give the second one a unique id automatically, so that I don't have to manage ids myself.
23. As a user, I want a *Duplicate* action on every prompt, so that I can fork a built-in or one of my own as a starting point for a variant.
24. As a user, I want a *Delete* action on user-defined prompts, so that I can prune ones I no longer use.
25. As a user attempting to delete a built-in, I want the system to refuse and suggest *Restore Default* instead, so that I can't accidentally lose a built-in entirely.
26. As a user who has edited a built-in, I want a *Restore Default* action to revert it to the factory wording, so that I can undo experimental changes without remembering the original text.
27. As a user, I want *Restore Default* to ask for confirmation before overwriting my edits, so that I don't lose work to a stray keystroke.
28. As a user, I want *Restore Default* to be hidden on built-ins I haven't edited, so that the action panel only shows actions that would actually do something.
29. As a user, I want *Restore Default* to be hidden on user-defined prompts (they have no factory default), so that the UI doesn't suggest a non-existent capability.
30. As a user upgrading the extension to a version that ships a new built-in, I want the new built-in to appear in my registry on next launch, so that I get new built-ins without having to reinstall.
31. As a user upgrading the extension, I want my edits to existing built-ins to be preserved across upgrades, so that I never lose customizations to a re-seed.
32. As a user using `{{text}}` in unusual positions (e.g. mid-sentence: `Translate "{{text}}" into French.`), I want the placeholder to be replaced wherever it appears, not just appended at the end, so that I have full control over prompt structure.
33. As a user with a multi-occurrence template (e.g. `Original: {{text}}\nRewrite: {{text}}`), I want every occurrence of `{{text}}` to be replaced, so that the template behaves intuitively.
34. As a Raycast user, I want my OpenRouter API key stored as a `password` preference, so that it isn't visible in plain text and doesn't leak into Raycast's clipboard history.
35. As a Raycast user, I want the model to be a `textfield` preference defaulting to `openai/gpt-4o-mini`, so that I can switch to any OpenRouter model ID (including ones that don't exist yet) without waiting for an extension update.
36. As a user who hasn't set an API key yet, I want the missing-key error to be clear and point me at the preferences pane, so that I'm not left guessing why nothing happens.
37. As a user with a very long selection (~10k+ chars), I want the extension to send it as-is to OpenRouter and surface any context-length error from the model in the view, so that I'm not silently truncated and I'm told the truth about what happened.
38. As a user who hits an OpenRouter rate limit, I want the rate-limit error message surfaced in the view so I can act on it (wait, switch model, etc.).
39. As a first-time user on macOS, I want a clear path to grant Raycast Accessibility permission, so that `getSelectedText()` works on apps that respect it. (This is a Raycast-level prompt; Verto's README documents it.)
40. As a user, I want a small, clear README that explains setup (API key, model, accessibility), so that I can get to a working state in minutes.
41. As a user, I want a populated `package.json` (categories, description, author, icon) and a 512×512 icon, so that the extension can pass `npm run build` checks and is suitable for eventual Store submission.
42. As a developer maintaining Verto, I want all four "run a prompt" entry points (3 top-level + picker) to share a single streaming view component, so that fixing a streaming/UX bug fixes it everywhere at once.
43. As a developer maintaining Verto, I want the prompt registry to be a deep module hiding all storage, seeding, and upgrade-merge logic, so that callers never touch JSON or the storage key.
44. As a developer maintaining Verto, I want the template renderer to be a pure function with no Raycast dependencies, so that it's trivially unit-testable.

## Implementation Decisions

### Modules

- **`template`** — pure renderer module. Single function `render(template, text)`: replaces all `{{text}}` occurrences with the input; if no placeholder is present, appends the input after a blank line. No I/O, no Raycast dependency.

- **`builtins`** — declarative catalog of seeded prompts. Exposes a static array of `BuiltInPrompt` records; each carries `id`, `name`, `template` (the *current shipped* version), and `defaultTemplate` (the factory wording, used by Restore Default). Also exposes a lookup `getBuiltInDefault(id)`. The catalog ships exactly three entries on day one: `polish`, `translate`, `polite`. Built-in ids are treated as stable identifiers across versions.

- **`prompts`** — LocalStorage-backed prompt registry. Public surface: `list()`, `get(id)`, `upsert(prompt)`, `delete(id)`, `restoreDefault(id)`, `isEdited(prompt)`, `slugify(name)`, `uniqueId(base)`. Internal responsibilities (hidden from callers): the storage key, JSON (de)serialization, seeding-on-first-run from the `builtins` catalog, on-launch upgrade merge that adds newly-shipped built-ins without overwriting user edits to existing ones, and the invariant that `delete(id)` rejects built-ins (they must be reset via `restoreDefault` instead).

- **`selection`** — selection-input adapter. Public: `readSelectionWithFallback()` returning a non-empty string or throwing a typed `NoInputError`. Tries `getSelectedText()` first, then `Clipboard.readText()`; both empty/whitespace-only ⇒ throw. `NoInputError.message` is user-actionable.

- **`llm`** — OpenRouter streaming client. Public: `streamCompletion(prompt, onChunk, signal)` returning the full assembled string. Internal: lazy OpenAI SDK client construction with `baseURL` set to OpenRouter, API key from preferences, `HTTP-Referer` and `X-Title` headers (per OpenRouter recommendations), model from preferences with the documented default fallback. Streaming uses the SDK's `stream: true` chat-completions API; abort propagates via `AbortSignal`.

- **`StreamingDetail`** — shared React component used by all four "run a prompt" entry points. Props: `promptId` (required) and an optional `preCapturedText` (used by the picker, which captures at launch so the `<List>` doesn't need to). Owns the lifecycle: load prompt → resolve input (pre-captured or via `selection`) → render template → stream → display in `<Detail>` → expose actions (Copy ⌘C, Paste ⌘V, Run Again ⌘R). Aborts the in-flight stream on unmount.

- **`PromptForm`** — Raycast `<Form>` for create/edit. Fields: `name` (text), `template` (textarea), and a hidden `id` (assigned via `slugify` + `uniqueId` for new prompts; immutable for existing). Submits via `prompts.upsert`. Validates that `name` is non-empty.

- **Top-level command wrappers** (`polish`, `translate`, `polite`) — each is a one-line `view`-mode command file that mounts `<StreamingDetail promptId="<id>" />`. They exist to give built-ins their own root-search entry and hotkey slot; they share zero logic with each other beyond the prop string.

- **`run`** view — the *Run Custom Prompt* picker. On mount: capture selection via `selection` (fail-fast toast + exit on `NoInputError`). Then render a `<List>` of all prompts from `prompts.list()`; pressing Enter on a row pushes `<StreamingDetail promptId={row.id} preCapturedText={text} />`.

- **`manage`** view — the *Manage Prompts* command. Renders a `<List>` of all prompts with action panel: Edit (push `<PromptForm>` pre-filled), Duplicate (auto-named "Copy of X", new id), Delete (only on `!isBuiltIn`), Restore Default (only on built-ins where `isEdited(prompt)`, with a confirmation `Alert`), New Prompt (push empty `<PromptForm>`).

### Manifest

- Five commands declared in `package.json`: `polish`, `translate`, `polite`, `run`, `manage`. Command `name` matches the source file name. All five are `mode: "view"`.
- Two **extension-level** preferences (shared by all commands): `apiKey` (`type: password`, `required: true`) and `model` (`type: textfield`, `default: "openai/gpt-4o-mini"`, `required: false`).
- Required Store-grade metadata: `title`, `description`, `author`, `license`, `platforms: ["macOS"]`, populated `categories`, and a 512×512 PNG icon at `assets/icon.png`.

### Schema

The LocalStorage value at key `verto.prompts.v1` is a JSON-serialized array of `Prompt`:

```
{ id: string, name: string, template: string, isBuiltIn: boolean }
```

`defaultTemplate` is **not** stored — it lives in code (the `builtins` catalog) and is consulted only when the user requests Restore Default. This keeps the storage layer free of dead data after upgrades change a built-in's factory wording.

### Template substitution rules

- All occurrences of the literal string `{{text}}` are replaced with the captured text.
- If the template contains no `{{text}}`, the captured text is appended after a blank line. This is the documented forgiving-mode for users who don't want to learn placeholder syntax.

### Selection capture timing

- Top-level commands: capture happens inside `<StreamingDetail>` on mount.
- Picker (`run`): capture happens **at picker launch**, before the `<List>` renders. If `NoInputError`, the command exits with a Failure toast and never shows the list.
- Manage Prompts (`manage`): does **not** read the selection at all.

### LLM call shape

- Single user message containing the rendered template (no system prompt). Rationale: simplicity, and built-in prompts are written as direct instructions that work fine as a user message.
- `stream: true`. Each delta chunk's `choices[0].delta.content` is appended to a running buffer; the buffer is mirrored into the `<Detail>` markdown via a `useState`.
- `AbortController` from the component is passed into the SDK call so unmounting cancels the request.

### Error surfaces

- `NoInputError` → Failure toast at the entry point, command exits.
- LLM errors (network, 401, 429, context length) → caught in `<StreamingDetail>`, displayed in the view as `**Error:** <message>` and surfaced via a Failure toast.
- "Built-in cannot be deleted" → thrown by `prompts.delete()`; surfaced as a Failure toast in the Manage view.
- Restore Default on a non-built-in → guarded at the UI layer (action hidden); also defended in `prompts.restoreDefault()` which throws if no default exists.

### Upgrade behavior

On every `prompts.list()` call (which happens at command launch), any built-in id present in the `builtins` catalog but absent from storage is appended to storage with its current shipped `template`. Existing prompts (built-in or custom) are never overwritten by this merge. This makes shipping a fourth built-in in a future version a one-line code change.

## Testing Decisions

A good test in this codebase tests **external behavior** at the module boundary, not internal implementation details. Tests should:

- Treat the module's public surface as the contract; never reach into private constants, file paths, or internals.
- Use real inputs and assert on real outputs (returned values, thrown error types, side effects observable on the same public surface).
- Avoid asserting "function X was called" except where the side effect *is* the contract (e.g. that `LocalStorage.setItem` is called with a correct serialized value).
- Be tolerant of refactors that change file layout, helper extraction, or internal data flow without changing externally observable behavior.

Modules with tests in the initial PRD scope:

- **`template`** (pure renderer). Cases: single `{{text}}` replacement; multiple occurrences all replaced; placeholder mid-sentence with surrounding punctuation; template with no placeholder appends after blank line; empty input string is replaced as-is; multiline input preserved.

- **`prompts`** (registry CRUD). Cases (with `LocalStorage` mocked): first-call seeds three built-ins; subsequent call returns the seeded array unchanged; `upsert` of an existing id updates fields but preserves `isBuiltIn`; `upsert` of a new id stores it with `isBuiltIn: false`; `delete` of a user prompt removes it; `delete` of a built-in throws; `restoreDefault` on a built-in resets `template` to its factory value and persists; `restoreDefault` on an unknown id throws; `isEdited` returns true only when a built-in's stored template differs from the factory; on second launch with a newer `builtins` catalog containing an additional id, that id is appended to storage and existing prompts are unchanged.

- **`builtins`** (catalog). Contract tests: every entry has all of (`id`, `name`, `template`, `defaultTemplate`); ids are unique; all `defaultTemplate`s contain `{{text}}` (cheap lint to catch a typo'd factory string).

- **`selection`** (fallback chain). With `getSelectedText` and `Clipboard.readText` mocked: returns selection when it's non-empty; falls back to clipboard when selection rejects; falls back to clipboard when selection returns empty/whitespace; throws `NoInputError` when both are empty/whitespace; the thrown error has an actionable message.

UI components (`StreamingDetail`, `PromptForm`, `run`, `manage`, top-level wrappers) are out of scope for unit tests in this PRD. They will be exercised via `npm run dev` (manual loop) until a future PRD introduces React-level tests with a Raycast test harness.

There is no prior art for tests in this repo (greenfield). The skeleton will be standard Node test runner (`node --test`) with TypeScript support via `tsx` or `ts-node`, plus a thin `mocks/` directory for the two Raycast surfaces we touch (`LocalStorage`, `getSelectedText`/`Clipboard.readText`).

## Out of Scope

- Built-in LLM providers other than OpenRouter. (Architecturally, `llm` could grow a provider switch; not on day one.)
- Per-prompt model overrides. The single global `model` preference applies to every prompt.
- Argument-driven Translate (`language` argument). The user explicitly chose to keep target language inside the editable Translate template; if it stops scaling, revisit.
- A "Run on Clipboard" command separate from the picker. Clipboard is already the documented selection fallback.
- Streaming display of intermediate tool calls / function calls. Plain chat completion only.
- Cost / token counters, request history, or per-prompt usage stats.
- Importing or exporting prompts (Markdown, JSON, sharing URLs). Possible follow-up if user demand emerges.
- Custom keywords / aliases for List search beyond Raycast's default substring match.
- Localized UI strings; UI is English only.
- Automated tests for view components or Raycast integration.
- Publishing to the Raycast Store. The manifest is built to pass `ray build`, but the Store submission flow itself is a follow-up.

## Further Notes

- **Why `{{text}}` instead of `{text}` or `$TEXT`**: Mustache-style braces are visually distinct, unlikely to collide with the user's literal prose, and familiar to anyone who's used a templating tool.
- **Why a single user message**: built-in prompts are written as imperative instructions ("Translate the following…") that don't benefit from a system/user split for these short, single-shot tasks. A future power-user feature could add an optional system field to a prompt schema.
- **Why "fail fast on no input" in the picker**: opening a picker that turns out to be useless (because there's nothing to act on) is more disorienting than a clear toast at launch. The `manage` command, which doesn't need input, is the only `view` command that doesn't gate on selection.
- **Why preserve `isBuiltIn` on upsert**: it prevents a malformed form submission from silently flipping a built-in into a user-deletable prompt. This invariant makes the "you can't delete built-ins" rule enforceable from a single check.
- **`HTTP-Referer` and `X-Title` headers on OpenRouter**: OpenRouter uses these to attribute traffic in their dashboards and to enforce per-app analytics. They're optional but recommended.
- **Why no `system` prompt for built-ins**: keeps the schema flat (`{ id, name, template, isBuiltIn }`). Adding `system` later is a non-breaking schema bump that defaults to undefined.
- **Future built-ins are free to ship**: a new built-in is one entry added to the `builtins` catalog plus (optionally) one new top-level command in `package.json`. The upgrade-merge in `prompts.list()` handles seeding for existing installs.
