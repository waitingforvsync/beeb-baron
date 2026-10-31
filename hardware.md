# The BBC Micro: hardware, memory map, and taking the machine over

Model B, OS 1.20, 2 MHz NMOS 6502. Sources: the OS 1.20 disassembly, beebjit's
`bbc.c` (github.com/scarybeasts/beebjit), exhale (github.com/richtw1/exhale), and what Teddy Boy, Portal and
Critical Mass do. Where a value here came from one project, it says so.

## 1. Memory with the OS alive

**Zero page**

| Range | Owner |
|---|---|
| `&00`-`&8F` | The current language. BASIC uses `&00`-`&6F` and leaves **`&70`-`&8F` to the user**: a loader running under BASIC uses only these. `&00`/`&01` are BASIC's LOMEM: write them only once nothing can fail back to BASIC. |
| `&90`-`&9F` | Econet |
| `&A0`-`&A7` | The NMI owner (the DFS) |
| `&A8`-`&AF` | OS temporaries (`*` commands) |
| `&B0`-`&BF` | Filing system scratch |
| `&C0`-`&CF` | The filing system's private workspace |
| `&D0`-`&E1` | VDU driver |
| `&E2`-`&FF` | OS: `&EC`/`&ED` keys pressed, `&EF`-`&F1` OSBYTE/OSWORD A, X, Y, `&F2`/`&F3` the command line, **`&F4` the paged ROM's number (ROMSEL's copy)**, `&FA`/`&FB` temporaries, **`&FC` where the MOS's IRQ entry saves A**, `&FD`/`&FE` the last BRK's address, `&FF` the ESCAPE flag. |

**Pages 1 to `&D`**

| Range | Use |
|---|---|
| `&0100`-`&01FF` | Stack |
| `&0200`-`&0235` | Vectors: `USERV` `&200`, `BRKV` `&202`, **`IRQ1V` `&204`**, `IRQ2V` `&206`, `CLIV` `&208`, **`BYTEV` `&20A`**, `WORDV` `&20C`, `WRCHV` `&20E`, `RDCHV` `&210`, `FILEV` `&212` ... `FSCV` `&21E`, `EVNTV` `&220`, `UPTV`, `NETV`, `VDUV`, `KEYV` `&228`, `INSV`, `REMV`, `CNPV` `&22E`, `IND1V`-`IND3V`. |
| `&0236`-`&02FF` | OS variables (OSBYTE `&A6`-`&FF`): `&240` vsync counter, `&248` **the Video ULA control register's copy** (OSBYTE 154 writes both), `&249` the last palette byte, **`&258` OSBYTE 200's byte** (bit 1: clear memory on BREAK). |
| `&0300`-`&037F` | VDU variables |
| `&0380`-`&03FF` | Cassette filing system, keyboard buffer |
| `&0400`-`&07FF` | The language's workspace (BASIC's integer variables and buffers) |
| `&0800`-`&08FF` | Sound workspace and buffers, printer buffer (`&880`), envelopes (`&8C0`) |
| `&0900`-`&0AFF` | RS423/cassette buffers, speech buffer (`&9C0`) |
| `&0B00`-`&0BFF` | Soft keys |
| `&0C00`-`&0CFF` | User-defined characters 224-255 |
| `&0D00`-`&0D9E` | The NMI routine (the filing system puts its handler here; the hardware NMI vector leads to `&D00`) |
| `&0D9F`-`&0DFF` | Extended vectors and the paged ROMs' workspace table |
| `&0E00`-`&18FF` | DFS workspace and file buffers (PAGE = `&1900` with a DFS, `&0E00` without) |
| up to the screen | User memory. Screen starts: MODEs 0-2 `&3000` (20K), 3 `&4000` (16K), 4-5 `&5800` (10K), 6 `&6000` (8K), 7 `&7C00` (1K). |
| `&8000`-`&BFFF` | The paged ROM or sideways RAM bank selected by ROMSEL |
| `&C000`-`&FBFF` | OS ROM |
| `&FC00`-`&FEFF` | FRED, JIM, SHEILA (I/O, §3) |
| `&FF00`-`&FFFF` | OS entry points (`OSBYTE` `&FFF4`, `OSWORD` `&FFF1`, `OSWRCH` `&FFEE`, `OSCLI` `&FFF7`, `OSFILE` `&FFDD`); `&FFB7` points to the default vector table; hardware vectors (in ROM: you can't change `&FFFE`). |

## 2. Taking the machine over

Rich's programs boot in two stages, as exhale does: a **`!BOOT`** that runs with the OS alive (loading screen,
finding sideways RAM, loading files) and a **loader** at the end of the main file that shuts the OS down,
puts the code where it runs and starts the game. Portal, Teddy Boy and Critical Mass all follow it.

