---
name: arabic-pptx-format
description: تنسيق عروض PowerPoint العربية بالخطوط المعتادة (MasmakBHD للعناوين، Watad لعناوين البطاقات، Alnaseeb للمتن)، وضبط اتجاه الفقرات يمين→يسار، وتحويل العرض عند الطلب إلى مقاس طولي 9:16 مناسب للجوال مع إعادة توزيع الشرائح رأسيًا.
---

## When to use
When the user, inside a PowerPoint deck in Arabic, asks to «نسّق بمهارتنا المعتادة»، «طبّق الخطوط المعتادة»، «نسّق الباوربوينت»، «اضبط اتجاه النص»، «خلّ الشرائح طولية / مناسبة للجوال»، or invokes /arabic-pptx-format. Companion of the Word skill `arabic-doc-format-v6`; this one carries only what applies to slides (fonts, RTL, portrait layout). Page margins, ornaments, TOC and indents belong to Word and are NOT applied here — say so in the report.

Messages may arrive typed on an English keyboard layout (e.g. «KSR FLIHVJKH HGLUJH]M» = «نسق بمهارتنا المعتادة»). Decode with the standard Arabic keyboard map before asking for clarification.

## Font scheme (keep the deck's own colors and sizes unless stated)
| Element | Font | Notes |
|---|---|---|
| Slide titles (placeholders of ph type `title` / `ctrTitle`) | MasmakBHD | Arabic only. MasmakBHD has no Latin glyphs → set the `<a:latin>` typeface to **Arial** and `<a:cs>`/`<a:ea>` to MasmakBHD, otherwise Latin words fall back to a serif. |
| Card / section headings (bold TextBox runs, sz 1800, text longer than 2 chars) | Watad | |
| Everything else (body, descriptions, subtitles, numbers inside circles) | Alnaseeb | Raise description size 13 → 14 pt (the floor for body text). |
| Theme (`ppt/theme/theme1.xml`) | majorFont latin=Arial, cs/ea=MasmakBHD; minorFont = Alnaseeb (latin, cs, ea, and `<a:font script="Arab">`) | So new slides inherit. Also sweep hardcoded `Arial` in slideMaster*/slideLayout*.xml. |

## Steps
1. **Inspect** (one `execute_office_js`): per slide load shapes `id,name,type,left,top,width,height` and each text frame's `font.name/size/bold` and first 80 chars. Done when: you know which shapes are title placeholders, bold 18 pt headings, and body, and whether fonts are hardcoded (`<a:latin typeface="Arial">`) or inherited.
2. **Theme & master** (one `edit_slide_master`): set majorFont/minorFont as in the table; replace `typeface="Arial"` in master and layouts with Alnaseeb (titles → MasmakBHD). Done when: theme XML reads back with the new faces.
3. **Fonts per slide** (one `edit_slide_xml` per slide, same code each time): inside each `<p:sp>`, decide face by rule (title placeholder → MasmakBHD; `b="1"` + `sz="1800"` + text > 2 chars → Watad; else Alnaseeb) and rewrite `typeface="Arial"` on `<a:latin|ea|cs>`; then `sz="1300"` → `sz="1400"`. On title shapes set `<a:latin>` back to Arial. Slide ids change after every edit — re-read `slidesMetadata` from the tool results rather than reusing old ids. Done when: every slide processed and a `verify_slide_visual` on a dense slide confirms Arabic in the display fonts and Latin in sans.
4. **RTL paragraphs** (one `edit_slide_xml` per slide): add `rtl="1"` to every `<a:pPr>` lacking it, and insert `<a:pPr rtl="1"/>` into `<a:p>` elements that have none. Use `/<a:pPr(\s[^>\/]*)?(\/?)>/` so self-closing tags are handled. Done when: bullets render on the right of the text and parentheses around Latin words are mirrored correctly.
5. **Portrait / phone size (only if asked)**:
   - `ppt/presentation.xml` `<p:sldSz>` cannot be changed from the add-in (the write does not persist). Ask the user to do it manually: تصميم ← حجم الشريحة ← حجم شريحة مخصص ← عمودي، عرض 14.29 سم × ارتفاع 25.4 سم (9:16 → 405 × 720 pt). Wait for «تفضل».
   - Then relay every slide in one `execute_office_js`: X=36, content width 333, bottom limit 680. Content slides: title at y=36 h=80 sz 28; subtitle TextBox at y=122 h=44 sz 14; cards stacked with 14 pt gaps, height = (680 − startY − gaps)/n; inside each card the number circle 40×40 at the right edge (inset 14), heading TextBox left of it sz 18, description under at y+64 sz 14, insets 14. Card shape order in the tree is card → number circle → number text → heading → description (yes/no cards: card → heading → body). Title/divider slides: dot 60×60 at y=250, title h=150 sz 36, subtitle sz 18, all centered vertically. Slide number at (36, 688).
   - Also move master/layout title and slide-number placeholders to the same positions so new slides inherit.
   Done when: `verify_slides` shows no out-of-bounds/overlaps and screenshots of slides 1, 2, a 3-card and a 4-card slide look right.
6. **Verify & report in Arabic**: fonts per level, RTL applied, size changed (or not), and the Word-only items that were intentionally skipped.

## Notes
- Edit text only through OOXML (`edit_slide_xml`); Office.js is for geometry and font size.
- `getImageAsBase64` + `attachImage` is enough for quick checks; use `verify_slide_visual` on one dense slide at the end.
- A card heading that wraps because of a long bilingual label («Scheduled — المهام المجدولة») is pre-existing; mention it, do not shorten the user's text.
