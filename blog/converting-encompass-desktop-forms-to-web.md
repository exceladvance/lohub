# Converting Encompass Desktop Custom Forms to Encompass Web: What Breaks and How to Fix It

*A hands-on guide for Encompass admins. No programming background required. Read it once, then follow it with your own form beside you.*

---

## Who this is for, and what you will walk away with

You maintain custom input forms in Encompass. They were built for the desktop client (SmartClient), they work, and your users like them. Now you need the same forms in Encompass for Web, and you have noticed that you cannot simply open an old form there and carry on.

By the end of this guide you will be able to:

1. **Explain** why a desktop form cannot be copied to web as-is.
2. **Inventory** a desktop form so you know exactly what must survive the move.
3. **Predict** the nine things that break, and apply the fix for each.
4. **Run the conversion** with a repeatable workflow, including a pass/fail check before anything reaches your users.
5. **Import, test and enable** the web form safely, and fix the usual import errors in minutes.

Where a step is done by a tool or script rather than by you, the guide says so, and tells you what *you* must check. The judgment calls stay with you, because you know your forms and your loan officers.

> **How to use this guide.** Each section ends with a short **Checkpoint**. Do not skip them. They are where the learning happens, and they catch mistakes while those are still cheap.

---

## Part 1: The mental model (10 minutes that save you hours)

### The poster and the bookshelf

Think of a **desktop form as a poster pinned to a wall with thumbtacks**. Every field has an exact spot: "this box sits 412 pixels from the left and 96 pixels from the top." Nothing moves. If the wall is a different size, tough luck.

Think of a **web form as a bookshelf that resizes itself to the room**. You do not say where a field sits in pixels. You say "this field takes 3 of the 12 columns available, and it comes after that other field." The page then flows to fit whatever screen it lands on.

Almost every conversion problem comes from this one difference. Keep the poster and the bookshelf in your head and the rest of this guide will feel obvious.

### Side by side

| Idea | Encompass Desktop | Encompass Web |
|---|---|---|
| Format | An older HTML dialect inside a `.empkg` package | A JSON form schema inside a `.zip` package |
| Position | Absolute pixels | A 12-column liquid grid (width 1 to 12, optional offset) |
| Field binding | A single attribute, `emid="2"` | **Two** settings: a data binding and a field label (more below) |
| Behaviour | Built-in macros that run immediately (VB.NET) | Optional script that runs asynchronously (JavaScript) |
| Calculations | Defined in the field's settings | Still defined in the field's settings (they do not move into the form) |
| Built with | Hand-edited HTML or the old form builder | Input Form Builder (IFB) |

### Three rules to tattoo on your hand

1. **Port the grid, never the pixels.** Do not try to reproduce desktop coordinates on web.
2. **Nothing visible on desktop may silently disappear.** Every field and every button either arrives on web or is written down as intentionally left out.
3. **Never invent an Encompass field ID.** Use IDs that came from the desktop form, from you, or from your instance. If you are unsure, flag it and verify.

> **Checkpoint 1.** In one sentence, why can't a desktop form with a field at "left: 560px" be imported into web? *(Answer: web has no pixel positions. It places controls by column width and order, so position has to be re-expressed as grid widths.)*

---

## Part 2: Prepare your inventory

Before converting anything, collect these six things. Ten minutes here prevents the "why is the right half of the page empty?" surprise later.

| # | You need | Why |
|---|---|---|
| 1 | The desktop package (`.empkg`, or the `.emfrm` / `FORM.htm`) | The source of truth |
| 2 | **A screenshot of the desktop form** | Layout tools guess which label belongs to which field. A screenshot settles it |
| 3 | A list of the custom fields (`CX.*`) the form uses | You will confirm they exist in the target instance |
| 4 | Admin access to Input Form Builder in the target instance | To import and test |
| 5 | A test loan with realistic data | To prove bindings load |
| 6 | A short list of **which buttons actually matter** | Desktop packages do not store macro behaviour, so you tell us what each button did |

### Build the "emid inventory"

On the desktop, every field is tied to a loan field by an `emid`. A tiny illustrative fragment looks like this:

```html
<!-- illustrative desktop fragment -->
<SPAN style="left:40px; top:96px">Loan Amount</SPAN>
<INPUT type="text" emid="2" style="left:160px; top:96px; width:110px">
<INPUT type="text" emid="CX.DD.2" style="left:285px; top:96px; width:110px">
```

Your inventory is simply the list of every `emid` on the form: `2`, `CX.DD.2`, and so on. This list is **the contract**. When you finish, every item on it must be either bound on the web form or explicitly listed as left out, with a reason.

The extraction step builds this list for you automatically. It also records:

- the container nesting (panels, group boxes, category boxes)
- every control type, dropdown options and calendar date fields
- the custom field definitions from the package manifest, including dropdown options and audit fields
- the desktop button handlers (the `<EMEVENTS>` section)

