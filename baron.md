# Baron: the language, the allocator and the house style

Baron 0.6 or later (github.com/waitingforvsync/baron; its `docs/guide.md`, `docs/reference.md` and
`docs/zero-page-allocation.md`). This is what Rich's projects rely on and what they learned (most examples are
from Teddy Boy); the guide is the full story. Every feature below was checked against 0.6.0-1 (`4f71e73`);
`baron --version` gives the build in use.

## 1. Coming from BeebAsm: the traps

- **No `ORG`, `SAVE`, `GUARD`, `CLEAR`, `COPYBLOCK`, `MAPCHAR`.** A `SECTION name, attr = e, ...` ...
  `ENDSECTION` does it all: `org`, `guard`, `filename` (the request to save), `load`, `exec`, `transform`,
  `virtual`. No `filename`, no file.
- **`TRUE` is 1, not -1.** `AND`/`OR`/`EOR`/`NOT` are logical on two booleans, bitwise (32-bit) on two
  numbers, and refused on a mix. A boolean coerces to 1/0 in arithmetic (`(x > 5) * 10`).
- **Symbols are immutable and case-sensitive** (keywords aren't). One name, one value, per scope. A name that
  clashes with a keyword is bound as `@next = 5` and used bare.
- **Each source file on the command line is its own assembly.** `src/boot.6502` and `src/teddy.6502` share
  nothing: both define `bankcode = &00` and `bankdata = &01` themselves.
- **NMOS only.** Never `cmos = TRUE`. CMOS mnemonics (`STZ`, `BRA`, `PHX`, `LDA (zp)`) are refused elsewhere.
- **As many passes as it takes**, so forward references work nearly everywhere (a `FOR` count, a section
  attribute, `LoadAddr`'s `MAX` of a later label).
- **Errors accumulate** (`file:line:col: error: ...`); nothing is written unless every file assembled.

## 2. Syntax in brief

| Thing | Syntax | Notes |
|---|---|---|
| Statement separator | `:` | The house style writes `LDA #0:STA x` with no spaces. Braces separate too. |
| Comments | `;` or `\` to end of line | The house style uses `\\` for headers, `;` for trailing comments. |
| Label | `.name` | At column 0, even inside a scope. |
| Scope | `.name` then `{` ... `}` | Its symbols are `name.sym` from outside (`touching.down`, `draw_part.lowery`, `plot_sprites.count`). Anonymous `{ }` is private. |
| Local label | `.@`, referenced `@-` / `@+` | Nearest behind/ahead in the scope. The house style uses `@+` for short skips everywhere. |
| Numbers | `42`, `&FF`/`$FF`, `%1010`, `'A'` | A character literal is just its code. |
| Low/high byte | `LO(x)`, `HI(x)`, `<x`, `>x` | `<`/`>` take the whole following expression. |
| Hex text | `~x` | For `PRINT "at &", ~P%`. No `&` included. |
| Current address | `*` or `P%` | Of this statement. |
| Data | `EQUB`, `EQUS` (same), `EQUW`, `EQUD`, `SKIP n`, `SKIPTO a`, `ALIGN n`, `INCBIN "f"` | `EQUB` flattens lists and ranges. Paths relative to the including file. |
| Skip trick | `BITABS` (`&2C`), `BITZP` (`&24`) | Never a raw `EQUB &2C`: the allocator models the keywords, not the byte. |
| Conditional | `IF e` / `ELIF` / `ELSE` / `ENDIF` | A dead branch computes nothing. Braces after `IF` open a real scope. |
| Loop | `FOR v = 0..7` ... `NEXT` | Unrolls; each iteration its own scope. Walks lists too: `FOR a = {&FE44, &FE45}`. |
| Macro | `MACRO NAME a, b` ... `ENDMACRO` | Define before use. Own scope. Overloads by shape, quoted punctuation matched (`MACRO ADD "#" n`). |
| Function | `FUNCTION F(x) = expr`, or a body ending `= expr` | Pure, define before use. Bodies take locals, `IF`, `FOR`, `ASSERT`. Overload by arity. |
| Checks | `ASSERT e, "msg ", v`, `ERROR "msg"` | Message evaluated only on failure. |
| Logging | `PRINT "x = &", ~x` | Final pass only, so values are resolved (ZA_AUTO addresses too). |
| Predefine | `-D name=expr` | Test with `DEFINED(name)` and bind a different name for a default. |

**Lists, ranges and broadcasting** are the language's power tools:
- `0..7`, `7..0`, `0..<n`, `0..2..10` (second element sets the step), `{a, b, c}`; `..` binds looser than
  arithmetic, so `0..n-1` needs no brackets.
- Operators and element-wise functions (`LO`, `HI`, `ABS`, `SIN`, `ROUND`, `CODES`, `WHERE`...) map across lists.
  `EQUW table + (0..<32) * 8` is 32 pointers; `EQUB ABS(SIN((0..255)/128*PI))*255+0.5` is a sine table.
- Whole-list functions: `LEN`, `SHAPE`, `RANK`, `SUM`, `MIN`, `MAX`, `SORT`, `REVERSE`, `CONCAT`, `ZIP`,
  `FLATTEN`, `REPEATED(n, v)`, `FIND(x, v [, otherwise])`, `REPLACE`, `CHR`, `AS_STRING`, `AS_NUMBER`.
- Subscripts: `L[i]` drops the axis, `L[a..b]` slices (clipped), `L[{0, 4}]` gathers, `M[.., 0]` a column.
- A user `FUNCTION` maps over a list only if everything in its body does; an `IF` needs one value (use
  `WHERE(mask, a, b)` for the element-wise choice).
- A string is one value; `CODES(s)` gives its codes, `CHR(list)` makes one string.

Examples from Teddy Boy and Portal:
```
GameKeys = {&48, &68, &61, &42, &00, &49, KeyP, KeySpace}   ; : / Z X SHIFT RETURN P SPACE
ASSERT LEN(GameKeys) == 8

FUNCTION RightPixel(c) = (c AND 1) + (c AND 2) * 2 + (c AND 4) * 4 + (c AND 8) * 8
FUNCTION Mode2Byte(left, right) = RightPixel(left) * 2 + RightPixel(right)
FUNCTION TileLine(tl) = TilesOrg + tl * 128

tileruns = {tilerun0, tilerun4, tilerun8, tilerunm4}    ; a list of labels: feeds ZA_CANCALL tileruns

MACRO TILERUN offset                                    ; runs.6502: unrolled by FOR inside a macro
    FOR line = 0..7
        tl = MAX({0, MIN({TileLines-1, line+offset})})  ; (a FOR iteration is a scope: tl rebinds)
        LDA TileLine(tl),Y:STA &FF00+line,X
    NEXT
    RTS
ENDMACRO
```
`letters.6502` draws its glyphs as lists of strings (`{".##.", "#..#", ...}`) and turns them into bytes in
`FUNCTION`s: art in the source, packed at assembly time.

## 3. Sections

- **Nesting is containment.** A child's bytes land in its parent where it's written. A child with its own
  `org` is *rephased*: labels at the run address, bytes in the parent's file. `MainCode` (`org = MainCodeOrg`,
  `&121`) sits inside `Teddy` (`org = LoadAddr`, saved as `TEDDY2`): the loader unpacks it to where its labels
  say it runs.
- **Two sections may share an address**: `SwramCode` and `SwramData` are both `org = &8000`. Each keeps its own
  counter, and since 0.6 each annotation keeps to its own section (Baron #29).
- **`guard`** is an address the section must not reach, checked at each `ENDSECTION` (each piece of a
  reopened one). Overruns report how far over, and assembly carries on.
- **`transform = "zx02"`** names a `FUNCTION` (in `src/zx02.6502`) that gets the section's bytes as a list and
  returns the bytes to keep. Labels keep the untransformed addresses. It runs once every byte is final; a size
  change sends Baron round again. A transformed section can't be reopened or be virtual. The two banks and
  main RAM's code are each packed this way (D38, D47) and unpacked by `src/dzx02.6502`.
- **`load`/`exec` take 32-bit host addresses**: `load = &FFFF0000+LoadAddr` (the I/O processor).
- **Reopening**: `SECTION GameVars` written again carries on from where it stopped, wherever the piece is
  written. Takes no attributes; the first line has them and must come first. Only top-level sections, and
  nested ones that took no room (virtual), can be reopened. Not from inside itself.
- **Virtual sections** keep no bytes and need their own `org` inside a real section. Teddy Boy's workspace is
  all virtual sections declared in `memory.6502` (`GameVars`, `MainVars`, `Workspace`, `PickRam`, `Tables`,
  `MoreTables`, `Mirrored`; `SoundVars` in `sound.6502`) and **claimed where they're used**:
  ```
  SECTION GameVars                      ; player.6502: the shots' state, next to the code using it
  .ShotState                                ; 0 none, ShotFlying, ShotSparkle
    SKIP MaxShots
  .ShotX                                    ; its left byte across
    SKIP MaxShots
  ENDSECTION
  ```
  A section opens no scope: a label claimed inside a routine's `{ }` belongs to that routine's scope.
  Code in a virtual section is ignored by the zero-page allocator.
- A top-level section takes no room in the default section; code outside any section can't be saved.

Teddy Boy's tree (`src/teddy.6502`): `Banks` (`TEDDY1`) holds `SwramCode` then `SwramData`, each packed;
`Teddy` (`TEDDY2`) holds `MainCode` (packed, rephased to `&121`) then `entry.6502`'s loader. `!BOOT` is
`src/boot.6502`, a separate assembly.

## 4. The zero-page allocator

`ZA_POOL &02..&FB, &FD..&FF` (`memory.6502`): `&00`/`&01` are the banks' numbers `!BOOT` writes; `&FC` is
where the MOS's IRQ entry saves A (its entry is still used). The pool is used to about `&C2`; Baron packs
everything else.

**Declaring.** `ZA_AUTO1 a, b`, `ZA_AUTO2 ptr` (low at `ptr`, high at `ptr+1`), `ZA_AUTO n, buf`. Declare in
the **smallest scope** that uses it. A routine publishes inputs and outputs as its own scoped variables
(`LDA scale.result`); a value two routines hand over in place gets one shared name instead (no relay copy).
The listing shows each choice as `name = &70 [auto]`; `grep '^name = ' build/teddy.lst`.

**What it guarantees.** Sharing follows liveness along the real control flow, not braces. A value live across a
`JSR` is kept off every byte the callee (and its callees) touch. An argument is protected until its call; a
result from its store to the caller's read. OS calls are free. A pointer whose high byte is written once and low
byte refreshed keeps its high byte. Paged banks may share bytes.

**What it can't see, and the annotations** (each is a promise; a wrong one silently misallocates):

| Annotation | When | In Teddy Boy |
|---|---|---|
| `ZA_CANCALL a, b` | After a self-modified or computed `JSR`: every routine it can reach (a list symbol works). | `JSR &FFFF : ZA_CANCALL states` (enemies), `ZA_CANCALL tileruns` / `zerorun` / `tilerunsplit` (draw), `ZA_CANCALL comprun0, comprun2` (sprites). |
| `ZA_CANJUMP a, b` | After an indirect or self-modified `JMP`, a self-modified branch, or an RTS/RTI dispatch. Replaces the literal target. | `BVS irqscrntop : ZA_CANJUMP irqscrntop, irqscrnbottom, irqscrnverybottom` (the IRQ's patched branch); `JMP &FFFF : ZA_CANJUMP pagepatches`; the sound's command dispatch. |
| `ZA_UNREACHABLE` | After an always-taken branch, or a `JSR` that never returns. | After every always-branch that has data (tables) after it: `BPL next ; always` then `ZA_UNREACHABLE`. |
| `ZA_RETURN` | After a jump that is really "back to my caller". | A routine left by an RTS patched in at run time: give it a visible exit, `JMP block : ZA_CANJUMP block : ZA_RETURN` (review 8). |
| `ZA_RETURNTO t` | Caller side: a data-consuming callee resumes elsewhere. | (unused) |
| `ZA_DISCARD v` | A variable rebuilt by indexed stores: its old value is dead here. Above a label: that path only; below: every path. | |
| `ZA_WIPE` | Straight after a whole-pool clearing loop. | |
| `ZA_INDEXEDBY 0..5` | After an indexed access to a `ZA_AUTO`: the index set. Turns the warning into a bounds check. | `LDY recfpos,X:ZA_INDEXEDBY 0..5` (sound); `LDA queue,X:ZA_INDEXEDBY 0..MaxSprites-1`. |
| `ZA_ENTRY` | A routine the outside world calls. One anywhere makes the markers the complete list of ways in. | `boot.6502`, `entry.6502` (the loader). |
| `ZA_INTERRUPT` | An interrupt handler: its variables are kept apart from the main program's. | `irq`, and `!BOOT`'s splits handler. |

**Rules and errors worth knowing:**
- A `ZA_AUTO`'s address can't shape the program: no `IF v`, `SKIP v`, `FOR n = v..8`, `org = v`
  ("Cannot use a ZA_AUTO address here"). Only `+n`/`-n` and `~` work on it.
- A variable nothing reads or writes gets no address and no definition (a warning; using its address is an error).
- `(v),Y` needs a `ZA_AUTO2`. `v+n` must stay inside the variable.
- An unannotated indirect `JMP` through your own vector is refused while anything is live; an RTS dispatch is
  silently trusted (annotate it).
- A recursive routine can't hold a per-level fresh value across its recursion ("freshly written and held live
  across recursion"). That error is also the symptom of a **bogus call edge** (see Baron #34 below).
- A multi-byte variable shared with the IRQ can be caught half-updated: keep shared state to single bytes.
- Stray pointers or self-modified operands aimed into the pool are invisible: keep fixed-address stores
  (screen, `&FE30`, workspace) out of `&02..&FF`.
- `--warn 2` flags stores into the pool at fixed addresses.

## 5. Traps found in practice

- **A local label shadows a global of the same name.** A loop label `.round` inside a routine made
  `LDA round` load the label's address, not the zero-page variable. Name loop labels distinctly from any
  variable or routine.
- **Annotations in banks cross-talked before 0.6** (Baron #29): a `ZA_CANCALL` at `&A220` in the data bank
  applied to a plain `JSR` at `&A220` in the code bank, so a live `c` shared a byte with the callee's `edge` and
  the game hung. Fixed in `7d663a7` (annotations carry their section); the main-RAM stubs and
  `tools/za_crosstalk.py` went (D45). An older Baron silently misallocates this source: the Makefile asks for
  0.6 or later.
- **Always-taken branch before labelled data** (Baron #34, fixed `dd240da`): the branch's not-taken edge fell
  into a block of size-0 labels whose fall-through hopped the data into the next code. Moving code then made a
  routine "fall into" `bonus_game`, giving bogus call edges and recursion errors. Rule now: put
  `ZA_UNREACHABLE` after every always-branch that has data after it (all three such places have it).
- **A runtime-patched RTS hides the exit.** A routine whose only exit is an RTS patched in at run time looks
  to Baron like it never returns, so callers' live variables were shared with the callee's
  (`start_round.col` with `draw_column.col`, review 8). Give it a visible exit (`JMP x : ZA_CANJUMP x :
  ZA_RETURN`).
- **Annotate a self-modified `JSR &FFFF` even though `&FFFF` looks like a harmless OS address**: unannotated,
  Baron reads it as an OS call that touches nothing.
- **A fall-through from a scope-less label into a scoped routine** is analysed as the CPU runs it (braces
  fence nothing). That is right, but check the listing when one changes.
- **Look at the listing.** A relay that loads and stores the same byte (`LDA a.result : STA b.input`, both
  `&70`) is wasted; fuse it by naming one variable. You can't remove it with an `IF` on the addresses.

- **A `ZA_AUTO` and a label can't share a name in one scope** ("Duplicate symbol").
- **`LDA #LO(v)` on a `ZA_AUTO` is refused** ("Incompatible types"): write `LDA #v`. Only `+n`, `-n` and `~` work on
  an allocated address.
- **`INCLUDE` paths are relative to the including file**, not the command line.
- **The `INCLUDE` order sets the addresses** in a reopened virtual section: moving one include moved 113 of 123 of
  Teddy's `GameVars` and changed its page crossings.
- **Overlapping virtual sections are not an error**: to carry on a pool, reopen its section rather
  than start a new one at the old one's end label.
- **Leave `&FC` out of the pool** while the MOS's IRQ entry is in use. `ZA_POOL &00..&FC` (as some Baron examples
  write it) includes `&FC`: `..` is inclusive. Rich's projects write `ZA_POOL &00..&FB, &FD..&FF`.
- **Baron takes an OS call as touching nothing**, but the pool runs over the OS's zero page: until the loader owns
  the machine, nothing in the pool may be live across an OS call or an interrupt in the OS's bytes. `!BOOT` (OS
  alive) uses `ZA_POOL &70..&8F`.
