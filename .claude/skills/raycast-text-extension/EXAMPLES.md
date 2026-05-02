# Raycast Text Extension — Examples

Three complete, copy-pasteable examples covering the common shapes.

## 1. Uppercase selection (no-view, no config)

The simplest shape. Reads selection, transforms, pastes back, shows HUD.

`package.json` (relevant fragment):

```json
{
  "name": "uppercase",
  "title": "Uppercase",
  "description": "Uppercase the selected text",
  "icon": "icon.png",
  "author": "you",
  "platforms": ["macOS"],
  "categories": ["Productivity"],
  "commands": [
    {
      "name": "index",
      "title": "Uppercase Selection",
      "description": "Uppercase the currently selected text",
      "mode": "no-view"
    }
  ]
}
```

`src/index.ts`:

```typescript
import { Clipboard, getSelectedText, showHUD, showToast, Toast } from "@raycast/api";

export default async function Command() {
  let input: string;
  try {
    input = await getSelectedText();
  } catch {
    await showToast({
      style: Toast.Style.Failure,
      title: "No selection",
      message: "Highlight some text first",
    });
    return;
  }

  await Clipboard.paste(input.toUpperCase());
  await showHUD("Uppercased");
}
```

## 2. AI rewrite with API-key preference (no-view)

Uses a `password` preference for an API key, an animated toast for in-flight feedback, and pastes the result.

`package.json` (commands + preferences):

```json
{
  "commands": [
    {
      "name": "rewrite",
      "title": "Rewrite Selection",
      "description": "Rewrite the selected text more clearly",
      "mode": "no-view"
    }
  ],
  "preferences": [
    {
      "name": "apiKey",
      "title": "API Key",
      "description": "Your provider API key",
      "type": "password",
      "required": true
    },
    {
      "name": "tone",
      "title": "Tone",
      "description": "Default rewrite tone",
      "type": "dropdown",
      "default": "neutral",
      "required": false,
      "data": [
        { "title": "Neutral", "value": "neutral" },
        { "title": "Concise", "value": "concise" },
        { "title": "Friendly", "value": "friendly" }
      ]
    }
  ]
}
```

`src/rewrite.ts`:

```typescript
import {
  Clipboard,
  getPreferenceValues,
  getSelectedText,
  showToast,
  Toast,
} from "@raycast/api";

async function rewrite(text: string, tone: string, apiKey: string): Promise<string> {
  // Replace with your provider call. Kept generic to stay self-contained.
  const res = await fetch("https://api.example.com/rewrite", {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${apiKey}` },
    body: JSON.stringify({ text, tone }),
  });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  const data = (await res.json()) as { output: string };
  return data.output;
}

export default async function Command() {
  const { apiKey, tone } = getPreferenceValues<Preferences.Rewrite>();

  let input: string;
  try {
    input = await getSelectedText();
  } catch {
    const clip = await Clipboard.readText();
    if (!clip) {
      await showToast({ style: Toast.Style.Failure, title: "No selection or clipboard text" });
      return;
    }
    input = clip;
  }

  const toast = await showToast({ style: Toast.Style.Animated, title: "Rewriting…" });
  try {
    const output = await rewrite(input, tone, apiKey);
    await Clipboard.paste(output);
    toast.style = Toast.Style.Success;
    toast.title = "Rewrote";
  } catch (err) {
    toast.style = Toast.Style.Failure;
    toast.title = "Rewrite failed";
    toast.message = String(err);
  }
}
```

## 3. Translate with argument fallback (view)

A `view` command that streams translated output into a `<Detail>`. Falls back to a required `language` argument when no text is selected.

`package.json` fragment:

```json
{
  "commands": [
    {
      "name": "translate",
      "title": "Translate Selection",
      "description": "Translate the selected text",
      "mode": "view",
      "arguments": [
        {
          "name": "language",
          "type": "text",
          "placeholder": "Target language (e.g. French)",
          "required": true
        }
      ]
    }
  ]
}
```

`src/translate.tsx`:

```tsx
import { Action, ActionPanel, Detail, getSelectedText, LaunchProps, showToast, Toast } from "@raycast/api";
import { useEffect, useState } from "react";

export default function Translate(props: LaunchProps<{ arguments: Arguments.Translate }>) {
  const { language } = props.arguments;
  const [markdown, setMarkdown] = useState("Loading…");
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    (async () => {
      try {
        const input = await getSelectedText();
        const translated = await translate(input, language); // your impl
        setMarkdown(translated);
      } catch (err) {
        await showToast({ style: Toast.Style.Failure, title: "Translate failed", message: String(err) });
        setMarkdown(`**Error:** ${String(err)}`);
      } finally {
        setIsLoading(false);
      }
    })();
  }, [language]);

  return (
    <Detail
      isLoading={isLoading}
      markdown={markdown}
      actions={
        <ActionPanel>
          <Action.CopyToClipboard content={markdown} />
          <Action.Paste content={markdown} />
        </ActionPanel>
      }
    />
  );
}

async function translate(text: string, language: string): Promise<string> {
  // Stub — replace with your translation call.
  return `[${language}] ${text}`;
}
```

## Patterns worth lifting

- **Selection → fallback to clipboard → fallback to argument.** Three fallbacks cover almost every "no text available" failure mode.
- **Animated toast → mutate to terminal state.** Keep the user oriented during async work without rendering a full view.
- **`Clipboard.paste` vs `Clipboard.copy`.** Paste inserts at cursor in the previously frontmost app; copy only puts it on the clipboard. For "act on selection" UX, paste is usually right.
- **`view` only when you need it.** A no-view command + HUD is faster and feels more like a native shortcut than a window opening for a one-shot transformation.