### What survives the takeover
- **The MOS's IRQ entry.** The 6502's IRQ vector is in ROM, so every IRQ goes through
  `STA &FC : PLA : PHA : AND #&10 : BNE brk : JMP (&204)` (19 cycles on NMOS after the 7-cycle interrupt;
  `JMP (abs)` is 6 on CMOS). Hook **IRQ1V** (`&204`) and end every handler `LDA &FC : RTI`. Keep `&FC` out of the
  zero-page pool. The handler keeps X and Y, or saves them in zero page (no pushes).
- **NMIs** still go to `&0D00`: put an `RTI` there once the DFS is gone.
- **`&258` = 2**, so BREAK clears memory and is a hard reset rather than re-entering a half-dead OS.
- **Nothing else.** No OS calls once the loader has overwritten its workspace. A `BRK` is fatal (the MOS's BRK path
  runs through ROM tables that are game code now).
- With all that, zero page `&00`-`&FF` bar `&FC`, all of pages 1 to `&D` bar what the conventions above fix, and
  everything up to `&7FFF` is yours.

### `!BOOT` (OS alive; Teddy's `src/boot.6502`)
1. `CLI`; `*FX15,0` (flush the BREAK beep); `*FX16,0` (**ADC off**: its interrupts made splits fire early or
   up to 5 lines late).
2. **Find sideways RAM** while MODE 7 can still show an error: for X = 15 down to 0, `STX &F4 : STX &FE30`, EOR
   `&8000` with `&FF`, store, compare, put it back. Take the highest banks found. Restore `&F4` and ROMSEL.
   `BRK` with a message if there aren't enough. Write the bank numbers to zero page (`&00`/`&01`) only after.
3. Set the loading screen's CRTC (R8 = `&F0` first, display off), and the ULA's control register **through
   OSBYTE 154**, not `&FE20`: the OS rewrites `&FE20` from its copy at `&248` at vsync when it flashes colours.
4. Unpack the screen; start any palette or mode splits under `SEI` (User VIA T1 free-running into IRQ1V,
   chaining to the old vector). Teddy also moved the OS's System VIA T1 tick to 64 µs after vsync with a
   period of exactly a frame, so the OS's own IRQ never lands on a split.
