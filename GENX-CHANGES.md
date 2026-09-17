# GenX-DOS changes to Thomas Skibo's PET 2001 emulator

This is a fork of Thomas Skibo's [skibo.github.io](https://github.com/skibo/skibo.github.io),
the repository behind his website, which holds his JavaScript Commodore PET 2001
emulator under `6502/pet2001/` (BSD-2-Clause). It is maintained for the
**GenX-DOS** project (<https://github.com/Retro-Jack/GenX-DOS>), whose Commodore PET
runs this emulator, so that every change we make to it lives here as readable
source with a reason attached, rather than as an edit to a copied file.

Only the PET emulator is of interest here. The rest of the repository — the
Apple II, the Sudoku solver, the site's pages — is Thomas's and is untouched.

## Branches

- **`master`** — Thomas's repository as forked, unchanged.
- **`genx`** — what GenX-DOS ships. It starts from upstream commit `55d18e4`
  (17 March 2021), because that is the version GenX-DOS copied in June 2026: six
  of its nine files are byte-identical to that commit. Thomas has improved the
  emulator a good deal since (a more cycle-accurate video model, typed arrays);
  moving GenX-DOS onto that is a separate job, not something this branch does.

The GenX-DOS bundle takes these files from `genx`:

| This fork (`6502/…`) | GenX-DOS (`systems/pet/pet2001/`) |
| --- | --- |
| `js/cpu6502.js` | `cpu6502.js` |
| `pet2001/js/pet2001.js`, `pet2001hw.js`, `pet2001ieee.js`, `pet2001io.js`, `pet2001roms.js`, `pet2001main.js` | same names |
| `pet2001/js/petkeys.js` | `petkeys.js` |
| `pet2001/js/pet2001video.js` | `pet2001video.js` |

---

## Change 1 — declare `via_t2ll`

**File:** `6502/pet2001/js/pet2001io.js`. **General fix** — offered upstream.

**What.** Declares the VIA's timer 2 latch-low register beside the other VIA
registers, and resets it to `0xff` with them.

**Why.** The emulator reads and writes `via_t2ll` but never declares it. Writing
it first quietly creates a global; *reading* it first throws
`ReferenceError: via_t2ll is not defined` and stops the machine. Most programs
never notice. Frog (P.J. Fellner's frog-crossing game, which GenX-DOS once
listed as Frogger) reads the latch before anything writes it, and crashed on
load. Thomas's current code still has no declaration.

## Change 1b — reset `via_t1_undf`, not `var_t1_undf`

**File:** `6502/pet2001/js/pet2001io.js`. **General fix** — offered upstream.

**What.** The VIA reset clears the timer 1 underflow flag under its real name.

**Why.** The reset routine wrote `var_t1_undf = 0`, a typo for `via_t1_undf`. That
assigned an unused global and left the real flag as it was, while timer 2's flag
beside it was cleared.

## Change 2 — timers take functions, not strings

**Files:** `petkeys.js` (two), `pet2001video.js` (one), `pet2001main.js` (two).
**General fix** — offered upstream.

**What.** `setTimeout("petkeyKeypressTimeout()", ms)` becomes
`setTimeout(function () { petkeyKeypressTimeout(); }, ms)`, and likewise for the
display-blanking timeout and the machine's main `setInterval`.

**Why.** A timer given a string has to compile that string as code, which a
page's Content-Security-Policy only permits with `'unsafe-eval'`. GenX-DOS tightens
its policy to allow WebAssembly compilation but not string evaluation, and under
that policy the string timers were blocked without a word: the emulator ran and
drew `READY.`, but every key it pressed stayed pressed, so the page's automatic
`LOAD` and `RUN` never arrived, and nor would anything typed.

**Why it is safe.** A string timer looks its callback up in global scope; a
function looks it up where it is written. All three callbacks are global
functions, and nothing in the files that schedule them declares anything with the
same name (the `blankTimeoutFunc` members on the emulator objects are properties,
which a bare name never reaches), so each wrapper calls exactly what the string
did.

## Change 4 — a choice of character ROM for text mode

**Files:** `pet2001roms.js`, `pet2001video.js`, `pet2001.js`. **Partly upstream,
partly ours** — the character data is Thomas's; the per-program switch is ours.

**What.** Adds the later character ROM's lower/upper case set (`petCharRom3`,
taken from Thomas's own `petCharRom2b`, January 2023) and a
`setNewCharRom(flag)` method on the emulator to choose it. Nothing changes
unless a page calls it.

**Why.** The original PET 2001 character ROM and the one fitted from the
2001-N and 3000 series on show the case of letters the other way round in text
mode — graphics mode is identical in both. A program shows its text as intended
only on the ROM it was written for: GenX-DOS's *Adventureland* (modified in May
1979) needs the original, while *Crazy Balloon*'s instruction screens appeared
as `mOVE THE SWAYING BALLOON TO THE gOAL`. Thomas ties the later ROM to BASIC 4;
GenX-DOS runs every program on BASIC 2, so it needs the choice per program
instead.

## Change 5 — held keys, and SHIFT on its own

**File:** `6502/pet2001/js/petkeys.js`. **General** — nothing changes unless a
page asks for it.

**What.** A held-key mode, turned on with `petkeySetHeldMode(true)` and fed by
a new `petkeyOnKeyUp` handler. While it is on, a key on the host keyboard stays
pressed on the PET for exactly as long as it is held, and either Shift key
presses the PET's SHIFT on its own. Keys are matched by physical position
(`event.code`), so holding Shift does not change which PET key a digit or
letter lands on. Auto-repeat is ignored, a key shared by two host keys (the
top-row and keypad `8`) is only let go when both are up, and everything is
released if the window loses focus. Typing through the command queue works as
before in either mode.

**Why.** The emulator presses each typed key for 50 ms and lets it go. That
suits typing, but a game that looks at the keyboard only now and then misses
the press — GenX-DOS's *Frog* ignored quick taps — and nothing can be held
down. And a SHIFT key could only be pressed together with another key, while
many PET games (*Space!*, *Aliens!*, *Space Ace*) fire with SHIFT alone.

The idea of a held-key "game" mode came from the PET emulator at
[commodoregames.net](https://www.commodoregames.net/), which offers one. This
implementation is written independently for this fork.

## Change 3 — GenX-DOS presentation

**Files:** `petkeys.js`, `pet2001video.js`. **GenX-DOS only** — these are our
choices, not improvements, and are not for upstream.

- **Green screen.** The video's foreground colour is `#60d0a0`, a green phosphor,
  in place of Thomas's white `#effeff`.
- **Keyboard picture at 600 pixels.** The click handlers for the on-screen
  keyboard picture scale their hit areas by 0.75, for the picture drawn 600 pixels
  wide instead of 800. GenX-DOS does not currently show that picture, so this has
  no visible effect; it is kept because it is what the bundle shipped.

---

## Offered upstream

Changes 1, 1b and 2 were offered to Thomas as
[skibo/skibo.github.io#1](https://github.com/skibo/skibo.github.io/pull/1),
rebuilt on his current code (branch `pet-fixes` in this fork). Change 3 is ours
and stays here.

## Updating GenX-DOS from this fork

1. Commit the change here, on `genx`, with a message that says why.
2. Copy the files in the table above into `systems/pet/pet2001/` in GenX-DOS.
3. Boot a game on the GenX-DOS staging site, type into it, and check that it
   autostarts.
4. Record the change in GenX-DOS's `CHANGELOG.md`.
