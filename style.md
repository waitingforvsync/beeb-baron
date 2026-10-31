# Rich's house style for Baron 6502

The written rules are Portal's `codingstandards.md` (6502 and C): read it when starting work in a
new project. Teddy Boy and Critical Mass follow the same rules with two differences, settled here for new code
(Rich, 2026-10-30):

- **Indent with 4 spaces**, never tabs (as Portal and the Baron examples). Teddy Boy and Critical Mass use tabs.
- **CamelCase for constants, `FUNCTION`s and anything in a memory section** (tables, workspace, variables
  outside zero page), as Teddy Boy and Critical Mass. Portal uses UPPER_SNAKE constants and lower-case tables.

**In an existing project, match its files**: Teddy Boy and Critical Mass stay tabs, Portal stays UPPER_SNAKE.
Never reformat a file wholesale.

## Layout

- Mnemonics upper case; hex `&` with upper-case digits; `%` binary for masks. Shifts name A: `ASL A`, `ROL A`.
- **No spaces round statement colons**: `LDA #15:LDX #0:JSR &FFF4`. Join what reads as one operation: a load
  and its store, a compare and its branch, a short run (`TXA:ASL A:ASL A:ASL A:STA t`). The exception is a
  one-line assembler `IF`: `IF NOT(down) : CLC : ENDIF`.
- **No spaces inside an operand**: `STA ptr+1`, `LDA tab+2,X`.
- `SECTION` attributes have spaces round `=`: `SECTION Boot, filename = "!BOOT", org = &400`.
- **Labels at column 0 on a line of their own**, loop labels and labels inside macros too. The one exception: an
  anonymous `.@` after the instruction it skips, `BCC @+:INC ptr+1:.@`.
- **Every routine braced**: label at column 0, `{` at column 0 on the next line, the body indented one level,
  `}` at column 0.
- **Trailing `;` comments on one column, column 40** (counting from 0); 4 further out where code runs past it.
- Lines to about 100 columns; comments wrapped to fit. A long macro call or `SECTION` line can run over.
- **Blank lines**: `ZA_AUTO`s, then a blank line, then code. A routine's parts are separated by a blank line, a
  `\\` comment saying what comes next, and another blank line. One blank line, then the next routine's header.
- Use `.@` with `@+`/`@-` for short skips, never `P%+n`.

## Naming

| What | Case | Examples |
|---|---|---|
| Routines | lower_snake | `make_tables`, `sprite_plot`, `sound_frame`, `psg_write` |
| Labels inside a routine | short lower case, run together | `.loop`, `.keyloop`, `.nextpair`, `.done`, `.found`, `.over` |
| Zero-page `ZA_AUTO`s | short lower-case nouns | `base`, `keys`, `framectr`, `eorptr`, `t` |
| Constants | CamelCase | `ScreenColumns`, `StackTop`, `FramePace`, `MaxSprites` |
| `FUNCTION`s | CamelCase | `RightPixel`, `Mode2Byte`, `TileLine` |
| Tables and variables in memory | CamelCase | `MaskTab`, `ShotState`, `BallXLo`, `SqHi` |
| Sections | CamelCase | `SwramCode`, `MainCode`, `ZeroCode`, `GameVars`, `Tables` |
| Macros | UPPER (underscores between words) | `PADTO`, `INPAGE`, `SAME_PAGE`, `FITPAGE`, `COPY` |
| Files | lower case, one word | `entry.6502`, `irq.6502`, `memory.6502`; the top level is the project's name |

- **Case says where a thing lives**: lower case is a code label or zero page; CamelCase is a constant or memory.
- **Whole words**: `major`, `logmajor`, `xsign`, not `maj`, `lmaj`, `sx`. Very local temporaries may be a letter
  or two (`t`, `k`).
- Labels name the event or place reached (`.done`, `.edged`, `.positioned`), not an action.
- Lo/hi split tables end in `Lo`/`Hi` (`SqLo`/`SqHi`, `ChanPeriodLo`); a pointer ends in `ptr` (`eorptr`). A
  two-byte zero-page value is one `ZA_AUTO2` read as `x` and `x+1`, never `x_lo`/`x_hi`.
- A self-modified operand is reached through the label on its instruction: `behave+1`, `loop+2`.
- **Never give a loop label the name of a variable or routine**: a local label hides a global of the same name
  (`.rpost` turned `STA rpost,X` into a store over the loop; `.round` made `LDA round` load an address).
- Hardware registers are **literal addresses**, named in a trailing comment: `STA &FE4D ; system VIA: IFR`.
  Never symbolic names like `SYS_IFR`.

## Comments

