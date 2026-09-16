---
title: Translating WhatsApp Web from the inside — a hackathon extension against a hostile DOM
date: 2026-08-20
description: ChatTranslator adds live translation and chat summaries to WhatsApp Web. The model calls were the easy part. Reading and writing a page that doesn't want to be scripted was the real work.
tags: [web, automation, ai, javascript]
---

[ChatTranslator](https://github.com/VirajAnand-02/ChatTranslator-) is a Chrome extension that puts a translation under every WhatsApp Web message, lets you write in your own language and send in theirs, and summarises a long group chat when you've been away. We built it as a team over a hackathon weekend at the end of March 2025.

WhatsApp has no public API for your personal chats, so the extension works entirely through the page. That choice shaped almost every line of code.

## The architecture is three processes

A Manifest V3 extension is really three programs that communicate by passing messages:

- **Content script** (`content.js`). Runs inside `web.whatsapp.com`. Finds messages, draws the translation overlays, and types into the chat box.
- **Service worker** (`background.js`). Owns the settings and every model call. Content scripts don't make network or model calls directly.
- **Popup and options pages.** Settings, "translate and send", and summary controls.

With every model call in the service worker, the content script doesn't need to care which model is behind a translation. It sends `{ text, targetLanguage }` and gets a string back.

## Reading messages you didn't render

WhatsApp Web is a React app with generated class names like `_ao3e` and `x1iyjqo2`, and they change without notice. No single selector survives for long, so the content script never depends on one:

```js
const textElement =
  bubble.querySelector("span._ao3e.selectable-text.copyable-text > span") ||
  bubble.querySelector("span[dir='ltr']._ao3e.selectable-text.copyable-text") ||
  bubble.querySelector("span.selectable-text.copyable-text:not([title])") ||
  bubble.querySelector('[data-testid="caption"]');
  // …and finally: the deepest element with more than a few characters of text
```

The chains go from the most specific selector to the most general. The last fallback throws out selectors altogether and looks for the deepest element in the bubble that contains real text. Stable hooks like `data-testid`, `data-icon="document"` and `data-pre-plain-text` are preferred wherever they exist, because they are part of how the page works, not how it looks.

New messages are found with a `MutationObserver` on the conversation pane. It only checks whether nodes were added, then waits 100ms before scanning:

```js
if (shouldCheckMessages) {
  clearTimeout(window.messageCheckTimeout);
  window.messageCheckTimeout = setTimeout(checkForNewMessages, 100);
}
```

Opening a chat inserts hundreds of nodes in one burst. Without the delay, that would be hundreds of scans and, much worse, hundreds of model calls. A second observer watches the chat header, so switching conversations resets the state.

### Knowing which messages you've already seen

Scanning is only safe if the same message is never translated twice. WhatsApp doesn't reliably expose a message id in the DOM, so the extension builds its own from what it can see:

```js
const position = Array.from(bubble.parentNode.children).indexOf(bubble);
return `${position}_${timestamp || ""}_${content.length}_${content.substring(0, 20)}`;
```

The timestamp comes from `data-pre-plain-text`, which WhatsApp uses for its own copy-to-clipboard feature. This was good enough for a demo. It is also the weakest part of the design: the position changes when older messages load above the current ones, and I'd now build the id from the timestamp, the sender and a hash of the whole message text instead.

## Writing into a Lexical editor

Sending a translated message turned out to be harder than reading one. The chat box isn't a `<textarea>`. It is a [Lexical](https://lexical.dev/) editor, which keeps its own document model and treats the DOM as output. If you set `textContent` on it, the text appears and then disappears on the next render, or stays on screen while the send button stays disabled.

What worked was building exactly the structure Lexical would have produced itself, and then firing the event it listens for:

```js
const span = document.createElement("span");
span.className = "selectable-text copyable-text xkrh14z";
span.setAttribute("data-lexical-text", "true");
span.textContent = text;
// …into a <p dir="ltr"> inside the contenteditable, then:
chatInput.dispatchEvent(new Event("input", { bubbles: true }));
```

Before that runs, an XPath lookup for the input paragraph is tried first, then two selectors, then "the last `contenteditable` on the page" as a last resort. That's four strategies for one text box. When you script a page you don't control, most of the code ends up being fallbacks.

## Models: on-device first, cloud as fallback

Translation goes to one of three backends:

1. **Gemini Nano, on device**, through Chrome's built-in Prompt API. No network, no key, no chat text leaving the machine. That matters a lot when the text is someone's private messages.
2. **The Gemini API** (`gemini-2.0-flash`) for browsers where Nano isn't available.
3. **Vertex AI**, for the same model inside a Google Cloud project.

On startup the service worker tries to create a Nano session. If that fails, it switches the saved preference so the UI shows what is actually running. If a single local call fails, that one request retries against the Gemini API, so one bad call doesn't show up as a broken message.

The prompt matters more than the model here. Chat messages are short, full of slang and very often already in your language, so the system prompt asks for a casual tone, no commentary, and a strict JSON reply:

```json
{ "translatedText": "…", "failed": "true / false" }
```

Models often wrap that JSON in a Markdown code fence anyway, so the parser strips a fenced block before calling `JSON.parse`. If parsing still fails, it shows the raw reply instead of an error. A slightly messy translation beats an empty overlay.

## Summaries you choose what goes into

The summary feature doesn't read the whole chat history. That would mean scrolling through it automatically, and WhatsApp only renders what is on screen. Instead, **Summarize Chat** starts recording: messages already on screen are captured immediately, and every message you scroll past is added too. A floating **Finish & Summarize** button sends what you recorded to the model with a separate prompt asking for topics, decisions, deadlines and open questions.

This came from a limitation, but it turned out to be the better design. You decide exactly what goes into the summary and what gets sent to a model.

## What I'd do differently

- **Treat selectors as data.** Selectors spread across a 1,900-line content script break one at a time, and each break looks different. One table of selectors with a single health check ("can I find a message bubble? the input box?") would turn silent failures into one clear warning.
- **Ship the cache or delete it.** There's a SQLite translation cache (sql.js in a worker) in the repo that was never fully connected. The idea is right: group chats repeat "ok", "haha" and "on my way" constantly. A small `chrome.storage` map would have delivered most of the benefit during the hackathon.
- **Configuration only in settings.** Endpoints, project ids and credentials belong in the options page and `chrome.storage`, never in the bundle. Anything shipped inside an extension can be read by anyone who installs it.
- **Track the Prompt API.** Chrome's built-in AI APIs have been renamed and reshaped since then. The `chrome.ai.languageModel` path we used has since been replaced by a global `LanguageModel`, so feature detection should check for both.

For a weekend project, the part I'm happiest with is the on-device default. A translator for private chats should keep them on your machine unless you choose otherwise.