5. **Load** with OSFILE (A = `&FF`, exec low byte 0, so it loads at the block's address). With no files open,
   the DFS's file buffers below PAGE (`&1100`-`&18FF`) are safe to load over.
6. **Page each bank in with `STA &F4 : STA &FE30`** (that order: an interrupt that pages a ROM in and out restores
   from `&F4`) and unpack into it; restore the original.
7. `*RUN` the next file through OSCLI.

### The loader (Teddy's `src/entry.6502`, after exhale and Portal)
```
.entry
    CLD
    CLI
    LDA &FFB7:STA osbyte                ; restore BYTEV from the MOS's default vector table
    LDA &FFB8:STA osbyte+1
    LDY #10
    LDA (osbyte),Y:STA &20A:INY
    LDA (osbyte),Y:STA &20B
    LDA #19:JSR &FFF4                   ; vsync: the calls below hold interrupts off (22-25 lines)
    LDA #16:LDX #0:JSR &FFF4            ; ADC off
    LDA #143:LDX #12:LDY #255:JSR &FFF4 ; claim NMIs from the filing system
    LDA #140:LDX #0:JSR &FFF4           ; *TAPE: deselect the DFS for safety
    LDA #19:JSR &FFF4                   ; vsync before changing the CRTC
    SEI
    LDX #LO(StackTop):TXS               ; (a small stack frees page 1 for code)
    ... CRTC R13 down to R0, &FE20, all 16 palette entries black ...
    LDA bank:STA &FE30                  ; no OS left to tell: ROMSEL alone
    ... unpack or copy the code to where it runs (over IRQ1V, &258 and &D00, which the image sets) ...
    ... make the tables ...
    LDA #&7F:STA &FE4E:STA &FE6E        ; every VIA interrupt off
    STA &FE4D:STA &FE6D                 ; every flag cleared
    STA &FE43                           ; DDRA: PA0-6 out, PA7 in (keyboard)
    LDA #&C0:STA &FE4E                  ; System VIA T1 only
    LDA #&40:STA &FE4B                  ; T1 free-running
    LDA #&0F:STA &FE42                  ; DDRB: PB0-3 drive the addressable latch
    LDX #&03:STX &FE40                  ; keyboard auto-scan off
    INX:STX &FE4C                       ; PCR 4 (the OS's: CA1 vsync on its falling edge)
    STX &FE40:INX:STX &FE40             ; &04, &05: the 16K screen wrap (§4)
    ... sync to vsync to the cycle, start T1 (§5), CLI, JMP to the game ...
```
- The code image is assembled to cover `&204` with `EQUW irq`, `&258` with `EQUB 2` and `&D00` with `RTI`, so
  unpacking it sets them. The loader comes last in the file, above anything it writes.
- Keep the display off (R8 = `&F0`) and the palette black until the first screen is drawn.
- Critical Mass runs code in zero page (`SECTION ZeroCode, org = ZeroCodeOrg, guard = &FC`): another reason the
  pool stops short of `&FC`.
- **Variants between the projects**:
  - IRQ1V either assembled into the image (`PADTO &204 : EQUW irq`, then `SKIPTO &258 : EQUB 2`: exhale, Teddy,
    Critical Mass) or stored by the loader (`LDA #LO(irq):STA &204:LDA #HI(irq):STA &205`: Portal).
  - Stack: `LDX #&FF:TXS` (Portal), or a 32-byte stack to free page 1 for code (Teddy, Critical Mass).
  - PCR: `&00` (Portal) or 4 (the OS's: exhale, Teddy, Critical Mass); both trigger CA1 on vsync's falling edge.
  - Critical Mass also resets the ACIA (`LDA #3:STA &FE08`), since a `*FX2` left over could have turned its
    interrupts on; and seeds its random numbers from `LDA &FE44` (System VIA T1 low, "as random as anything").
  - Critical Mass fetches the default BYTEV with a self-modified `LDA &FFFF,Y:STA &20A-10,Y`, so as not to touch
    zero page while the OS is live.

## 3. I/O (SHEILA, `&FE00`-`&FEFF`) and the 1 MHz bus

| Address | Device | Bus |
|---|---|---|
| `&FE00`/`&FE01` | 6845 CRTC: register number, then data | 1 MHz |
| `&FE08`/`&FE09` | 6850 ACIA (RS423, cassette) | 1 MHz |
| `&FE10` | Serial ULA | 1 MHz |
| `&FE20` | Video ULA control register | 2 MHz |
| `&FE21` | Video ULA palette | 2 MHz |
| `&FE30` | ROMSEL (write-only) | 2 MHz |
| `&FE40`-`&FE4F` | System VIA | 1 MHz |
| `&FE60`-`&FE6F` | User VIA | 1 MHz |
| `&FE80`-`&FE9F` | FDC: 8271 at `&FE80`, or the Acorn 1770 board (control `&FE80`, 1770 at `&FE84`-`&FE87`) | 2 MHz |
| `&FEA0` | Econet | 2 MHz |
| `&FEC0` | uPD7002 ADC (joystick) | 1 MHz |
| `&FEE0` | Tube | 2 MHz |

A 1 MHz access is stretched by 1 or 2 cycles depending on phase: `STA &FE4D` costs 5 or 6, and cycle counts in
Rich's comments take 6. FRED and JIM are 1 MHz too.

## 4. Video

### CRTC
Write the register number to `&FE00`, then the value to `&FE01` (both 1 MHz).

| Reg | Meaning | MODE 0-2 | MODE 4-5 |
|---|---|---|---|
| R0 | Horizontal total - 1 | 127 | 63 |
| R1 | Horizontal displayed | 80 | 40 |
| R2 | Hsync position | 98 | 49 |
| R3 | Sync widths: vsync << 4 \| hsync | `&28` | `&24` |
| R4 | Vertical total - 1 (rows) | 38 | 38 |
| R5 | Vertical total adjust (lines) | 0 | 0 |
| R6 | Vertical displayed (rows) | 32 | 32 |
| R7 | Vsync position (row) | 34 | 34 |
| R8 | Interlace and skew | The OS: `&01` (interlace sync). Games: `&F0` display and cursor off, `&C0` display on, cursor off (non-interlaced) | |
| R9 | Scanlines a row - 1 | 7 | 7 |
| R10/R11 | Cursor | R10 = 32 hides it | |
| R12/R13 | Start address ÷ 8, high then low | `&3000` → `&06`/`&00` | `&5800` → `&0B`/`&00` |

- 64 µs a line in every mode: R0 = 127 at the 2 MHz character clock (MODEs 0-3), 63 at 1 MHz (4-7).
  312 lines a PAL frame = 19,968 µs = 39,936 cycles.
- **R4-R7 act within the running CRTC cycle; R12/R13 are read only when a cycle starts.** That is what makes a
  vertical rupture work: set the *next* cycle's start address during this one.
- Narrower screens recentre with R2: exhale's formula, `R2 = 98 - (80 - width) / 2` (Teddy: 72 wide, R2 = 94),
  and the vsync row `35 - (32 - rows) / 2`.
- Hardware wrap: the screen's addresses wrap at `&8000` back to the start of a 20K, 16K, 10K or 8K block, set by
  addressable latch bits 4 and 5. The OS writes these pairs to `&FE40` (from its code, not its comments, which
  disagree):

  | Screen | Writes |
  |---|---|
  | 20K, `&3000` (MODEs 0-2) | `&04`, `&0D` |
  | 16K, `&4000` (MODE 3) | `&04`, `&05` |
  | 10K, `&5800` (MODEs 4-5) | `&0C`, `&0D` |
  | 8K, `&6000` (MODE 6) | `&0C`, `&05` |

### Video ULA
- **Control register `&FE20`**: bit 0 flash select, bit 1 teletext, bits 2-3 characters a line (10/20/40/80),
  bit 4 the 2 MHz CRTC clock (MODEs 0-3), bits 5-7 cursor width. MODE 0 `&9C`, 1 `&D8`, 2 `&F4`, 3 `&9C`,
  4 `&88`, 5 `&C4`, 6 `&88`, 7 `&4B`.
- **Palette `&FE21`**: one write per entry, `(logical << 4) | (physical EOR 7)`. Physical 0-7 steady; 8-15
  flash. Write all 16 entries: in MODE 2 each logical colour is one entry; in 4-colour modes logical 0 is entries
  {0,1,4,5}, 1 is {2,3,6,7}, 2 is {8,9,12,13}, 3 is {10,11,14,15}; in 2-colour modes paper is 0-7, ink 8-15.
- **The ULA shifts 1s in** behind a byte's pixels: in a 2-colour mode a blank byte's later pixels read entries 1,
  3 and 7. So a mid-screen mode split writes `&FE20` *before* its palette, or a stretched split shows the wrong
  colours for a line.
- **MODE 2 bytes**: two pixels, the left pixel's colour bits 3-0 at bits 7, 5, 3, 1, the right's at 6, 4, 2, 0:
  ```
  FUNCTION RightPixel(c) = (c AND 1) + (c AND 2) * 2 + (c AND 4) * 4 + (c AND 8) * 8
  FUNCTION Mode2Byte(left, right) = RightPixel(left) * 2 + RightPixel(right)
  ```
  MODEs 1 and 5: four pixels, pixel n's high bit at bit 7-n and low bit at 3-n.
- **Character-cell layout**: each 8-line row is width × 8 bytes, a column's 8 bytes consecutive:
  `addr = base + row * (columns * 8) + column * 8 + line`.
- A free bit in MODE 2: set logical 8-15 to the same physical colours as 0-7 and bit 3 of every pixel becomes a
  mark the display ignores (Teddy's opaque-byte mark, `optimisation.md`).

### The vertical rupture (exhale)
Each 312-line frame is two CRTC cycles: the play area and the rest (status row, vsync). Their lengths sum to 312
whatever the fine scroll, so the display never rolls. System VIA T1 free-runs with a reload per interrupt; each
handler sets the CRTC registers for the cycle that's running and R12/R13 for the next. Fine scroll is R5 (the
adjust lines) and the start row's first lines hidden by R8 until the play area starts. Teddy's numbers are in its
`docs/display.md`.
- **Make every reload `rows × 512 - 2`** (T1 counts at 1 MHz and fires at latch + 2 µs): its low byte is then
  always `&FE`, so a handler writes only T1L-H (`&FE47`), which sets the next period and clears T1's flag in one
  store. `ASSERT` it.
- **Dispatch by a self-modified branch**: `BIT &FE4D : BVS handler` (T1 is IFR bit 6, so `BIT` puts it in V), each
  handler rewriting the branch offset to the next one. The handlers must all lie in the branch's reach (ASSERT it).
- **Display off (R8 = `&F0`) during the cycle that isn't shown**, on again at the top.

## 5. The VIAs

| Offset | Register |
|---|---|
| 0 / 1 | ORB/IRB, ORA/IRA (with handshake) |
| 2 / 3 | DDRB, DDRA |
| 4 / 5 | T1C-L, T1C-H (writing T1C-H loads the counter from the latches and starts T1) |
| 6 / 7 | T1L-L, T1L-H |
| 8 / 9 | T2C-L, T2C-H (writing T2C-H starts T2) |
| A | Shift register |
| B | ACR: `&40` T1 free-running; `&00` T1 one-shot, T2 timed |
| C | PCR |
| D | IFR: bit 7 any, 6 T1, 5 T2, 4 CB1, 3 CB2, 2 shift, 1 CA1, 0 CA2. Write 1s to clear. |
| E | IER: write bit 7 set to enable the bits given, clear to disable them (`&7F` all off) |
| F | ORA/IRA without handshake |

- **System VIA interrupts**: CA1 vsync, CA2 keyboard, CB1 ADC end of conversion, CB2 light pen, T1 the OS's
  100 Hz clock, T2 speech. Once the machine is yours: IER = `&C0`, T1 only.
- **Reading T1C-L clears T1's flag**: never read it while T1 drives the IRQ, or an interrupt is lost. To use T1 as a
  beam clock, read T1C-H (`&FE45`, 256 µs = 4 lines a step) and re-read the handler state around it.
- **A vsync IRQ** (Portal, Critical Mass): IER `&82` (CA1 only). Acknowledge by reading port A (`LDA &FE41`:
  reading ORA clears CA1's flag) or `LDA #2:STA &FE4D`. Portal double-buffers through it: the main loop sets
  `request_swap`, the IRQ writes R12 at the next vsync and clears the flag (`LSR request_swap ; 1, to 0`), and the
  main loop waits `.wait LDA request_swap:BNE wait`. Critical Mass paces 25 fps by counting vsyncs against its
  frames, with a cap on catching up.
- **Raster splits with the OS alive** (Baron's `examples/colours`, Teddy's loading screen): User VIA T1
  free-running into IRQ1V, chaining on to the old vector (`.old JMP &FFFF`, patched); lock the OS's System VIA T1
  to the frame by giving it a period of `312*64-2`.
- **T2 as a stopwatch**: `LDA #&FF : STA &FE68 : STA &FE69`, then read `&FE69`: elapsed ≈ (255 - value) × 256 µs.
- **A free-running counter for randomness**: Teddy's `random` EORs User VIA T1C-L (`&FE64`) into its table. Once
  the loader has synced to vsync, everything from power-on is deterministic, and any change of cycles anywhere
  changes what it reads.

### Syncing to vsync to the cycle (hexwab/tricky, in exhale and Teddy)
```
    LDA #2:STA &FE4D                    ; clear CA1
.waitforvsync
    BIT &FE4D:BEQ waitforvsync          ; within ~10 cycles
IF HI(P%) <> HI(P% + 17)                ; the loop's cycle count needs no branch crossing a page
    JMP syncloop
    ALIGN &100
ENDIF
.syncloop
    STA &FE4D       ; 6
    LDX #209        ; 2
.outerloop
    LDY #37         ; 2
.innerloop
    DEY             ; 2
    BNE innerloop   ; 3/2
    DEX             ; 2
    BNE outerloop   ; 3/2 (39,920)
    BIT &FE4D       ; 6
    BNE syncloop    ; 3: 39,935, a cycle short of a frame, until vsync didn't come
    ASSERT HI(syncloop) == HI(P%), "This loop must execute within the same page"
```
Then load T1 with the time to the first interrupt, counting the latency in a comment (IRQ 7, MOS entry 19,
`BIT &FE4D` 6, `BVS` 3), as `entry.6502`'s `InitialTimer` does. CA1 fires on vsync's falling edge (PCR 4), 2
lines after vsync starts.

## 6. The addressable latch, the keyboard and the sound chip

**The latch** (8 bits behind System VIA PB0-3): write `&FE40` = bit number + 8 × value.

| Write | Effect | Write | Effect |
|---|---|---|---|
| `&00` | Sound chip write enable (active low) | `&08` | Sound chip disabled |
| `&03` | Keyboard auto-scan off (read keys yourself) | `&0B` | Auto-scan on |
| `&04`/`&0C` | Latch bit 4 = 0/1 (screen wrap) | `&05`/`&0D` | Latch bit 5 = 0/1 |
| `&06`/`&0E` | CAPS LOCK LED on/off | `&07`/`&0F` | SHIFT LOCK LED on/off |

**Keyboard**, auto-scan off, DDRA = `&7F`: write the key's internal number to `&FE4F` and read bit 7 back.
```
    LDY key:STY &FE4F:CPY &FE4F         ; C clear if the key is down (PA7 reads back as 1)
```
Internal number = -INKEY - 1: SHIFT `&00`, CTRL `&01`, Q `&10`, A `&41`, Z `&61`, X `&42`, P `&37`, O `&36`,
`:` `&48`, `/` `&68`, RETURN `&49`, SPACE `&62`, ESCAPE `&70`, W `&21`, S `&51`, D `&32`, `,` `&66`,
`.` `&67`; cursor up `&39`, down `&29`, left `&19`,
right `&79`. Scanning 8 keys costs about 180 cycles.

**SN76489**, DDRA = `&FF`:
```
    STA &FE4F                           ; the byte to port A (no handshake)
    LDA #&00:STA &FE40                  ; write enable low
    NOP:NOP:NOP:NOP:NOP                 ; hold at least 8 µs (32 of its 4 MHz clocks)
    LDA #&08:STA &FE40                  ; and high again
```
- Tone: first byte `&80 | channel << 5 | period AND 15` (channels 0-2: `&80`, `&A0`, `&C0`), then `period >> 4`
  (6 bits). Frequency = 4 MHz / (32 × period). Volume: `&90 | channel << 5 | attenuation` (`&F0` channel 3;
  15 is silent). Noise: `&E0 | mode`.
- Write only registers that changed (keep shadows).

**Port A is shared by the keyboard and the sound chip, so one owner**: the IRQ. If the main loop writes the chip
while the IRQ scans keys (or the reverse), one corrupts the other. And with auto-scan off the keyboard drives PA7
whatever DDRA says: Teddy was silent on jsbeeb (and real hardware) because every byte arrived with bit 7 clear,
while b2 let the VIA win. Around chip writes: DDRA = `&FF` and latch `&0B` (auto-scan on, as the OS has it);
afterwards `&03` and DDRA `&7F`. Hand keys to the main loop through a byte the IRQ ORs into, taken under
`SEI`/`CLI`. Hand sound requests over through a small lock-free queue.

## 7. Sideways RAM
- ROMSEL `&FE30` is write-only. With the OS alive, write `&F4` first, then `&FE30`. With the OS gone, `&FE30` alone.
- Find RAM banks by inverting a byte at `&8000` and reading it back (above). Don't assume bank numbers.
- Code runs from a bank at full speed. Only code the IRQ runs, and code that must see both banks, needs main RAM.
- **Teddy's rule**: no code in one bank reads the other's data or calls into it; each routine lives in the bank
  with its data, or in main RAM paging in the bank it reads (`LDA bankdata:STA &FE30` ... `LDA bankcode:STA
  &FE30`). The IRQ never pages. No trampolines.
- Two banks assemble as two sections at `&8000`; Baron keeps their symbols and annotations apart.

## 8. Loading and the DFS
- `--opt 2` (`*RUN !BOOT`) or `--opt 3` (`*EXEC`) on the disc; `!BOOT` is just a section saved with that name.
- The 8271 DFS (B-DFS 1.2) and the Acorn 1770 DFS differ in timing: the 1770's claims NMIs around every file
  (OSBYTE 143), masking IRQs for ~580 µs (9 lines). Test loading screens on both (`title_check.mjs --model
  B1770`).
- OSBYTE 143 and 140 in the loader mask interrupts for 1.4-1.6 ms: run them straight after a vsync (OSBYTE 19).
- Both DFSes write the catalogue entry back into OSFILE's block: set the load address again before a second load.
- A small loader can live in the DFS's buffers (`&1200`), and later files can load over its spent data.

## 9. Gotchas, collected

| What happened | Rule |
|---|---|
| Silent sound on jsbeeb: the keyboard drove PA7 during chip writes. | The IRQ owns port A and the latch; auto-scan on and DDRA `&FF` around chip writes. Test on jsbeeb, not only b2 (b2 is more lenient). |
| A split showed wrong colours for a line. | Write `&FE20` before the palette (the ULA shifts 1s in). |
| The OS put its own `&FE20` value back. | Set it through OSBYTE 154 while the OS lives (it rewrites from `&248` at vsync). |
| The 32-byte stack overflowed into the IRQ's code at `&1FF`; the game died in the MOS's BRK handler. | No stack temporaries; a sentinel under the stack checked every frame; count the IRQ's depth (MOS entry 3 bytes, plus its calls) on top of the main loop's deepest. |
| Splits early or late on the loading screen. | `*FX16,0` (ADC off); margins of 3+ lines for DFS NMIs. |
| A rupture interrupt lost. | Never read T1C-L of the timer driving the IRQ. |
| The vsync sync drifted by a cycle. | Keep its loop in one page (`ALIGN`, `ASSERT`). |
| Bank paged wrongly mid-unpack. | `&F4` before `&FE30` while the OS lives. |
| BASIC's LOMEM trashed on an error path. | Write `&00`/`&01` only after nothing can fail. |
| Garbage on screen while loading. | R8 = `&F0` and a black palette until the first frame is drawn. |
| py65 harnesses never run the IRQ. | Keys, sound and anything timed must be checked in jsbeeb with the real IRQ. |
| A harness poked workspace during the load and corrupted the image. | Wait until the code is in place (IRQ1V points at the game's handler and the unpack is done). |

## 10. The Master 128

Portal targets the Master 128 (b2's "Master 128 (MOS 3.20)") but writes NMOS code only, so it runs the same on a
B with the right memory. What differs:
- The CPU is a 65C12: same NMOS opcodes, but some timings change. `JMP (abs)` is 6 cycles (5 on NMOS) and
  `BIT &FE4D` 5 rather than 6 in Teddy's latency count, so cycle-exact code is counted per machine.
- HAZEL, `&C000`-`&DFFF`, is the filing system's 8K of RAM: free once the OS is gone (Portal's checks leave their
  answers there).
- The 1 MHz map is wider: all of SHEILA except `&FE20`-`&FE27`, `&FE2C`-`&FE3F` and above `&FEA0` (beebjit).
- Sideways RAM is in banks 4-7. Probe for it all the same.

## 11. A worked map: Teddy Boy as built

| Range | Use |
|---|---|
| `&00`, `&01` | The two banks' numbers (written by `!BOOT`) |
| `&02`-`&FB`, `&FD`-`&FF` | `ZA_POOL` (used to about `&C2`); `&FC` the MOS's |
| `&0100` | Stack sentinel `&5A` |
| `&0101`-`&0120` | Stack, 32 bytes (27 the deepest) |
| `&0121`-`&2FFF` | Main RAM code (packed in the file, unpacked by the loader): the IRQ first (handlers in reach of the patched `BVS`), `IRQ1V` at `&204`, variables at `&206`-`&257`, `&258` = 2, hot plotting and drawing placed by anchors, an `RTI` at `&D00`, the 50 Hz border IRQ (keys, sound) from `&D04`, the sound driver and data, sprite set-up |
| `&3000`-`&35FF` | Sound state, per-frame scratch, tables and game variables (virtual sections) |
| `&3600`-`&39FF` | The map buffer, with its first rows repeated after it |
| `&3A00`-`&3D7F` | Tables made by the loader (the mask table on a page) |
| `&3D80`-`&3DBF` | Workspace |
| `&3DC0`-`&3FFF` | The status row (576 bytes, outside the wrap) |
| `&4000`-`&7FFF` | The play area, a 16K hardware wrap (72 × 28 rows of 576 bytes) |
| `&8000`-`&BFFF` code bank | Maps and decoder, game logic (paged in normally) |
| `&8000`-`&BFFF` data bank | Tiles (on a page), sprite frames, the plotter and erasing that read them, frames mirrored for the round |

While loading: `!BOOT` at `&1200`, the banks' packed file over its spent title data from `&147A`, the loading
screen at `&6800`; the game file at `&3000`, its loader above the status row.