- **Whole-line comments are `\\`; trailing comments are `;`.**
- **Terse**, British spelling, built from possessives, colons and parentheses. No jokes in the projects (the
  examples allow a wry aside). Comments say why, or what holds; never restate the instruction.
- **File header**: `\\ name.6502 - what's in it`, two lines at most.
  ```
  \\ irq.6502 - the interrupt handler, in main RAM so it's there whatever's paged in at &8000.
  \\ Vsync is its only source.
  ```
- **Routine or macro header**: what it's for in two lines at most, a blank `\\`, `Inputs:` with an indented list
  (5 spaces, one to a line), a blank `\\`, `Outputs:` likewise. What it preserves or corrupts goes under
  `Outputs:`, flags too when callers rely on them. "none" is written out. Then a blank line, then the label.
  ```
  \\ The vsync: show the frame drawn, if asked, count it, and gather the keys held.
  \\
  \\ Inputs:
  \\     base: the buffer drawn into
  \\     request_swap: 1 to show it
  \\
  \\ Outputs:
  \\     base: the other, if it was shown
  \\     request_swap: 0
  \\     A, X, Y: preserved
  ```
  Other forms seen: `none: it never returns`; `A: 8, Z clear (sound_frame's BNEs rely on it)`;
  `carry: clear`; `odddata+1, evendata+1: the data for odd and even lines (self-modified)`.
- **A part comment** is lower case, a noun phrase, often `topic: detail`, no full stop:
  `\\ the machine: no OS from here on, and no interrupts but strays`.
- **Each `ZA_AUTO` gets a trailing comment**: `in:`, `out:` or `local:`, and what it holds:
  `ZA_AUTO1 height                     ; local: the art's rows`.
- **Flag and register facts in parentheses**, the most recognisable habit: `; (always)`, `; (C clear)`,
  `; (X = 0)`, `; (always: psg_write leaves 8)`, `; (INX, TXA and EOR keep the carry)`.
- **Self-modified code** says so on the patched instruction, and what goes there:
  `JSR &FFFF ; self-modified: the state's frame`, `INC loop+2 ; the operand's high byte`.