> **Checkpoint 2.** Open your desktop form. Count the visible buttons and write the number down. You will compare it with the web form at the end.

---

## Part 3: What breaks, and how to fix it

Each item follows the same pattern: **what you will see, why it happens, how to fix it.**

### Break 1: The layout collapses or wastes half the page

**You see:** fields stacked in one long column, or a big empty area on the right.

**Why:** on desktop, a two-column form is just fields placed at different pixel positions. Web has no such positions.

**Fix:** translate pixels into the 12-column grid.

- A wide two-column desktop form becomes **one panel** containing group boxes whose widths add up to 12, for example **5 + 7**. Emit them in reading order: left box, its right-hand partner, then the next pair.
- Desktop "label on the left, field to the right" rows become a **standalone Label plus inputs on the same grid row**, with widths summing to 12 or less:

```
Label "Loan Amount"   width 4
TextBox (Current)     width 3
TextBox (UW Decision) width 3
TextBox (Variance)    width 2        4 + 3 + 3 + 2 = 12
```

- Keep column widths **consistent down a box** so it reads like a table.
- A control can never have `width + offset` greater than 12.
- Buttons and fields that sit in a group box title bar belong in that group box's **header**.

> **A tip on very wide forms.** If the desktop canvas is wider than about 1500 pixels, do not try to split it into page columns. Stack the major sections full width instead. Web scrolls vertically happily. A table with more than 12 columns of cells gets split into two stacked, aligned tables with the row label leading both.

### Break 2: Every field shows "(Unassigned)" in Input Form Builder

**You see:** the form imports, but every control says "(Unassigned)".

**Why:** on desktop one attribute did all the work. On web, a binding needs **two** pieces:

1. the data binding (which loads and saves the value), and
2. a control-level `field` label, shaped like `12 (Subject Property City)`, which is what the Field box in IFB reads.

Miss the second one and IFB believes nothing is assigned.

**Fix:** make sure every bound control carries both, and that the ID in the label matches the ID in the binding. Generated packages do this for you. If you ever build one control by hand in IFB, the tool handles it.

### Break 3: Standard fields load blank

**You see:** a field like Loan Amount (`2`) shows nothing on a real loan.

**Why:** web needs the field's loan-model path (for example `loan.baseLoanAmount`). The desktop package never stored that. It only said `emid="2"`.

**Fix, in order of preference:**

1. Use the built-in table of verified paths (it covers a few dozen common fields such as 2, 3, 4, 11 to 15, 353, 976 and the borrower name fields).
2. For anything else, use an **ID-only binding** and keep that control **read-only**. Encompass resolves the details itself, and a failed lookup shows blank instead of writing wrong data.
3. To learn the real path, bind one control by hand in IFB, export the form, and read what IFB wrote. This is always the authoritative answer.

Whatever you use, write every ID-only binding in your delivery notes.

### Break 4: Buttons do nothing

**You see:** buttons are on the form but clicking them has no effect, or they vanished.

**Why:** desktop buttons ran VB.NET macros, and the `.empkg` does not contain macro behaviour you can simply carry over. Also, a button's `emid` such as `voe` or `uwvalidate` is a **trigger name, not a loan field**. Never bind it to a field.

**Fix:**

1. Keep every button, so nothing visible disappears.
2. Ask the business which buttons matter.
3. For the ones that matter, port the logic to a script. Desktop events that were written in the form's `<EMEVENTS>` section translate roughly like this:

| Desktop | Web script |
|---|---|
| `Macro.GetField("X")` | `await loan.getField("X")` |
| `Macro.SetField("X", v)` | `await loan.setFields({ X: v })` |
| `Now.ToShortDateString` | a date string built from `new Date()` |
| `Macro.Popup(...)` | does not port. Often already commented out |
| Form `Load` event | `FormLoad()` |

4. Buttons you decide not to port stay as **dead buttons**, listed in your notes with their captions.

