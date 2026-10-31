# 6502 optimisation on the BBC Micro

What paid, what didn't, and how a speed-up is proved. Most examples are from the Teddy Boy port
(fifteen optimisation reviews, in its `docs/review-optimisation-*.md`); the costs are general.
"C/D/M" below are the code bank, data bank and main RAM; cycles are a frame unless said otherwise.

## 1. The method

- **Measure, don't estimate.** Estimates, Rich's own ideas included, were wrong often enough that nothing is
  taken unmeasured. Report losses as plainly as gains.
- **Time with the real interrupts.** A py65 harness that didn't charge the IRQ's time showed a gain that b2
  showed as a loss, because the game's time-based decisions reacted to it. Time in an emulator with real VIA
  timers and IRQs (headless jsbeeb in Teddy: `tools/jsbeeb_rate.mjs`), never by polling from the host clock.
- **Prove a speed-up exact.** It must make the same decisions and draw the same screen: hash the screen and
  state after each part of a frame in a cycle-independent run (Teddy's `diffpick.py`), and every set-up
  (`setuphash.py`). Arithmetic rewrites are brute-forced over every input. If anything reads a free-running
  timer (Teddy's `random` reads User VIA T1), any cycle change alters play: hook the timer read to a function of
  frame and read count before comparing.
- **No part gets slower.** Small rises are usually page alignment: re-place rather than accept them.
- **Speed against memory.** Show the cycles saved and the bytes it costs (by region) before committing. Take it
  if it's faster *and* smaller, or a significant speed-up for its bytes; otherwise hold it and let the user decide,
  and note a marginal keep as a removal candidate. Quote cycles saved per byte spent (3-20 was the range taken;
  about 1 only when it fixed the worst frames). Main RAM is the scarcest.
- **Judge by the bad frames.** Where the engine spends saved time (a deferral that plots more when there's
  time), the mean barely moves: judge by frames over budget, things left out, and so on.

## 2. The cost model

At 2 MHz a PAL frame is 312 × 64 µs = 39,936 cycles; a 25 fps frame 79,872.

| Operation | Cycles |
|---|---|
| `LDA #`, `TAX`, `INX`, `CLC`, `ASL A` | 2 |
| `LDA zp`, `STA zp`, `BIT zp`, `PHA`, `PHP`, `JMP abs` | 3 |
| `LDA abs`, `LDA abs,X` (+1 crossing a page), `STA abs`, `PLA`, `PLP` | 4 |
| `LDA (zp),Y` (+1 crossing), `STA abs,X`, `INC zp`, `JMP (ind)` | 5 |
| `STA (zp),Y`, `INC abs`, `JSR`, `RTS`, `RTI` | 6 |
| Branch | 2 not taken, 3 taken, 4 taken across a page |
| Interrupt entry | 7, then the MOS's own IRQ entry before IRQ1V |
| `&FC00`-`&FEFF` on the 1 MHz bus | +1 or +2 a read or write, by phase: CRTC/ACIA/serial (`&FE00`-`&FE1F`), both VIAs (`&FE40`-`&FE7F`), ADC (`&FEC0`), FRED, JIM. Not the Video ULA or ROMSEL (`&FE20`-`&FE3F`), the FDC, Econet or the Tube (beebjit's table, from b-em). |

**Cost per byte moved**, as measured in Teddy's runs:

| Run | Per byte |
|---|---|
| `STA abs,X` alone (blank store, page patched in) | 5 |
| `LDA abs,Y : STA abs,X` (tile line in the operand, page patched in) | 9 |
| `LDA abs,X : STA (zp),Y : INY` | 12 |
| `LDA (zp),Y : STA (zp),Y` with `LDY #n` / `INY` | 13 |
| Masked composite `LDA (spr),Y : STA m+1 : LDA tile : AND MaskTab : ORA (spr),Y : STA (scr),Y` | 30-32; 16 if marked opaque |

**The machinery round the runs costs more than the bytes.** Of the plotter's 11.3 ms a frame only 4.8 ms
moved bytes; a band cost 150-180 cycles of set-up, a run call about 180 plus 56-60 a column, an erase
rectangle 40-85 a byte in fixed cost. Most wins cut per-call, per-band and per-column overhead.

## 3. Techniques that paid

### Fast from the first line (Portal's written rules)
Every routine is written as fast as it can be from the start, not tidied up later. Weigh every line by what
it costs on a 2 MHz 6502.
- **Values in registers, not zero page**, wherever they fit: a count in X, counted down by `DEX:BNE`, needs no
  store to set up and costs 2 cycles a time; a `pages` variable needs the store and a 5-cycle `DEC`.
- **No copy made only to be read** (read it where it is), and **no copy made only to be changed** when nothing wants
  the original: work on it in place and list it as corrupted.
- **Follow the flags, C most of all.** Loads, stores, transfers, `INX`/`DEX`/`INY`/`DEY`, `INC`/`DEC`,
  `AND`/`ORA`/`EOR` and `BIT` keep C. Hoist `CLC`/`SEC` out of a loop whose body leaves C known at its branch back
  (`CPY #n:BNE loop`, going round only under n, leaves C clear). A value can live in C: never let a `CMP` sit
  between an `ADC` and the `ROR` that wants its carry. A known C folds a constant: `EOR #&FF:ADC #k+1` with C clear
  is k - x. A shift's carry is known when the bit it shifts out is.
- **Follow the registers**: no load of a byte a register already holds; `TAY` where A is known 0, not `LDY #0`.
- **Shift a multi-byte value with one byte in A**: `ASL A:ROL dv+1`, not `ASL dv:ROL dv+1`; in a loop
  `LSR q+1:ROR A` (3 cycles a bit better than `ROR q`), stored once at the end.
- **An index that's one short stays as it is**: fold the one into the address, `LDA table+1,X`, never `INX` first.
- **Unsigned magnitudes with the sign picking the path; never negate a product** (fuse the sums so terms go in by
  add or subtract).
- **Bytes where bytes will do, tables instead of multiplies, work hoisted out of loops.** But before adding a table,
  see whether arithmetic on one already there does it (6 cycles more against two 65-byte tables, used rarely).
- **Hot multiplies inline as macros**; their rare callers share one routine that expands it. A macro taking a bound
  (`LOG2_16 bits`) gets the tightest bound the value can reach: the bound picks the expansion.
- **Hot branches followed by `SAME_PAGE target`**, which fails the build if the branch crosses a page. Fix one by
  moving code, not padding; code carrying these checks goes first in its section, before data whose size changes.
- **Never a base before a table (`table-1,X`)** unless `FITPAGE_FROM` puts the base on the table's page: with the
  table on a page boundary every access pays the crossing. Number tables from 0, or keep element 0 unused.
- **The rare case out of line**: in a hot loop the common path falls through, and the rare step (up a character
  row) sits after the loop and branches back `; (always)`.
- **Code that runs once** (the loader) can be a macro, inlined and self-modifying (`abs,Y` with the operand's high
  byte stepped by `INC`) rather than a zero-page pointer.

### Unrolled runs, entered and left by patching
- Unroll the inner loop once (a band's 8 lines, a column's 24) as a run of identical steps. Enter by patching
  the calling `JSR`'s operand from a step table (made by the loader); leave by storing `RTS` (`&60`) over the step
  after the last, and put the step's first opcode back afterwards. No byte pays loop overhead.
- **Every step starts with the same opcode**, so the put-back is one constant store. Give the step table a
  spare last entry pointing at a dummy byte, so "stop after the last line" writes harmlessly.
- **A run that ends at its own end isn't patched**: its own RTS ends it (-125 and -141 cycles in two reviews).
- **A whole band skips the patching**: `TAY:BNE partial : CPX #7:BNE partial : JMP sprfast` (-55 a band).
- **Share one index between source and destination**: point each column's pointer 8n bytes *before* its data,
  so `LDY #8n+line : LDA (cols+2n),Y : STA (scr),Y` walks six columns and the screen cell with one Y. A 6-wide
  band is one call.
- **Self-modified absolute stores beat `(zp),Y`**: `LDA TileLine(tl),Y : STA &FF00+line,X` with the page patched
  into the operands (9 cycles a byte against 11-13). `abs,X` carries within a page for free; only a page change
  costs a patch. Split any wrap (the 16K screen wrap) out at set-up, never in the inner loop.
- `JSR` straight at the run, never through a `JMP &FFFF` trampoline: 3 cycles a call.

### Data laid out for the inner loop
- **Line-planar, page-aligned tiles**: line `tl` of tile `t`, byte `s` at `TilesOrg + tl*128 + t*3 + s`. One index
  picks tile and byte, the line is in the operand, nothing crosses a page and no pointer is needed. Store the map
  as `tile × 3` so nothing multiplies.
- **Emit the final form from the tools** (MODE 2 bytes, tile × 3, trimmed frames), not data the code converts.
- **Trim data to what's drawn** (sprite frames to their own lines: 6.5% fewer bytes plotted).
- **No indexed read crosses a page**: `INPAGE n` before a table, frames laid out wholly within a page (and 7 bytes
  in, since the pointer sits up to 7 before them). Teddy's frame-read crossings went 15.7 a frame → 0.04 for
  100 bytes of pads.

### Marks in the data
- Spare bits that mean something to the hardware anyway: in MODE 2, logical colours 9-15 set to the same
  physical colours as 1-7 give each pixel a free bit 3. Teddy marks a sprite byte whose two pixels are both
  opaque with `&C0`, and the composite step tests the byte it loads anyway:
  ```
  LDA (cspr),Y : BMI opaque : STA mask+1 : LDA TileLine(tl),X
  .mask AND MaskTab : ORA (cspr),Y
  .opaque STA (cscr),Y : INY
  ```
  -1,525 cycles (3.4%) for 64 bytes.
- **The rule**: a mark pays only where the byte is loaded anyway and the mark replaces a lookup or a branch. A read
  costs about what a store does, so a mark read *instead of* storing saves nothing.

### Tables (built by the loader, not stored)
- 256-byte **mask table on a page**: store the sprite byte into the low byte of an `AND MaskTab` operand, leaving X
  free for the background byte.
- **Divide and modulo tables** (÷3, mod 3, ÷12), reused for more than one job (splitting a count into whole tiles
  and a remainder).
- **Wrapped copies past the ends** remove wrap tests: a column table 8 entries past its end, row tables 6 past;
  copies *before* the table too, so a biased coordinate indexes from -8 directly. A map buffer with its first
  rows repeated after its last makes "the row below" always +32 (`TYA:ADC #32:TAY`).
- **Cache derived values** where their inputs change (a wall's reach, "changed since drawn", "on tiles last
  time" as a hint that gates a test). A hint may only gate an optimisation, never change the result.
- Small tables for multiplies (`×36` by lookup), `1 << n` bits, screen row addresses.
- **Quarter-square multiply**: a × b = f(a+b) - f(a-b) with f(n) = n²/4, from page-aligned `SqLo`/`SqHi` tables
  read through pointers whose low bytes are a and a EOR &FF: `SEC:LDA (sqptr),Y:SBC (sqnegptr),Y` (starglobe's
  `demo.6502` derives it; Portal's `tables.6502` builds the tables by a recurrence).
- Overlapping tables and strings where their bytes agree, each overlap pinned by an `ASSERT`.

### Cutting the machinery
- **Step state; don't recompute it** (pointers stepped by 8 a band, the phase's next state copying only what
  changed).
- **Work something out once a plot, not per band**; repeating an idempotent OR (a tile's lines twice) can drop a
  loop and its count.
- **One loop per case** instead of a general loop testing a mode; better still, remove the mode.
- **The cheapest run for the easy case**: an empty background column goes through the plain copy (13 against 30
  cycles a byte).
- **Patch loop-invariant operands once** (an `ORA #` immediate, an `LDX` operand) rather than reloading per column.
- **Test a wrap by the high byte only**, and the 16K screen wrap with the carry known clear: `BPL @+ : SBC #&3F`.
- **Unroll pointer set-up as a running sum** under `FOR n`, the low bytes chained in A, the high byte riding in Y.
- **Skip work a later stage overwrites anyway** (erase only what the new plots won't cover).
- **Reject cheaply first, inline**, before the `JSR` to the full test; scan in the order that finds the common case
  first.

### Flags and the carry
- **Always-branches on known flags** instead of `JMP`, each commented `; always` (and why): `BNE` after storing a
  high byte that can't be 0, `INC ptr+1 : BCS` where the carry is already known set (`INC` leaves C alone),
  `CLC:BCC`, `BNE` after a callee known to return Y ≠ 0. In warm code only when both ends are close, or a later page
  crossing costs more than the `JMP` saved.
- **Reversed subtract**: `EOR #&FF:SEC:ADC x` is x - A; `CLC:SBC x` is A - x - 1.
- **Constants that absorb a known carry**: `ADC #MapWidth-1 ; (carry set)`.
- **A compare that makes the carry for the next add**: `LDA frac,X : CMP #Past : LDA row,X : ADC #Down` adds a
  fraction and its carry with no division.
- `CMP #&80:ROR A` is a signed halving; `LDA #0:ROL A` turns C into 0/1; `SEC:ROR flag` sets bit 7;
  `TXA:ADC v:ROR A` a 9-bit average. `BIT flags : BMI/BVS` tests two flags at once; `LSR A:BCS` splits a case.
- **Let the load set the flag**: an empty tile is 0, so `LDX tile` sets Z for the branch.
- **A loop index left as a flag**: a `DEX:BPL` scan leaves X = `&FF` if nothing was found.
- **Document carry and register contracts** in routine headers (`carry: clear`, `X: preserved`) and keep them;
  callers rely on them.

### Loops, calls and the stack
- Count down and test at the bottom (`DEX:BPL`, `DEC n:BNE`); no `JMP` back.
- Walk a bit mask with `LSR` on memory: `INX : LSR m : BCS found : BNE next` (Z ends it, C finds it).
- Unroll short fixed loops (1-3 iterations) when the bytes are few.
- Count up from a negative index to 0 so `INX:BNE` ends the loop with no compare:
  `LDX #-3 AND &FF` ... `LDA Best-(-3 AND &FF),X` ... `INX:BNE compare` (Critical Mass).
- `BITABS` (`&2C`) skips the next 2-byte instruction, so two entry points share a tail with no jump
  (`.on LDA #&C0 : BITABS : .off LDA #&F0 : STA ...`).
- `PHP`/`PLP` carries a flag across code that changes it.
- Patch an operand once a call rather than going through a zero-page pointer, where the loop then runs many times
  (`.mand AND #0 ; self-modified: NOT mask`).
- **Dispatch**: number states from 1 so `LDA StateLo-1,Y:STA call+1 : LDA StateHi-1,Y:STA call+2` needs no
  `DEY`, then a patched `JSR &FFFF` with `ZA_CANCALL`. `JMP (zp)` measured no better.
- **Tail calls**: `JSR x:RTS` → `JMP x`; a `JMP` to the next routine → fall through; arrange routines so one falls
  into the next. Share tails (`CLC:RTS`) only in cold code or where it costs no cycles.
- **No stack temporaries.** Teddy's 32-byte stack (to leave main RAM for code) overflowed into the IRQ's code.
  Use a zero-page local or read the byte again. Tail `JMP`s keep the depth down. A sentinel under the stack,
  checked every frame, catches overflow.

### Zero page
- Hot pointers in zero page for `(zp),Y`. Let Baron's allocator pack the rest (`baron.md`): scope variables
  tightly; three 2-byte pointers share better than one 6-byte block; a block live across calls can't share.
- Big per-frame scratch can go to main RAM (`abs,X`) to hold zero page's high-water mark.
- **Self-modifying hot loops go in zero page if there's room** (Rich, Critical Mass D24): each patch becomes a
  3-cycle, 2-byte zero-page store, and a loop there can't cross a page. Assemble it as a nested section
  (`SECTION ZeroCode, org = ZeroCodeOrg, guard = &FC`) and let the loader copy it down.
- In zero-page code an instruction's operand is itself a zero-page pointer (flow): `sprite = P%+1:LDA &FFFF,Y`,
  and elsewhere `LDA (sprite),Y` reads through the same two bytes.
- Keep the IRQ off X and Y and restore A with `LDA &FC:RTI` (the MOS saved it there).

### Placement
- Branches crossing a page cost a cycle when taken, and indexed reads crossing one cost a cycle. Profile how often
  each is taken (`tools/branchprof.py`) and place blocks by anchors (`PADTOLOW lo`, `PADTO addr`, `INPAGE n`) at
  the offsets that cost least (`tools/placement.py`). Teddy: 171 → 38 cycles a frame.
- Moving cold code or data shifts the hot code after it: keep variable-length data (a recorded demo) last before
  an anchor so its length moves nothing; take cold code from a bank's tail, not its middle.

### The big wins were algorithmic
- Use the time otherwise spent waiting for the beam: plot the next frame's sprites in the wait, each once the beam
  has passed its place for the last time (the frame's work 23.2 → 16.3 ms).
- Repair only what was drawn over (over-budget frames 3.29% → 0.70%).
- Work out overlaps once a frame with a broad phase (a near mask per slot).
- Update only what changed (the time bar's one lost byte: 3,529 → 127 cycles).
- Run the logic once a frame at doubled speeds, each move's sign and size known (7,074 → 4,163 cycles).

### Memory levers that cost no speed
- Pack the images with ZX02 (a Baron section transform) and unpack in place at load.
- Move `SKIP`-reserved state out of a ROM-bank image into main RAM's gaps as virtual sections.
- Put cold code in the pads before anchors.
- Code runs as fast from sideways RAM as from main RAM: only code that must see both banks, or that the IRQ runs,
  needs main RAM.

## 4. Measured as losses (don't redo these without a new reason)

- **One compositor for a pile of overlapping sprites**: byte-exact but twice the plot (the union fragments into
  ~6-byte pieces, each a run call). Overlapping sprites are plotted back to front, masked only in the overlap.
- **Each sprite drawn once, back to front**: +760 cycles (cutting bands costs ~480 a band).
- **Reading the screen to skip work** (an occupancy bit, "destination already 0", "store unchanged"): a read and
  compare costs more than a 6-cycle store, and most cells hold the sprite's own last image anyway.
- **Skipping blank stores over an already-blank screen**: wrong over lines that had been hidden; nothing once safe.
- **Wide unrolling** (6-column composites, masked fast runs, two copies of a run, triple tile runs): 0.3-0.5 ms for
  0.3-3K. Measure cycles per byte; most were under 1.
- **Fusing the erase into the plot**: net about 0.
- **`JMP`-in/`JMP`-out runs** without `JSR`/`RTS`: Baron's liveness can't follow them across scopes.
- **Interleaved 4-line chunks via `(zp),Y`**: 11-13 cycles a byte against 9 line-planar.
- **A cheap path for tiny sprites**: the test for it cost nearly what it saved.
- **Sharing code that adds a `JSR` to a warm path**: measure per patch; two such shares were taken back out.
- **Undocumented opcodes (`LAX` and so on)**: not used.

## 5. Waste checklist (Teddy's review 15: 74 exact patches, 1,102 bytes)

1. A conditional branch over a `JMP` whose target a branch reaches: invert it, drop the `JMP`.
2. A `JMP` to the next instruction or routine: fall through. `JSR x:RTS`: `JMP x`.
3. A `JMP` taken on a known flag: a branch, commented `; always` with why.
4. `CLC`/`SEC` where the carry is already known (after an add that can't overflow, after a taken `BCC`/`BCS`,
   after a matching `CMP`, on entry where every caller leaves it known).
5. Reloading what a register holds (`LDA` after `CMP` of the same, `LDA #0` with A = 0, `STA v ... LDA v` →
   `TAY ... TYA`, `LDX en` after callees that keep X). Check every exit path of the callee first.
6. Copying a caller's values into a callee's variables when the caller could write them there, or the callee
   could read them in place (Rich's first rule).
7. A variable that's always one constant: an immediate.
8. State written but never read; dead code (report surprising dead paths as bugs, don't just delete them).
9. Set-up overwritten before it's read; zeroing what the loader already zeroed.
10. Repetition in cold code: share it (never at a warm path's expense).
11. Round trips that undo themselves; wrap tests the extended tables already cover.
12. A table that's really a constant or derivable from another; tables only the loader reads.
13. Arithmetic in fewer steps (a modular window as one add and two compares), brute-forced equal.
14. Several read-modify-writes of one flag byte merged into one store.
15. `SKIP` space inside a bank image: move it to a main-RAM virtual section.
16. Branches and tables sitting on page boundaries: re-place after any byte change.
17. Stale comments, unused constants and labels.

Out-of-range branch: move code so it reaches before inverting round a `JMP` (`baron.md` §6).