- How a routine works goes in its trailing comments, if anywhere, not the header.
- Cross-references in parentheses: `(D24)`, `(as starglobe's)`, `(sprite.c's pt_sprite_win)`.
- Cycle counts: per instruction in cycle-exact code (`STA &FE4D ; 6`, a total at the end, as exhale's sync loop);
  otherwise in prose ("each patch is a zero page store, 3 cycles and 2 bytes").

## Zero page

- **`ZA_AUTO` for every zero-page variable; none placed by hand** (bar fixed hand-offs like `!BOOT`'s bank
  numbers). The pool is the `ZA_POOL` in `memory.6502`, and leaves out `&FC`.
- Declare each next to where it's used, in the smallest scope: after the `{` for one the whole routine uses,
  after a part's `\\` comment for one only that part uses. Each part gets its own, named for what it holds there.
- Inputs and outputs in the routine's own scope; callers write `routine.var` (`LDA instance_xlo,Y:STA
  sprite_plot.x`). A one-shot argument nothing wants after the call can be worked on in place and listed as
  corrupted.
- State several routines share is a `ZA_AUTO` at file scope, in the file that owns it (`irq.6502` for what the
  IRQ shares with the main loop).
- A macro never reaches into its caller's scope: its own `ZA_AUTO`s are declared inside it; anything the caller
  sets or reads is passed in as an argument.
- `ZA_ENTRY` only on roots nothing calls; `ZA_INTERRUPT` on every interrupt or `BRK` handler; `ZA_UNREACHABLE`
  after every always-taken branch, every `JSR` that never returns and every `BRK`'s message; `ZA_WIPE` after a
  wholesale zero-page wipe; `ZA_INDEXEDBY` where the index set is known; `ZA_DISCARD` before an indexed rebuild.
- Recursion keeps nothing in zero page across the recursive call but a running count; what a level wants back
  goes on the stack round the call (`LDA v:PHA` ... `PLA:STA v`).
- **Check the `[auto]` lines in the `-v` listing by hand** whenever they change: Baron's allocator has had bugs.
- Until the loader owns the machine, nothing in the pool may be live across an OS call or an interrupt in the OS's
  bytes (Baron takes an OS call as touching nothing).

## Program structure

| File | What |
|---|---|
| `boot.6502` | `!BOOT`, assembled on its own (OS alive): loading screen, sideways RAM, loads, unpacks. Portal's is inline `BASIC ... ENDBASIC` ending in `CALL`. |
| `<project>.6502` | The top level: includes, then the sections: banks, main RAM's code rephased to where it runs, the loader last. |
| `constants.6502` | Constants and the screen geometry, derived (`InitialTimer = ...`, `R2 = 98-(80-w) DIV 2`). |
| `memory.6502` | A map of memory drawn as a comment, the `*Org` constants, the `ZA_POOL`, and empty virtual sections for workspace. |
| `macros.6502` | Macros for any file; a file's own macros in `<file>_macros.6502` beside it, included first (Portal). |
| `entry.6502` | The loader: shuts the OS down, places the code, `make_tables`, syncs, starts the IRQ. Disposable. |
| `irq.6502` | The interrupt handler and the state it shares, in main RAM. |
| `main.6502` / `start.6502` / `game.6502` | The main loop; feature files after. |
| `zeropage.6502` | Code that runs in zero page (Critical Mass). |

- **`memory.6502` opens with the map as a comment**, one line per region:
  ```
  \\   &0000  zero page: the pool, bar &FC
  \\   &0100  the stack
  \\   &0200  the OS's vectors and workspace: left alone (IRQ1V)
  ```
- **Workspace is claimed where it's used**, by reopening a virtual section from `memory.6502`:
  ```
  SECTION Between
  .FillSpare SKIP 1
  ENDSECTION
  ```
  Pools are grouped by lifetime against the frame (Portal: `Persist`, `Frame`, `Capture`, `Between`, the loader's
  `Scratch`); pools that never coexist share addresses as separate virtual sections over one range (Baron has no
  unions).
- **Tables of any size are made by the loader** (`make_tables`) in a `Tables` section, not assembled in: from
  recurrences (quarter squares), division loops (reciprocals), deltas packed a bit each at assembly time and
  unpacked by a macro. Only a handful of bytes stay as `EQUB`s by their code. Size tables by the design's caps
  (`MaxVerts`), never by the demo data's counts.
- **Struct of arrays**: every field of a record is a table of its own (`BallXLo,X`, `BallXHi,X`).
- **The loader is disposable**: it runs above everything it writes, can be macros, self-modifying and unrolled,
  and its memory becomes workspace afterwards.
- **Generate at assembly time**: unroll with `FOR` (`FOR key = {&49, &00, &42, &61}` unrolls a key scan), derive
  constants in the source, build palettes and tables with lists:
  ```
  EQUB (0..15)*16 + (PhysCols[(0..15) DIV 8*2 + (0..15) DIV 2 MOD 2] EOR 7) ; MODE 5's palette
  ```
- **`ASSERT` every invariant** the code relies on, with a message: alignment, step sizes, branch reach, table
  adjacency, a fit (`ASSERT maincode_image >= MainCodeOrg, "main RAM's code must load above where it runs"`).
- **Place hot code deliberately**: tables read indexed within a page (`FITPAGE`/`INPAGE`, or `ALIGN &100`); hot
  branches followed by `SAME_PAGE target`; code carrying `SAME_PAGE` checks first in its section, before data
  whose size changes; `PADTO`/`PADTOLOW` anchors that `PRINT` the bytes they waste.
- Test builds through `-D TEST=TRUE` and `IF DEFINED(TEST)`.

## Habits that make it read like Rich's

- **Fast from the first line**: every routine written as fast as it can be from the start, not tidied later.
  The specifics are in `optimisation.md` §3.
- **Routines fall through into each other** rather than `JMP` or `JSR`/`RTS` (`.fill { ... ; then on into
  fill_colour }`).
- **Macros for hot inlined paths**; their rare callers share one routine that expands the macro. A macro that
  takes a bound gets the tightest one the value can reach, and an assembly-time `IF` on the parameter picks the
  cheapest expansion.
- **The rare case goes out of line** in a hot loop so the common path falls through and stays on its page.
- **Literal hardware addresses with a trailing comment**, never names.
- **The OS shut down once, the same way each time** (`hardware.md` §2).
- **No** CMOS opcodes, `P%+n` branches, hand-placed zero page, assembled tables the loader could make,
  `table-1,X` bases across a page, or a tidy-up pass in place of fast code now.

## A routine in house style

```
\\ Clear the sprite buffer's eight pages to colour 0.
\\
\\ Inputs:
\\     none
\\
\\ Outputs:
\\     A, X: 0
\\     Y: preserved

.clear_buffer
{
    LDA #0
    TAX                                 ; (A = 0)
.loop
    FOR pg = 0..<8
        STA SpriteBuf+pg*256,X
    NEXT
    INX
    BNE loop
    SAME_PAGE loop
    RTS
}
```
`SAME_PAGE`, `FITPAGE` and `FITPAGE_FROM` are in Portal's `src/macros.6502`; `PADTO`, `PADTOLOW` and `INPAGE` in
Teddy Boy's `src/macros.6502`. Copy them into a new project's `macros.6502`.