Two lessons learned the hard way. First, **control names lie**: a control called `...commentlog` may be bound to a completely different field, so always look up each control's real field before porting. Second, a leading apostrophe in VB (`'`) means the line was a comment and never ran, so do not port it.

### Break 5: A dropdown quietly disappears

**You see:** a dropdown that existed on desktop is missing on web.

**Why:** some desktop combo boxes are not normal dropdowns. They are embedded objects (`DropdownEditBox`) that hide both the field ID and the option list inside parameter tags. Naive tools skip them.

**Fix:** port each one as a web **Dropdown with its options entered inline**. Because desktop could write to the field through that combo, these may need to stay editable, which has to be acknowledged explicitly in the checks (see Part 5).

### Break 6: Controls change type on the way over

| Desktop | Web |
|---|---|
| Text box for a date, plus a calendar button | One **Calendar** control |
| Dropdown (SELECT) | **Dropdown** (options come from the field definition) |
| Multi-line text area | **MultilineTextBox**, full width (12) |
| Checkbox | **Checkbox**, with its caption stored in the first option's text |
| Group of radio-style checkboxes on one field | Separate Checkboxes bound to the same field, or one Dropdown (cleaner, but agree it with your users first) |
| Zip lookup icons, divider lines, decorative images | Dropped |

### Break 7: Calculated fields show blank

**You see:** a calculated column like "Variance" or "Income Expires" is empty on a real loan.

**Why:** calculations such as `IIF([CX.LD.2] <> [2], "Y", "N")` do not live in the form. They live in the **field definition** in your admin settings. The conversion carries the definitions descriptively, but it cannot create fields in your instance.

**Fix:** this one is yours, as the admin.

1. Open the Custom Field settings in the target instance.
2. Confirm each `CX.*` field exists, with the right format and calculation.
3. Create or correct any that are missing.
4. Only if that is not possible, use a script as a fallback to compute values.

### Break 8: Values that another process used to fill in

**You see:** a "snapshot" section that is permanently empty.

**Why:** desktop forms often *displayed* data that a separate macro or automation wrote. Move the form and that other process may not come with it.

**Fix:** ask who or what populated each such field. The web form may need a button or an automation to take over that job. Flag it.

### Break 9: Surprises in the source form

Three things that are easy to miss:

- **Hidden containers.** Containers marked as not visible on desktop (`emvisible="0"`) were script-driven. Skip them or port them hidden, and say which you did.
- **Duplicate IDs.** Desktop forms often repeat a field. Decide whether that was intentional or a copy-paste slip, and note it.
- **Off-canvas controls.** Some fields sat beyond the declared width and were never seen. Place them deliberately.

> **Checkpoint 3.** Match each symptom to its break number: (a) "Everything says (Unassigned)", (b) "My calculated column is blank", (c) "A dropdown is gone", (d) "The right half of the page is empty". *(Answers: a = 2, b = 7, c = 5, d = 1.)*

---

## Part 4: The conversion, step by step

This is the workflow. The extraction, layout draft, package build and validation steps are run by the conversion tooling (the Claude skill for this job runs them for you). Your own work is marked **You**.

### Step 1: Extract and inventory

The tool reads the desktop package and produces an inventory of containers, controls, emids, custom field definitions and button handlers.

**You:** compare the control count and button count with what you wrote down in Checkpoint 2.

### Step 2: Draft the layout

The tool proposes a grid plan: page bands, group box widths in reading order, and widths and offsets for each row.

**You:** hold the draft next to your desktop screenshot.

- Are labels paired with the right fields?
- Do rows read the way they did on desktop?
- Do widths add up sensibly?

Treat the draft as a first cut. The grid is design judgment, and you are the judge. Once you are happy, the reviewed widths are fixed into the build script for that form.

### Step 3: Build the package

The tool turns your reviewed model into the web package. A few built-in safeguards are worth knowing because they explain why generated packages import cleanly:

- the form name comes from the **source form**, never from a copied script (a script copied from another project once imported a form with the wrong name)
- every desktop emid is bound exactly once, or documented
- a **catch-all "Additional Fields" box** collects anything the layout did not place, so nothing is silently lost
- the build **aborts** on known import-killers instead of shipping a broken package

By default the package is **scriptless**. Bindings load and save by themselves, and calculations run from the field definitions. A script is added only when the desktop form had real button or load behaviour.

### Step 4: Validate (the mandatory gate)

The validator checks hierarchy, grid geometry, checkbox captions, field definitions, import rules, field coverage, button-caption parity and button-handler parity.

The rule is binary: **keep re-running until it reports ALL PASS.**

If a failure is genuinely intentional, you may acknowledge it with a flag. Each flag downgrades the failure to a warning that **must** appear in your delivery notes:

| Flag | You are saying |
|---|---|
| `--allow-unbound` | "Some emids are deliberately not bound" |
| `--allow-missing-buttons` | "Some desktop buttons are deliberately not on web" |
| `--allow-unported-events` | "Some button handlers are deliberately not ported" |
| `--allow-editable-idonly` | "These ID-only bindings stay editable on purpose" |

Renaming a button caption counts as dropping the button until you acknowledge it.

### Step 5: Import and test

**You:**

1. In Input Form Builder, choose **Import** and select the zip.
2. Open the form. It arrives **disabled** on purpose.
3. Open a **test loan** and confirm that values load.
4. Compare it to the desktop form beside it, field by field, using your emid inventory.
5. Test every button you decided to port.
6. Only then enable the form.

### Step 6: Round-trip and iterate

Right after the first successful import, **export the form back out of IFB**. The server assigns its own identifiers (including the form's masked ID). Adopt them in your build so future imports **update** the form instead of creating duplicates.

If something looks off, change widths or offsets in the **build** and re-import. Do not hand-edit the exported JSON. And when you meet an unfamiliar control or binding, build it by hand in IFB once, export, and read the result. That is the most reliable teacher there is.

> **Checkpoint 4.** Why do we export a form right after importing it? *(Answer: to pick up the server-assigned IDs, so later imports update the form in place rather than duplicating it, and to learn any contract shape we were unsure about.)*

---

## Part 5: When the import fails

Every row below happened on a real instance.

| Message or symptom | Real cause | Fix |
|---|---|---|
| "There was an error processing `<name>_default.js` status update" | A script is listed in the manifest but missing from the schema's **Assets** list (or the IDs differ). The script's content is *not* the problem | Add the matching asset entry and make the IDs equal |
| "Invalid data for parameter(s) description : size must be between 0 and 100" | A description is longer than 100 characters (in the manifest or in the schema metadata) | Shorten to 100 or less. Long explanations belong in the script's header comment |
| Fields show "(Unassigned)" | The control-level `field` label is missing | Add `"<fieldId> (<description>)"` to each bound control |
| Fields blank on a real loan | The field is genuinely empty, an ID-only binding did not resolve, or a calculation is missing in the instance | Check the loan data, then the real field path, then the admin field definition, in that order |

**If you meet a brand-new error,** read the message literally. It is precise. Then isolate the cause by importing small variants: a layout-only version with no script, then a version with a trivial script. Whichever one fails tells you where the problem lives.

---

## Part 6: Delivery notes template

When you hand the form over, include this. Anyone should be able to see what was decided and why.

```
FORM: <name>              SOURCE: <desktop form name / package>

