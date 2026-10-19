---
name: i18n-translator
description: >-
  Duplicates the selected FRAME or SECTION once per target language and
  translates every text node inside each copy — including text nested inside
  component instances — via the Plugin API. Intended as a pre-implementation
  layout/display check before the real multilingual build is coded — not
  the implementation itself. Auto-detects the source language from the
  text content itself; presets cover English, Spanish, French, German,
  Portuguese, Chinese (Simplified), Japanese, and Korean, plus free-text
  custom languages. Also auto-tunes text-wrap style and resets
  Japanese-specific alignment for readability. Warns before applying if
  the target lacks auto-layout (text-length changes can break the layout
  otherwise), and reminds the user to visually check every duplicated
  frame afterward. Figma Design Agent edition. Part of KMRVID Figma
  Skills, a 31-skill bundle: kmrvidskills.dev/en/figma/
---

# i18n Translator — Figma Multilingual Layout Check (Figma edition)

**Ver:** ver.202610191253

Duplicate the selected FRAME or SECTION once per requested target language, and translate every text node inside each copy — including text nested inside component instances — while leaving the original untouched.

**Output language:** All user-facing output is English. If the user writes in another language, follow their language instead. This is independent of the *target* languages being translated into — those are chosen explicitly in Step 3.

**Environment:** This skill runs inside Figma's agent environment. The target comes from the current canvas selection, and all reads/writes go through the Plugin API script execution tool (e.g. `evaluate_script`). No URLs, no external MCP tools — every operation below runs as one self-contained script, with the necessary Plugin API recipe (font loading, cloning, reparenting) written out inline rather than delegated to anything else.

## Scope

**Positioning: this is a pre-implementation check, not the multilingual implementation itself.** A real production build's actual translations and i18n routing belong in the codebase. This skill exists so a designer can see, before any code is written, how the layout holds up once text gets longer/shorter across languages — catch that early here, then build the real thing in code.

This skill does exactly two things: **duplicate** and **translate text content**. It deliberately does not:

- Fix layout breakage caused by translated text being longer/shorter than the source (Step 4 warns about this instead — see "Why no auto-fix" below)
- Adapt numbers, dates, or currency to each locale's formatting conventions
- Support right-to-left languages (Arabic, Hebrew, etc.) — not in the v1 preset, and RTL text direction / mirrored layout is out of scope
- Translate anything other than TEXT node `characters` — images, icons, and non-text content are copied as-is, untranslated

**Why no auto-fix for layout breakage:** whether a frame reflows cleanly when its text changes length is a property of how the frame was built (auto-layout with `HUG`/`FILL` sizing vs. fixed dimensions), not something this skill can safely infer and correct without risking unwanted structural changes to the user's design. Instead, this skill warns clearly before applying (Step 4) and reminds the user to check visually after (Step 8). Treat this as a firm design decision, not a gap to fill in — don't add resizing/reflow logic on your own initiative.

**Two small, in-the-box exceptions to that rule (Step 6):** `textWrapStyle` and alignment resets never change a node's size or any ancestor's layout — they only change how text sits within the box it already has — so they don't reopen the "no auto-fix" decision above:

- **Text wrap style.** Figma's `textWrapStyle` property (`'AUTO'`/`'BALANCE'`/`'PRETTY'`, added 2026-08-14) directly targets the orphaned-word and uneven-heading problems a translation-driven line-break change commonly introduces — exactly the kind of thing this skill would otherwise be leaving for the user to notice and fix by hand. Set on every successfully-translated node in Step 6.
- **Alignment reset for Japanese-specific justification.** Japanese body text commonly uses JUSTIFIED (均等揃え) alignment, a convention that doesn't read as intentional once the text is some other language — a duplicate leaving Japanese should switch those nodes to `LEFT` rather than silently carrying the source alignment over.

