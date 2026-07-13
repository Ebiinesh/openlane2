# Opening Your Chip Layout in Magic — Troubleshooting Log

This is the story of trying to open a finished chip layout (a `.gds` file) inside a tool called **Magic**, and everything that went wrong before it finally worked.

If you've just made your first chip layout and Magic won't open it properly, this guide is for you. Every step is explained simply, like you've never used Magic before.

---

## The Big Picture (Read This First)

Think of your `.gds` file like a **drawing of your chip**.

To look at that drawing correctly, Magic needs two things:

1. **The drawing itself** — your `half_adder.gds` file
2. **A rulebook** — called the "technology file" — that tells Magic:
   - what colors mean what
   - what the layers are (metal, silicon, etc.)
   - how big things really are

If Magic opens the drawing **without** the rulebook, it's like looking at a map with no legend — you'll just see a blank or broken mess. That's exactly what happened here.

```
.gds file  (the drawing)
     +
Technology file  (the rulebook: "sky130A")
     =
A correct, readable chip layout in Magic
```

Everything below is the journey of getting both pieces to load together correctly.

---

## Problem 1: "How do I even open this file in Magic?"

### The starting command

```bash
magic -rcfile ~/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130A/libs.tech/magic/sky130A.magicrc ~/vlsi/projects/half_adder/runs/RUN_2026-07-09_20-55-31/final/gds/top.gds
```

### What this command means, piece by piece

| Piece | What it means (in plain English) |
|---|---|
| `magic` | "Open the Magic program" |
| `-rcfile ...sky130A.magicrc` | "Use this settings file so you know it's a Sky130 chip" |
| `...final/gds/top.gds` | "Here's the drawing file to open" |

### Two simple ways to open Magic with the rulebook

**Option A — the recommended way, using `-rcfile`:**

```bash
magic -rcfile ~/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130A/libs.tech/magic/sky130A.magicrc ~/vlsi/projects/half_adder/runs/RUN_2026-07-09_20-55-31/final/gds/top.gds
```

**Option B — pointing directly at the technology file with `-T`:**

```bash
magic -T ~/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130A/libs.tech/magic/sky130A.tech ~/vlsi/projects/half_adder/runs/RUN_2026-07-09_20-55-31/final/gds/top.gds
```

**Why `-rcfile` is usually better:** the `.magicrc` file is like a "starter pack" — it automatically loads the correct technology *and* colors, display style, and other Sky130 settings all at once. `-T` only loads the technology rulebook by itself.

⚠️ **First small mistake to avoid:** don't literally type `/path/to/sky130A.tech`. That's just a placeholder example. You must use your *actual* file path, the long one starting with `~/.volare/...`.

---

## Problem 2: The file wasn't opening — but was it even the right filename?

The command above used `top.gds`, but the actual generated file (confirmed by checking the output folder) was named `half_adder.gds`.

### How to check the real filename

```bash
ls -lh ~/vlsi/projects/half_adder/runs/RUN_2026-07-09_20-55-31/final/gds/
```

This just means: **"show me the files in this folder, with their sizes."**

**Lesson:** always double-check the exact filename before using it in a command. A single wrong letter (`top` vs `half_adder`) means the file "doesn't exist" even though a similar one does.

---

## Problem 3: Magic opens... but shows the wrong "rulebook"

When Magic was opened plainly, like this:

```bash
magic
```

it opened fine — no crash — but the title bar said:

```
Technology: minimum
Using technology "minimum", version 0.0
```

### Why this is a problem

`"minimum"` is Magic's **built-in placeholder rulebook** — it's not Sky130 at all. It's like opening a book written in a language Magic doesn't actually understand for your chip. The layout will look empty, broken, or meaningless because Magic has no idea what your chip's colors and layers mean.

**What we needed to see instead:**

```
Technology: sky130A
Using technology "sky130A", version 1.0.493
```

### The fix — always tell Magic which rulebook to use

Never just type `magic` by itself for a Sky130 design. Always include the rulebook:

```bash
magic -rcfile ~/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130A/libs.tech/magic/sky130A.magicrc
```

---

## Problem 4: A small but easy-to-make mixup — two different "typing places"

At one point, this was typed:

```
echo $DISPLAY
```

**...but it was typed *inside* Magic's own command box (called `tkcon`)**, not in the normal terminal.

### Why this caused an error

```
can't read "DISPLAY": no such variable
```

### The simple explanation

Think of it like having **two different notebooks**:

1. 📓 **Your Linux terminal** — where you type things like `ls`, `cd`, `magic`
2. 📕 **Magic's tkcon window** — a *different* notebook, with its own language (called Tcl), for typing commands *inside* Magic like `load`, `gds read`, `view`

```
Linux terminal (bash)
     │
     │  type:  magic -rcfile ...
     ▼
Magic opens, and a tkcon window appears
     │
     │  now type Magic-only commands here, like:
     │  load half_adder
     │  gds read ...
     │  view
```

**Rule of thumb:** if a command is about your computer in general (`echo`, `ls`, `cd`, `export`) → type it in the **terminal**.
If a command is about the chip layout itself (`load`, `gds read`, `view`, `cellname list`) → type it in **tkcon**.

---

## Problem 5: The scariest one — Magic opens for a split second, then disappears

Running this:

```bash
magic -rcfile ~/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130A/libs.tech/magic/sky130A.magicrc
```

...opened a window for about one second, and then it just **closed itself**, like a crash.