Bindings
  Verified standard paths: <list>
  ID-only (read-only) bindings: <list>
  Duplicate bindings kept: <list and reason>

Buttons
  Ported and wired: <caption -> handler>
  Dead buttons (not ported): <captions>

Left out on purpose
  Hidden containers skipped: <list>
  Decorative controls dropped: <list>
  Fields swept into "Additional Fields": <list>

Instance checks (admin)
  CX fields confirmed present: <list>
  CX fields missing or needing calculations: <list>
  Snapshot-style fields and who fills them: <list>

Validator flags used: <flags, if any, with reason>
Status: imported disabled. Tested on loan <number> on <date>. Ready to enable: yes / no
```

---

## Part 7: Quick reference

### The 10-line cheat sheet

1. Port the grid, never the pixels.
2. Wide forms: one panel, group boxes that add up to 12 (5 + 7, or 6 + 6).
3. Label-left rows: standalone Label + inputs, widths sum to 12 or less.
4. A binding is two settings. Miss one and you get "(Unassigned)".
5. Unknown standard field: ID-only binding, read-only, and list it.
6. Button `emid`s are triggers, not fields. Keep every button.
7. Watch for hidden dropdowns (`DropdownEditBox`).
8. Calculations live in the field definitions. Verify them in the instance.
9. The form must pass validation. Acknowledge exceptions in writing.
10. Import disabled, test on a real loan, export back, then enable.

### Glossary

- **emid**: the desktop attribute that ties a control to a loan field.
- **CX field**: a custom field you define in your instance.
- **IFB**: Input Form Builder, where web forms are built and imported.
- **Id-only binding**: a binding that gives only the field ID and lets Encompass work out the rest.
- **Round-trip**: import, then export the form to learn what the server actually stored.
- **Offset**: blank columns placed before a control on the grid.

---

## Final exam (no peeking)

1. A control has width 8 and offset 5. Will it be accepted? Why or why not?
2. Your form imports, but every field says "(Unassigned)". Which key is missing?
3. A button is called "VOE" and its emid is `voe`. Should you bind `voe` to a loan field?
4. A calculated field is blank on web but fine on desktop. Who fixes it, and where?
5. You renamed "Validate" to "Run Validation" on the web form. What happens at validation, and what do you do?

**Answers.**
1. No. 8 + 5 = 13, and width plus offset cannot exceed 12.
2. The control-level `field` label (`"<id> (<description>)"`).
3. No. Button emids are macro triggers, not loan fields.
4. The admin, in the Custom Field settings of the instance, since calculations live in field definitions.
5. The validator treats the rename as a dropped button and fails. Restore the caption, or acknowledge the change with the matching flag and list it in the delivery notes.

---

## One last thought

A conversion is not a copy job. It is a translation, and good translations preserve meaning, not word order. Your loan officers do not care that the Loan Amount box sat at pixel 160. They care that it is there, that it loads, and that the Variance column still tells them when something changed. Keep the inventory honest, let the grid do the layout, test on a real loan, and you will ship forms people trust.
