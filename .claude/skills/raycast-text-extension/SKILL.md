---
name: raycast-text-extension
description: Build, debug, and publish a Raycast extension whose command takes the user's selected text and transforms or acts on it (translate, rewrite, lookup, send, etc.). Use when the user is creating a Raycast extension command, working with `getSelectedText`, scaffolding an extension via `ray create-extension`, picking a command `mode` (view / no-view / menu-bar), wiring preferences or arguments, debugging with `npm run dev`, or preparing an extension for the Raycast Store.
---

# Raycast Text Extension

Build a Raycast extension command that reads the user's currently selected text from the frontmost app, runs an operation on it (transform, send to API, lookup), and returns the result via paste, clipboard, or HUD.

## Quick start

```bash
# 1. Scaffold (run inside the parent directory you want)
npx ray create-extension my-ext

# 2. Install + start dev server (hot reload + error overlay + console logs)
cd my-ext && npm install && npm run dev
```

Minimal `no-view` command that uppercases selected text and pastes it back:

```typescript
// src/index.ts
import { Clipboard, getSelectedText, showHUD, showToast, Toast } from "@raycast/api";

export default async function Command() {
  try {
    const selected = await getSelectedText();
    await Clipboard.paste(selected.toUpperCase());
    await showHUD("Uppercased");
  } catch (err) {
    await showToast({ style: Toast.Style.Failure, title: "No text selected", message: String(err) });
  }
}
```

`package.json` command entry (the `name` must match the file `src/<name>.ts`):

```json
{
  "commands": [
    { "name": "index", "title": "Uppercase Selection", "description": "Uppercase the selected text", "mode": "no-view" }
  ]
}
```

## Workflow

1. **Pick command `mode`** — `no-view` for silent transformations (paste/HUD only), `view` for showing a React UI (List/Detail/Form), `menu-bar` for a persistent status-bar item. For "act on selected text" the default is `no-view`; switch to `view` only if you need to show results, options, or a streaming response.
2. **Wire selected-text input** — `await getSelectedText()` rejects if nothing is selected. Fall back to `Clipboard.readText()` or a command `argument` so the command still works without a selection. See [REFERENCE.md](REFERENCE.md#selected-text--fallbacks).
3. **Return output** — `Clipboard.paste(text)` inserts into the active app; `Clipboard.copy(text)` puts it on the clipboard; `showHUD("…")` flashes a message; `showToast({ style, title })` is interactive while Raycast is open.
4. **Add config** — secrets/options go in `preferences` (read with `getPreferenceValues<Preferences.CommandName>()`). User-typed input goes in `arguments` (read via `props.arguments`). See [REFERENCE.md](REFERENCE.md#manifest).
5. **Debug** — `console.log/debug/error` prints to the dev terminal. Unhandled rejections show as an error overlay. For React commands, install `react-devtools@6.1.1` and press `⌘⌥D`. For breakpoints, attach the VS Code Node debugger.
6. **Build & lint** — `npm run build` does type checks + manifest checks + asset checks (the same checks the Store runs). `npm run lint` runs ESLint with the Raycast config.
7. **Publish** — `npm run publish` opens a PR against `raycast/extensions`. Required: 512×512 PNG icon (with optional `@dark` variant), `README.md`, populated `categories` / `description` / `author`. See [REFERENCE.md](REFERENCE.md#publishing).

## Common patterns

- **`getSelectedText` failure handling** — it throws when no app is frontmost, the app blocks accessibility, or nothing is selected. Always wrap in try/catch and offer a fallback (clipboard, argument, or actionable toast).
- **macOS Accessibility permission** — first run prompts for Accessibility access for Raycast. If selection always fails on a real install, that's the cause; surface it in the README.
- **Async work + feedback** — start a `Toast.Style.Animated` toast, `await` the work, then mutate `toast.style`/`toast.title` to `Success`/`Failure`. `showToast` falls back to `showHUD` automatically when the Raycast window closed.
- **Long-running AI calls** — keep the command `no-view` only if it finishes in a few seconds. For streaming output use `mode: "view"` with a `<Detail>` and update `markdown` as chunks arrive.

## See also

- [REFERENCE.md](REFERENCE.md) — manifest fields, full APIs (selected text, clipboard, toast, preferences, arguments), debugging matrix, publishing checklist.
- [EXAMPLES.md](EXAMPLES.md) — three complete commands: uppercase (no-view), AI rewrite with preference (no-view), translate-to-language with argument fallback (view).