### How we figured out what was happening

**Step A — Catch the error message before it disappears**

Normally, if a program crashes, the error flies by too fast to read. The trick is to **save it to a file** instead of just watching the screen:

```bash
magic -rcfile ~/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130A/libs.tech/magic/sky130A.magicrc 2>&1 | tee magic.log
```

**In plain English:** "Run Magic, and whatever it prints (even errors), save a copy into a file called `magic.log` so I can read it afterward — even though the window disappeared."

Then read it back:

```bash
cat magic.log
```

**Step B — Test the technology file on its own, without the rc file**

```bash
magic -T ~/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130A/libs.tech/magic/sky130A.tech
```

### What this test told us

| Command used | What happened | What it tells us |
|---|---|---|
| `magic` (nothing extra) | Opens fine, but wrong rulebook (`minimum`) | Magic itself works |
| `magic -rcfile ...sky130A.magicrc` | Opens, then instantly crashes | Something inside the **rc file** (the "starter pack") is causing a problem |
| `magic -T ...sky130A.tech` | **Opens correctly, with `Technology: sky130A`** ✅ | The actual rulebook file is totally fine |

**The conclusion:** the core files (Magic itself, and the Sky130 technology file) are both perfectly healthy. The issue was narrowed down specifically to *how the `.magicrc` "starter pack" file behaves* when used with `-rcfile` — possibly expecting something extra from the environment that wasn't set up (to be investigated further, e.g. environment variables like `PDK_ROOT`).

---

## ✅ The Working Solution

Skip the `-rcfile` starter pack for now, and load the technology file directly with `-T`:

```bash
magic -T ~/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130A/libs.tech/magic/sky130A.tech
```

You should see in the title bar or terminal output:

```
Using technology "sky130A", version 1.0.493...
```

✅ That means it worked — Magic is using the correct Sky130 rulebook.

### Now load your actual chip drawing

Inside the **tkcon** window (not the terminal!), type:

```
gds read ~/vlsi/projects/half_adder/runs/RUN_2026-07-09_20-55-31/final/gds/half_adder.gds
```

Then check what shapes/cells were loaded:

```
cellname list
```

You should see something like:

```
half_adder
```

Then actually load and display it:

```
load half_adder
```

Finally, zoom to fit the whole layout on screen:

```
view
```

(or from the menu bar: **View → Full**)

---

## Quick Summary — What Actually Fixed It

```
❌ magic
   → opens, but uses fake "minimum" rulebook (wrong)

❌ magic -rcfile .../sky130A.magicrc
   → opens for a second, then crashes (starter pack has an issue)

✅ magic -T .../sky130A.tech
   → opens correctly with the real "sky130A" rulebook

then inside tkcon:
✅ gds read .../half_adder.gds
✅ cellname list
✅ load half_adder
✅ view
```

---

## Common Errors — Quick Reference

### Title bar says `Technology: minimum`
**Cause:** Magic was started with no technology file at all (just `magic` by itself).
**Fix:** always launch with `-T <path to .tech>` or `-rcfile <path to .magicrc>`.

### `can't read "DISPLAY": no such variable`
**Cause:** typed a normal Linux command (`echo $DISPLAY`) inside Magic's `tkcon` window instead of the terminal.
**Fix:** close nothing — just switch to your actual terminal window to run Linux commands. Use tkcon only for Magic commands like `load`, `gds read`, `view`.

### Magic window opens and disappears almost instantly (looks like a crash)
**Cause (in this case):** the `-rcfile` "starter pack" file was causing an early exit — the core Magic program and the technology file were both fine on their own.
**Fix:** use `-T <path to .tech>` directly instead of `-rcfile` until the rc file issue is investigated further.
**How to catch the real error next time:** redirect the output to a log file so it doesn't disappear with the window:
```bash
magic -rcfile <path> 2>&1 | tee magic.log
cat magic.log
```

### "File not found" when loading the GDS
**Cause:** using the wrong filename (e.g. `top.gds` when the real file is `half_adder.gds`).
**Fix:** always check first with:
```bash
ls -lh <path to final/gds folder>
```

---

## Things Still To Investigate Later

- *Why* exactly does `-rcfile sky130A.magicrc` crash while `-T sky130A.tech` works? The rc file likely expects an environment variable (such as `PDK_ROOT`) to already be set, which may not be happening outside of OpenLane's own automated run.
- Reading the first ~30 lines of `sky130A.magicrc` to see what extra setup steps it performs beyond just loading the technology file:
  ```bash
  head -30 ~/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130A/libs.tech/magic/sky130A.magicrc
  ```
- Comparing this against the **exact Magic command OpenLane itself used** during the successful automated run (found by searching the run's logs):
  ```bash
  grep -R "magic" ~/vlsi/projects/half_adder/runs/RUN_2026-07-09_20-55-31/logs -n
  ```

---

## Key Lesson From This Whole Debugging Session

When something "doesn't work" in a tool like Magic, don't just try random things — **change one thing at a time** and observe:

```
Does the program even open?          → tests the program itself
Does it show the correct rulebook?    → tests the technology/config file
Does it load the actual drawing?      → tests the GDS file / filename
Which "notebook" am I typing in?      → terminal vs tkcon
```

By testing each piece separately (`magic` alone → `magic -T ...` → `magic -rcfile ...`), it became possible to pinpoint *exactly* which piece was broken, instead of guessing.

---