**Target node types:** FRAME or SECTION only. If the selection is a `COMPONENT`, `COMPONENT_SET`, or anything else, tell the user this skill only handles frames and sections (translating a component's master would affect every instance across the file, which is out of scope here), and ask them to select one.

**Do NOT skip `INSTANCE` subtrees.** Some structural-cleanup tools skip instances, since a rename or variable binding inside one only creates a diverging local override. Here the opposite is true: instances routinely contain the actual visible copy a user needs localized (button labels, card titles, nav items), so skipping them would silently leave large parts of the UI untranslated. Walk into every instance and translate its text nodes like any other. An instance's inherited children can't be added or removed, but their existing properties — including `characters` — can be changed freely. Expect a higher failure rate inside instances than elsewhere (locked layers, a text property that isn't exposed for override) — catch and report those individually rather than letting one failure abort the whole pass (see Step 6).

---

## Step 1: Resolve the target

Use the current canvas selection (`figma.currentPage.selection`). If nothing is selected, ask the user to select the target frame or section on the canvas.

Check the type: `FRAME` or `SECTION` → continue; anything else → ask for a valid frame/section selection (see Scope). If more than one node is selected, tell the user this skill duplicates a single frame or section at a time and ask them to select just one.

---

## Step 2: Scan text content

In one `evaluate_script` call (read-only), walk the selected node's full tree — including into `INSTANCE` subtrees, per the Scope note above — and for every `TEXT` node collect:

    {
      index,      // position in a stable depth-first pre-order walk (e.g. via a recursive children-first traversal) —
                   // this is how translations get matched back to nodes after cloning in Step 6
      characters,  // current text content
      nodePath,    // e.g. "HeroSection > CTAButton > Label", for translation context and the confirmation table
      locked,      // node.locked — locked nodes are still listed but flagged, since they'll likely fail to translate in Step 6
    }

The traversal order matters: this exact same walk, run again on a freshly cloned copy of the same tree in Step 6, must visit nodes in the identical order, since clone() produces a structurally identical tree. Don't sort or filter the collected list afterward.

Also record, cheaply, the root node's `layoutMode` and its immediate children's `layoutMode` (one level deep only, not a full-tree audit) for the Step 4 warning.

If zero text nodes are found, tell the user there's nothing to translate and stop.

---

## Step 3: Ask for target languages

Auto-detect the source language yourself by reading the collected `characters` — you're doing the translating, so judge the language directly from the content; no separate detection step is needed. Mention the detected source language briefly so the user can correct you if a frame's placeholder text made it ambiguous.

Present the preset as a single informational prompt with free text welcomed:

> **Which languages should this be duplicated into? (multi-select, anything not listed can be typed in freely)**
> - **English**
> - **Spanish**
> - **French**
> - **German**
> - **Portuguese**
> - **Chinese, Simplified**
> - **Japanese**
> - **Korean**
> - **Other (free text)** — any language not listed above (e.g. Thai, Italian)

Skip offering a language that matches the auto-detected source language (translating a frame into its own language is a no-op) unless the user explicitly asks for it anyway.

For each confirmed target language, resolve an ISO 639-1-based code for the layer-name suffix (Step 5): `en`, `es`, `fr`, `de`, `pt`, `zh-CN`, `ja`, `ko`. For a custom language, pick the standard ISO 639-1 code yourself (or a sensible short code if none exists) and show it in the Step 5 confirmation table so the user can correct it before anything is applied.

If the user names zero languages, ask again rather than proceeding with nothing to do.

---

## Step 4: Auto-layout warning

Using the `layoutMode` data gathered in Step 2, check whether the root node itself has auto-layout enabled (`layoutMode !== 'NONE'`).

**If the root has no auto-layout**, report it as its own message, then force a pause before continuing — a message that flows straight into the next step's question can get folded away and missed in Figma's chat UI. Ask via `ask_user_question`, exactly 2 options — never fewer, since a single-option call fails in this environment and the failed call still renders before the retry, producing a confusing duplicate-looking question:

> This frame doesn't have auto-layout set up. If the translated text ends up longer or shorter than the source, the layout may break.
> - **Continue anyway**
> - **Stop for now** — I'd like to check the auto-layout setup in Figma first

If "Stop for now" is chosen, stop here — don't proceed to Step 5.

**If the root has auto-layout** but one or more immediate children don't, mention it as a lighter informational note (no forced pause needed, since it's not a blocking concern) and continue — e.g. "The root has auto-layout, but some of its children don't. Please check those areas visually after duplication for any breakage."

**If everything checked has auto-layout**, skip this warning entirely and move on silently.

Don't extend this into a deeper structural audit (per-text-node vertical sizing, nested frame checks, etc.) — see "Why no auto-fix" above. This step exists so the user isn't caught by surprise, not to diagnose every node.

---

## Step 5: Translate and present for confirmation

Translate every collected text node's `characters` into each confirmed target language. Use `nodePath` to judge tone and register (a nav label and a paragraph of body copy shouldn't share a voice), and preserve line breaks and obvious placeholders (e.g. `{{name}}`, `%s`). Numbers, proper nouns, and inline formatting are translated on a best-effort basis — per the agreed scope, don't build special protection logic for these; a careful direct translation plus the Step 8 visual-check reminder is the intended safety net, not automated guarding.

