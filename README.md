# i18n Translator — Figma Multilingual Layout Check (Figma Design Agent edition)

Register this skill in Figma's custom skill feature, select a FRAME or SECTION on the canvas, and run it from the chat. It duplicates the frame once per target language, translates every text node inside each copy — including text nested inside component instances — and leaves the original untouched.

> *日本語 README はこちら: [README.ja.md](README.ja.md)*

---

## What this is

The real multilingual implementation — actual translations, i18n routing, locale switching — belongs in the codebase. What's much harder to see in code review is whether the *layout itself* survives translation: German headlines run long, Japanese sentences pack tight, and a card that fits three lines of English text can overflow, collide, or awkwardly wrap once the copy changes length.

This skill catches that early, before any code is written. Duplicate the frame into whichever target languages you're planning to ship, watch the layout hold up (or not) right there on the canvas, and go into implementation already knowing which sections need auto-layout attention.

```
Select a FRAME / SECTION on canvas
        │
        ▼
Walk tree via Plugin API, don't skip INSTANCE subtrees
        │
        ▼
Auto-detect source language → ask which languages to duplicate into
        │
        ▼
Warn if the frame lacks auto-layout (text-length changes can break it)
        │
        ▼
Translate every text node, present a preview table per language
        │
        ▼
Duplicate + apply: clone, translate, auto-tune text-wrap style & alignment
        │
        ▼
Verify a sample, then report applied/skipped counts + a visual-check reminder
```

`INSTANCE` subtrees are never skipped — button labels, card titles, and nav items dropped inside component instances are exactly the kind of visible copy this skill exists to check.

It also sets Figma's new `textWrapStyle` property (`BALANCE` for headings, `PRETTY` for body copy) on every translated node, to cut down on orphaned words and uneven headings — one less thing to manually clean up before you take a screenshot.

---

## Scope

- **Writes real content.** This isn't a read-only preview — it duplicates the frame and applies the translated text via the Plugin API. The original frame is never touched.
- **Target node types:** `FRAME` or `SECTION` only.
- **Languages:** English, Spanish, French, German, Portuguese, Chinese (Simplified), Japanese, Korean as presets, plus any free-text language.
- **What this does NOT do:** fix layout breakage automatically (it warns instead — see below), adapt numbers/dates/currency to locale conventions, or support right-to-left languages (Arabic, Hebrew, etc. — out of scope for now).
- **Translation quality:** the translations are AI-generated. Treat them as a strong first draft for the layout check, not a final copy — get a native-speaker review per language before anything ships.

**Why no automatic layout-fixing:** whether a frame reflows cleanly when its text changes length depends entirely on how the frame was built (auto-layout with proper sizing vs. fixed dimensions) — not something this skill can safely infer and correct without risking unwanted structural changes. Instead, it warns before duplicating if the frame lacks auto-layout, and reminds you to check visually afterward. That's the point of the tool: surface the risk, don't paper over it.

---

## Requirements

- **Figma (Design Agent / custom skill feature).** This skill only uses Figma's built-in Plugin API script execution tool (e.g. `evaluate_script`). No external MCP servers or API keys required — register `SKILL.md` as a custom skill and it runs.

---

## Repo structure

```
skills/
  i18n-translator/
    SKILL.md          — skill definition to register in Figma's custom skill feature
    LICENSE
```

---

## Getting started

**Once published on Figma Community**, you'll be able to add this skill directly from the [AI Skills library](https://www.figma.com/community/ai-skills) — no download needed. Until then, register it from source:

**From source:** clone this repo and register the skill file yourself.

```bash
git clone https://github.com/gaspanik/i18n-translator-skill
```

**1. Register `skills/i18n-translator/SKILL.md`** in Figma's custom skill feature.

**2. Select a FRAME or SECTION** on the canvas.

**3. Run it from the chat**

```
/i18n-translator
```

```
Duplicate this frame in English, Spanish, and Japanese so I can check the layout
```

```
このフレームを英語版と中国語版に複製して、レイアウトが崩れないか確認したい
```

Larger frames (60+ text nodes) take a little longer — that's normal.

---

## Example output

```
### Result

| Language      | Duplicated frame | Applied | Skipped |
|----------------|-------------------|---------|---------|
| English (en)   | Home_en           | 24 / 26 | 2       |
| Japanese (ja)  | Home_ja           | 26 / 26 | 0       |

Headings got the BALANCE text-wrap style and body copy got PRETTY. If you'd
like to adjust this yourself, select the text layer and change "Wrap style"
in the Typography section of the right panel.

Please visually check every duplicated frame. Areas without auto-layout in
particular may have broken layout depending on how much the translated
text's length changed. The translations themselves are AI-generated as
well — a native-speaker review per language is recommended before shipping.
```

---

## Part of a larger set

This skill is one of a 21-skill bundle, **KMRVID Figma Skills**, covering AI-slop-resistant page generation, multi-layout-pattern exploration, layer cleanup, contrast/accessibility checks, tokenization, component audits, ALT text suggestions, and more: [gaspanik.gumroad.com/l/kmrvid-figmaskills](https://gaspanik.gumroad.com/l/kmrvid-figmaskills)

---

Built by Masaaki Komori - [@cipher](https://x.com/cipher) · Skill for [Figma](https://www.figma.com/)
