---
title: Template editor upgrades (richer formatting, merge-field chips, preview and test send)
status: approved        # proposed | approved | in-progress | built | dropped
owner: Qamar Ul Islam
area: docs/areas/inbox.md; changes proposals/2026-10-message-templates.md (template editor) and builds on proposals/2026-10-live-merge-fields.md
screens: [engage (Templates › email template editor, SMS template editor), inbox (New email, reply and forward)]
tickets: []             # filled in after approval
created: 2026-10-20
---

# Template editor upgrades

## Problem
Found in the demo (2026-10-20). The email editor is basic: bold, italic, a bullet list and a link that opens a browser prompt. That's not enough for real sales emails, which need headings, a call-to-action button, an image or a highlighted line. The app already uses TipTap, which has all of this; we only switched on four features. Merge fields show as raw `{{first_name}}` text that's easy to break with a typo. Reps also can't see how an email looks on a phone, or send one to themselves before using it.

## Proposal
One upgraded editor for every place a rep writes a formatted email. **Screens** (design.nrtur.io):

- `engage` › **Templates** › email template editor (the dialog from the templates proposal):
  - the upgraded editor;
  - the preview gets a **Desktop / Mobile** toggle and a **Plain text** tab;
  - a **Send test email to me** button;
  - a size warning.
- `engage` › **Templates** › SMS template editor: an **Emoji** picker.
- `inbox`: **New email** (Compose) and **reply / forward** use the same upgraded editor. Live merge-field filling (proposals/2026-10-live-merge-fields.md) works unchanged, and the chips show the filled values.

Other rich-text fields (notes, record descriptions and so on) keep today's basic editor. In the prototype the upgrade is the opt-in `full` mode of `RichTextField`.

## Rules

**Toolbar.** Grouped with dividers, and it wraps on narrow widths. Every button has a tooltip and an accessible name.
1. **Text:**
   - a block-type select: Normal, Heading 1, Heading 2, Heading 3;
   - bold, italic, underline, strikethrough;
   - **text colour** and **highlight**, each a popover with a fixed palette of 8 colours (highlight also has "No highlight");
   - align left, centre and right;
   - clear formatting.
2. **Blocks:** bullet list, numbered list, quote, divider (horizontal line).
3. **Links and buttons:**
   - **Link** opens a popover: URL, "Open in new tab" (on by default), Apply, and Remove when the cursor is in a link. It replaces the browser prompt.
   - A URL without a scheme gets `https://`. Anything else that isn't `http(s)://` or `mailto:` is refused with a message.
   - **Button** opens a popover with the button text (required, up to 40 characters) and a link (required). It inserts a call-to-action link styled as a rounded button in the brand colour with a white label, and shows a preview.
4. **Images:**
   - **Image** opens a popover with **Upload** or **Paste a link** (https only), a width (Small 160 px / Medium 320 px / Full width) and alt text.
   - Uploads are JPG, PNG or GIF, up to **5 MB**.
5. **Merge fields as chips:**
   - In the editor, merge fields show as small non-editable chips labelled with the field name (e.g. "First name", or "First name · there" with a fallback).
   - Insert one from the toolbar **Merge field** menu, which lists the 11 merge fields with descriptions, or by typing `{{`, which opens a suggestion list (filters as you type; arrow keys and Enter pick). Typing or pasting a complete known field, like `{{company_name}}`, also turns it into a chip.
   - **Clicking a chip** opens a popover: set or clear a **fallback** (stored as `{{first_name|there}}`), or **Remove field**.
   - **Storage:** a chip is stored with the literal token inside it: `<span data-merge="first_name">{{first_name}}</span>`. Filling, leftover detection and the 422 for unknown fields all work on the token, unchanged.
   - Once a composer fills a field, the chip shows the value (e.g. "Sarah"), still styled as a chip. Its popover offers "Show the field again" or Remove.
   - **On send** a chip is plain text: the value, with no markup.
6. **Speed:**
   - **Undo** and **Redo** buttons.
   - Shortcuts: ⌘/Ctrl+B, I and U; ⌘/Ctrl+K for a link; ⌘/Ctrl+Z and ⇧⌘/Ctrl+Shift+Z.
   - A **"/" command menu**: typing `/` at the start of a line opens Heading 1–3, Bullet list, Numbered list, Quote, Divider, Button, Image and Merge field. It filters as you type; arrow keys and Enter pick; Escape closes it.
   - Escape in a popover or menu closes only that popover or menu, never the dialog underneath.

**Email safety**
- Only the formatting listed above is allowed. Anything else (pasted fonts, sizes, tables, scripts, other colours) is removed when the message is saved.
- Colours come only from the fixed palettes:
  - text: grey, red, orange, amber, green, teal, blue, purple;
  - highlight: yellow, green, blue, pink, orange, purple, grey, red.