Present one confirmation table per target language:

### Translation preview — English (en)

| # | Node | Source | Translated |
|---|------|--------|------------|
| 1 | HeroSection > Heading | 新商品が入荷しました | New arrivals are here |
| 2 | HeroSection > CTAButton > Label | 詳しく見る | Learn more |

If the combined node count is large (roughly 30+), group each language's table by section/region so the user can scan in chunks instead of one long list.

**Do not follow the table directly with a free-text "yes" prompt.** In this environment, a table shown immediately before waiting on free-form "yes" text can lead the agent to apply changes without the user ever actually confirming. Ask explicitly via `ask_user_question`, exactly 2 options:

> Apply all of the above (duplicate + translate)?
> - **Apply all**
> - **Apply only some** — specify which languages or row numbers to apply in your next message (e.g. "English only", "skip row 3 in Japanese")

If "Apply only some" is chosen, wait for the user's next message and use it to filter which languages/rows proceed to Step 6.

---

## Step 6: Apply — duplicate and translate

For each approved target language, run one `evaluate_script` call:

    const original = await figma.getNodeByIdAsync(rootId);
    const clone = original.clone();

    // Reparent before any other mutation: changing a clone's properties
    // before it's safely placed risks losing the clone entirely if a later
    // line throws and the whole script rolls back. Appending to the current
    // page (rather than trying to re-insert next to the original) also keeps
    // every language variant easy to find as its own top-level frame.
    figma.currentPage.appendChild(clone);

    // Now safe to reposition and rename.
    clone.x = original.x + (original.width + GAP) * (languageIndex + 1);
    clone.name = `${original.name}_${langCode}`; // e.g. "Home" -> "Home_en"

    const textNodes = clone.findAll(n => n.type === 'TEXT');
    // Must line up 1:1 with the ordered list collected in Step 2 — same
    // traversal, same tree shape, since clone() is a structurally exact copy.

    const skipped = [];
    for (let i = 0; i < textNodes.length; i++) {
      const node = textNodes[i];
      const newText = translations[i]; // resolved from Step 5's approved table for this language
      if (newText == null) continue; // row was skipped by the user

      if (node.locked) {
        skipped.push({ index: i, nodePath: paths[i], reason: 'locked node' });
        continue;
      }

      try {
        // Canonical text-edit recipe: every font run actually used in the
        // node must be loaded before its characters can be changed. A node
        // can carry more than one font run even without figma.mixed showing
        // on fontName, so always resolve via getStyledTextSegments rather
        // than assuming a single uniform font.
        const segments = node.getStyledTextSegments(['fontName']);
        for (const seg of segments) {
          await figma.loadFontAsync(seg.fontName);
        }
        // Whole-node replacement: any mixed inline styling within the
        // original string (e.g. one bold word mid-sentence) collapses to
        // the first run's style. This is a known, accepted limitation
        // (see Step 8) — don't try to preserve per-character styling
        // across a full text replacement.
        node.characters = newText;
      } catch (e) {
        skipped.push({ index: i, nodePath: paths[i], reason: String(e) });
        continue; // don't attempt the enhancements below on a node whose text didn't even apply
      }

      // Enhancement 1: text-wrap style. Requires the font to already be
      // loaded, which the block above just did. Headings/short strings get
      // BALANCE — Figma's own docs cap it at 6 wrapped lines or under, so it
      // isn't reliable for longer body copy, which gets PRETTY instead. This
      // is a best-effort enhancement, not part of the translation itself —
      // caught separately so a failure here (e.g. running against a Figma
      // version that predates this property) never counts as a translation
      // failure.
      try {
        const isHeadingLike = /heading|title|hero|cta/i.test(paths[i]) || newText.length <= 50;
        node.textWrapStyle = isHeadingLike ? 'BALANCE' : 'PRETTY';
      } catch (e) {
        // Silently skip — nice-to-have, not a translation failure.
      }

      // Enhancement 2: language-appropriate alignment. Only fires when the
      // Step 3 source-language detection was Japanese and this clone's
      // target isn't — never touches alignment in any other direction (e.g.
      // a non-Japanese source, or Japanese as the target itself).
      if (sourceLanguageIsJapanese && langCode !== 'ja' && node.textAlignHorizontal === 'JUSTIFIED') {
        node.textAlignHorizontal = 'LEFT';
      }
    }

    return { langCode, cloneId: clone.id, cloneName: clone.name, appliedCount: textNodes.length - skipped.length, skipped };

