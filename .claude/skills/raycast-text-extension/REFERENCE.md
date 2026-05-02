# Raycast Text Extension — Reference

## Manifest

`package.json` is the extension manifest. Top-level required fields: `name`, `title`, `description`, `icon`, `author`, `platforms`, `categories`, `commands`.

### Command properties

| Field         | Required | Purpose                                                                |
| ------------- | -------- | ---------------------------------------------------------------------- |
| `name`        | yes      | Maps to entry file: `name: "rewrite"` → `src/rewrite.ts(x)`            |
| `title`       | yes      | Shown in Store and root search                                         |
| `description` | yes      | One-line description in root search                                    |
| `mode`        | yes      | `"view"` \| `"no-view"` \| `"menu-bar"`                                |
| `icon`        | no       | PNG, 512×512, optional `@dark` variant; defaults to extension icon     |
| `subtitle`    | no       | Secondary line in root search                                          |
| `interval`    | no       | For background commands: `"30s"`, `"1m"`, `"12h"` (no-view/menu-bar)   |
| `arguments`   | no       | Up to 3 inline args entered before launching (text/password/dropdown)  |

### When to use each mode

- **`no-view`** — silent commands: read selection, do work, paste/HUD. No UI rendered. Best for "act on selected text" extensions.
- **`view`** — render a `<List>`, `<Detail>`, `<Form>`, or `<Grid>`. Use when you need to display results, accept multi-step input, or stream output.
- **`menu-bar`** — persistent status-bar item rendered with `<MenuBarExtra>`. Independent of the user's current selection.

### Preferences (per-command or per-extension)

```json
"preferences": [
  {
    "name": "apiKey",
    "title": "API Key",
    "description": "Your provider API key",
    "type": "password",
    "required": true
  },
  {
    "name": "model",
    "title": "Model",
    "description": "Model to use",
    "type": "dropdown",
    "data": [
      { "title": "Fast", "value": "fast" },
      { "title": "Smart", "value": "smart" }
    ],
    "default": "smart",
    "required": false
  }
]
```

Read in code with full TypeScript inference:

```typescript
import { getPreferenceValues } from "@raycast/api";
const { apiKey, model } = getPreferenceValues<Preferences.Rewrite>();
```

### Arguments (inline input before launch)

```json
"arguments": [
  { "name": "language", "type": "text", "placeholder": "Target language", "required": true }
]
```

```typescript
import { LaunchProps } from "@raycast/api";
export default function Command(props: LaunchProps<{ arguments: Arguments.Translate }>) {
  const { language } = props.arguments;
}
```

Limits: max 3 arguments per command; argument order is significant in root search.

## Selected text & fallbacks

`getSelectedText()` reads the selection from the **frontmost** app:

```typescript
import { getSelectedText, Clipboard } from "@raycast/api";

async function readInput(): Promise<string> {
  try {
    return await getSelectedText();
  } catch {
    const clip = await Clipboard.readText();
    if (clip) return clip;
    throw new Error("Select text or copy something to the clipboard first");
  }
}
```

Failure causes (rejection):

- Nothing is selected.
- The frontmost app does not expose its selection via the macOS Accessibility API (some Electron / sandboxed apps).
- Raycast lacks Accessibility permission. First launch prompts the user; if denied, every call throws. Document this in `README.md`.

## Clipboard

```typescript
import { Clipboard } from "@raycast/api";

await Clipboard.copy("text");                       // place on clipboard
await Clipboard.copy("secret", { concealed: true }); // skip clipboard history
await Clipboard.copy({ file: "/path/to.pdf" });     // copy a file

await Clipboard.paste("text");                       // paste into frontmost app at cursor

const { text, file, html } = await Clipboard.read(); // read everything
const text = await Clipboard.readText();              // text only
const previous = await Clipboard.readText({ offset: 1 }); // history (0–5)

await Clipboard.clear();
```

## User feedback

```typescript
import { showHUD, showToast, Toast } from "@raycast/api";

// Fire-and-forget banner; closes Raycast first
await showHUD("Done");

// In-app feedback
await showToast({ style: Toast.Style.Success, title: "Saved" });
await showToast({ style: Toast.Style.Failure, title: "Failed", message: String(err) });

// Animated + mutate to terminal state
const toast = await showToast({ style: Toast.Style.Animated, title: "Rewriting…" });
try {
  await doWork();
  toast.style = Toast.Style.Success;
  toast.title = "Rewrote";
} catch (err) {
  toast.style = Toast.Style.Failure;
  toast.title = "Failed";
  toast.message = String(err);
}
```

`showToast` automatically falls back to `showHUD` when the Raycast window has closed.

## Debugging

| Tool                   | When to reach for it                                                            |
| ---------------------- | ------------------------------------------------------------------------------- |
| `console.log/debug/error` | First-line debugging. Output streams in the `npm run dev` terminal.          |
| Error overlay          | Unhandled exceptions / rejections render full stack traces during dev.          |
| React DevTools         | `view`-mode commands. `npm i -D react-devtools@6.1.1`, then `⌘⌥D` in Raycast.   |
| VS Code debugger       | Install Raycast VS Code extension → "Attach Debugger" → set breakpoints.        |
| Extension Issues       | After publishing: <https://www.raycast.com/extension-issues> shows prod errors. |

In dev mode, Raycast caches built JS — restart `npm run dev` if module-level changes don't take effect.

## Build & publish

```bash
npm run lint     # ESLint with @raycast/eslint-config
npm run build    # type-check + manifest validation + asset checks (matches Store)
npm run publish  # open a PR to raycast/extensions
```

Store checklist:

- 512×512 PNG icon at `assets/icon.png` (optional `assets/icon@dark.png`).
- `README.md` describing the extension, screenshots in `metadata/`.
- `categories` populated in `package.json`.
- All `preferences` have a `description`.
- No console errors when running the dev build.
- Accessibility permission caveats noted if you use `getSelectedText`.

## Useful CLI

```bash
npx ray create-extension <name>   # scaffold
npx ray develop                    # same as npm run dev
npx ray build                      # same as npm run build
npx ray lint                       # same as npm run lint
npx ray publish                    # open Store PR
```