- **Images are hosted by nrtur** and referenced by URL. Images are never embedded in the message.
- **Formatting is inlined at send**, as inline styles, so Gmail, Outlook and Apple Mail show it the same.
- Every formatted email is sent with an **auto-generated plain-text part**:
  - headings and paragraphs on their own lines;
  - bullets as "•" and numbered items as "1.";
  - links as "text (url)" and buttons as "Label: url";
  - quotes prefixed with "> ";
  - images as their alt text.
- Merge-field chips are stored as `{{tokens}}` (see 5).

**Template editor dialog (email)**
7. **Preview:**
   - The live preview stays, filled with the sample contact (Sarah Chen, Meridian Agency).
   - **Email / Plain text** tabs. Plain text shows the plain-text part generated from the current email.
   - **Desktop / Mobile** toggle. Mobile shows the email in a 375 px wide phone frame.
8. **Send test email to me** (left side of the footer):
   - Sends the current version, saved or not, to the signed-in user's own address. It can't be sent to anyone else.
   - Merge fields are filled with the sample contact.
   - A toast confirms: "Test email sent to <address>".
   - Disabled while the template is invalid: missing name, subject or body, a duplicate name, or an unknown merge field.
9. **Size warning:** when the body HTML is over **100 KB**, an amber note shows: "This email is large — Gmail may cut it off. Remove images or text." Gmail clips messages over about 102 KB. Saving is still allowed.

**SMS template editor**
10. **Emoji:**
    - An **Emoji** button next to the message box opens a grid of 24 common emoji. Picking one inserts it at the cursor.
    - **Any emoji switches the whole message to Unicode:** 70 characters per SMS, or 67 per part when it's split.
    - The counter adds "Emoji use Unicode (70 characters per SMS)" and counts SMS on that basis.
    - Without emoji, counting is unchanged, and the 1,600-character limit still applies.
11. **Merge fields in SMS** stay as text tokens inserted by the existing chip buttons. An SMS is plain text, and a plain text box can't hold chips.

**Composers** (inbox New email, reply and forward)
- They use the same editor and toolbar.
- Inserting a template wraps its merge fields as chips. Live filling (proposals/2026-10-live-merge-fields.md) then fills and re-fills them as To changes, and the filled chips show the values.
- The fill panel, the "filled for" line and the Send block for leftover fields are unchanged.
- What's sent is the text on screen, with chips as plain text.

## Permissions
No change. Anyone who can edit a template or write an email gets the full editor. Test emails only ever go to the person sending them.

## Plan and pricing impact
None. All plans get the full editor. Nothing in `pricing/` changes.

## Out of scope
- Tables, custom fonts and font sizes.
- A drag-and-drop email builder.
- HTML import or a code view.
- Attachments in templates.
- AI writing.
- Sequence email steps stay plain text (templates proposal, decision 4).

## Decisions
1. **Owner, 2026-10-20:** images are included in v1.
2. **Owner, 2026-10-20:** the inbox composers (New email, reply and forward) use the same editor as the template editor.

## Decisions (owner, 2026-10-22)
1. **Any non-GSM character switches the SMS counter to Unicode,** not only emoji: curly quotes (’ “ ”), many accented letters, and so on. This matches what the SMS provider bills: 70 characters per SMS, 67 per part when split. The hint says "Emoji or special characters (like ’ or “) use Unicode: 70 characters per SMS".
2. **"Send test email to me" goes to the user's own login email,** whichever mailbox is selected as From.

## Tickets (to create after approval)
- **FE:** template editor upgrades.
  - TipTap extensions:
    - Heading (1–3), Underline, Strike, TextStyle + Color, Highlight (fixed palettes), TextAlign, OrderedList, Blockquote, HorizontalRule;
    - Link with a popover (no `window.prompt`), Image (upload and link, width, alt);
    - a custom **Button** node, a custom **MergeField** node (stored as `{{key}}` / `{{key|fallback}}`, with a fallback popover) and a `{{` suggestion;
    - a "/" slash-command menu, undo/redo and shortcuts.
  - Use the full editor in the email template editor and the inbox composers only.
  - Template dialog: Desktop / Mobile / Plain text preview, the size warning, and the Send test email button.
  - SMS template editor: the emoji picker and Unicode counting.
  - Follows SCRUM-1282 and SCRUM-1283; works with the live merge fields ticket.
- **BE:** email formatting, images and test send.
  - Update the HTML sanitizer allowlist with the new tags (h1–h3, u, s, blockquote, hr, img), attributes (`a[target=_blank]`, `a[data-button]`, `img[src|alt|width]`, `span[data-merge]`) and styles (palette colours, text-align, the button and image styles). Everything else is stripped.
  - Image upload and hosting through the files module: JPG, PNG or GIF, up to 5 MB. Returns a public https URL.
  - Inline CSS at send for email-client compatibility.
  - Add the plain-text alternative part to every formatted email.
  - `POST /templates/{id}/test-send` (or a generic test-send that takes the unsaved subject and body) to the current user's own address, with sample merge values. Rate-limit it.
  - OpenAPI.