Notes specific to this skill:

- **Don't call `resize()` on the clone or any of its descendants.** This skill only ever sets `.characters` — never manually resize anything. If a frame has proper auto-layout, text changes reflow it automatically; if it doesn't, that's exactly what Step 4 already warned about, and forcing a resize here would be exactly the kind of layout auto-fix this skill deliberately avoids.
- **Instance text failures are expected, not exceptional.** A locked instance layer or a text property that isn't exposed for override will throw — catch it per-node (as above) and keep going. Don't let one failed node abort the whole language's pass.
- **`sourceLanguageIsJapanese`** is just the Step 3 auto-detection result carried into this script — resolve it once before looping over languages, don't re-detect per node.
- **Space the clones out** — offsetting `x` by `original.width + GAP` per language (a few hundred px is plenty) keeps every duplicated frame visible and separately selectable afterward.

---

## Step 7: Verify

Sample 2–3 translated nodes per language and confirm the change actually persisted (`node.characters` reads back as the translated string, not the original) before trusting the applied count from Step 6. If a sample doesn't match, treat that language's whole result as suspect and re-check rather than reporting success.

---

## Step 8: Report

### Result

| Language | Duplicated frame | Applied | Skipped |
|----------|-------------------|---------|---------|
| English (en) | Home_en | 24 / 26 | 2 |
| Japanese (ja) | Home_ja | 26 / 26 | 0 |

**Needs manual review** (breakdown of skips):

| Language | Node | Reason |
|----------|------|--------|
| English (en) | HeroSection > Badge > Label | locked node |
| English (en) | Card3 > Instance > Title | non-overridable text property |

Briefly mention what was auto-adjusted so the user doesn't re-check things that are already handled, and point to where it can be changed by hand — this is a very new Figma property (added 2026-08-14), so many users won't know it exists or where to find it — e.g. "Headings got the BALANCE text-wrap style and body copy got PRETTY. Any nodes that were JUSTIFIED in the Japanese source were also switched to LEFT for this non-Japanese duplicate. If you'd like to adjust this yourself, select the text layer and change 'Wrap style' in the Typography section of the right panel." Skip this sentence if neither adjustment actually fired for a given language (e.g. no JUSTIFIED nodes existed).

Always close with both reminders, regardless of whether anything was skipped:

> Please visually check every duplicated frame. Areas without auto-layout in particular may have broken layout depending on how much the translated text's length changed. Numbers, proper nouns, and line breaks were translated mechanically, so please also confirm they came out as intended. The translations themselves are AI-generated as well — a native-speaker review per language is recommended before shipping.

If any nodes were skipped, list them explicitly (as above) rather than only stating a count — the user needs to know exactly where to look.

---

## Error handling

- No selection, multiple nodes selected, or the selection isn't FRAME/SECTION: ask the user to select a single frame or section
- If zero text nodes are found in Step 2: tell the user and stop
- If the user names zero target languages: ask again
- If a script execution fails entirely for one language (not just individual nodes): report that language as failed, note the clone may exist in a partial state, and continue with the remaining languages rather than aborting the whole run
- If a node from Step 2 doesn't resolve to a matching node in the clone at apply time (tree shape changed between scan and apply): skip it and note "structure changed since scan — re-run recommended"