- **Check the `[auto]` lines in the listing by hand** whenever they change: no two variables share a byte while
  both are live. The allocator has had bugs (#29), and a behaviour trace after every change caught them.
- **Cycle-exact loops must not cross a page**: `IF HI(P%) <> HI(P%+n) : JMP loop : ALIGN &100 : ENDIF` before,
  `ASSERT HI(loop) == HI(P%)` after (Teddy's and exhale's vsync sync).

## 6. Building, listing and symbols

- `make` runs `baron -v --symbols build/teddy.json -o build/teddy.ssd --title YYMMDD-HHMM --opt 2
  src/boot.6502 src/teddy.6502 > build/teddy.lst`.
- `baron --check file.6502` assembles and writes nothing. `-vv` is the uncut listing.
- The listing shows each label's address and every assignment's value (`PLAY_R7 = 28 [34 - 4 - 2]`): grep it
  for addresses. `build/teddy.json` holds every symbol by section (`za_autos` included); filter keys with `@`.
- `PADTO`/`PADTOLOW` `PRINT` their waste ("PADTOLOW &1A: 17 bytes free at &..."): grep the listing for the
  free bytes in each gap. A transformed section prints `; transform "zx02": N -> M bytes`.
- `--warn 3` lists every branch that crosses a page; `tools/pagecross.py` does it from the listing, and
  `tools/branchprof.py` weighs them by how often they're taken.
- **"Branch out of range"**: first move code so the branch reaches (put the target next to its branches,
  reorder blocks, share a nearby RTS, put a short block after the long one). Only invert it round a `JMP` when
  nothing else works, and say which you did (Rich, 2026-10-28).

## 7. House style

Formatting, naming, comments and structure are in `style.md`.
