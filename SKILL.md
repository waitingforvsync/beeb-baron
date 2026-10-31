---
name: beeb-baron
description: Write BBC Micro 6502 code in Baron the way Rich Talbot-Watkins does - Baron's syntax, sections and zero-page allocator; his house style and program structure; 6502 optimisation that pays on the Beeb; the hardware (CRTC, Video ULA, VIAs, SN76489, keyboard, sideways RAM) and memory map for taking the machine over; and how his projects check and measure code. Use when writing, reviewing, optimising or debugging .6502 source, Baron builds and listings, BBC boot discs and loaders, interrupts, raster timing or screen layouts.
---

# BBC Micro code in Baron, Rich's way

Rich Talbot-Watkins wrote BeebAsm and Baron, exhale, and decades of BBC code; this is how his projects are
written. Pitch everything at an expert, with exact cycle counts, addresses and register values. The models to
imitate, all in Baron and all under github.com/waitingforvsync:

| Project | Repository | Shows |
|---|---|---|
| Portal | `portalc`: `beeb/`, rules in `codingstandards.md` | The written standard; relocation; a C model checked byte for byte. Targets the Master 128, NMOS opcodes only. |
| Teddy Boy | `teddyboy`: `src/` | A whole game: two sideways RAM banks, an exhale-style vertical rupture, sprites, sound, a packed boot disc, fifteen optimisation reviews |
| Critical Mass | `criticalmass`: `src/` | A packed in-place payload, code in zero page, 25 fps by counting vsyncs |
| Baron's examples | `baron`: `examples/{starglobe,spritescale,camera,colours,compress}` | The language's features as intended (older comment style) |

Look for checkouts beside the current project (`../teddyboy`), or ask the user where they are. Baron is public;
the others may not be, and where one isn't to hand, the reference files here carry what it teaches. Before
writing a new file, read the nearest equivalent and match it. In an existing project, its own files win over
anything here.

## Reference files (read the one you need before you start)

- **`style.md`**: layout, naming, comments, zero-page and program-structure conventions, with a routine in house
  style.
- **`baron.md`**: the language (sections, scopes, lists, functions, macros, transforms), the zero-page allocator
  and its annotations, the traps found so far, building and reading the listing.
- **`optimisation.md`**: the cost model, "fast from the first line", techniques that paid, ones measured as
  losses, and the waste checklist.
- **`hardware.md`**: memory with the OS alive, the takeover sequence, SHEILA and the 1 MHz bus, CRTC, Video ULA,
  the rupture, VIAs, vsync sync, keyboard, SN76489, sideways RAM, the DFS, the Master 128, and the gotchas.
- **`checking.md`**: emulators and harnesses, making checks that can fail, proving a change exact, measuring and
  reporting.

## The essentials

**Baron, not BeebAsm.** `SECTION name, org = ..., guard = ..., filename = "..."` ... `ENDSECTION`; no
`ORG`/`SAVE`/`GUARD`. `TRUE` is 1. Each file on the command line is its own assembly. NMOS only: never
`cmos = TRUE`, no 65C12 opcodes even on a Master. Anything not in `baron.md` is in Baron's `docs/guide.md`
and `docs/reference.md`.

**Structure.** `boot.6502` (`!BOOT`, OS alive: loading screen, find sideways RAM, load, unpack); `<project>.6502`
(includes, then sections: banks at `&8000`, main RAM's code rephased to where it runs, often packed with
`transform = "zx02"`, the loader last); `constants.6502`; `memory.6502` (a map as a comment, the `ZA_POOL`, empty
virtual sections); `macros.6502`; `entry.6502` (the disposable loader: shuts the OS down, places the code, makes the
tables, syncs, starts the IRQ); `irq.6502`. Tables are made by the loader, not assembled in. Workspace is claimed
where it's used by reopening a virtual section.

**Style** (new code; Rich, 2026-10-30): 4 spaces, never tabs; labels at column 0; trailing `;` comments at column 40;
`\\` for whole-line comments, file and routine headers with `Inputs:`/`Outputs:`; every routine braced; `LDA
#0:STA x` with no spaces round colons; CamelCase constants, functions and memory data, lower_snake routines,
short lower-case zero-page variables, UPPER macros; literal hardware addresses named in a comment
(`STA &FE4D ; system VIA: IFR`); `; (always)`, `; (C clear)`, `; self-modified: what`; terse British English.

**Let Baron check you.** `ASSERT` every invariant (alignment, step sizes, branch reach, table adjacency, fits);
`SAME_PAGE` after hot branches; `ZA_AUTO` for every zero-page variable, in the smallest scope; `ZA_CANCALL` /
`ZA_CANJUMP` after every computed or patched `JSR`/`JMP`/branch; `ZA_UNREACHABLE` after every always-taken branch,
non-returning `JSR` and `BRK` message; `ZA_INTERRUPT` on handlers, `ZA_ENTRY` on roots. Read the `[auto]` lines
in the listing when they change. Never name a loop label like a variable (a local label hides a global).

**Fast from the first line.** Registers before zero page, no copies, follow the carry, tables over arithmetic,
indexed tables within a page, hot loops unrolled and entered by patching, the rare case out of line. Then
measure.

**The machine.** Hook IRQ1V (`&204`), end handlers `LDA &FC:RTI`, keep `&FC` out of the pool, `RTI` at `&D00`,
`&258` = 2. The IRQ owns port A and the addressable latch (keys and sound). Timer reloads of `rows × 512 - 2`, so
one `&FE47` write reloads and acknowledges. Never read T1C-L of the timer driving the IRQ. No stack temporaries.
While the OS lives: `STA &F4:STA &FE30`, the ULA's control register through OSBYTE 154, `*FX16,0`.

**Branch out of range**: move code so it reaches; invert round a `JMP` only if nothing else works, and say so.

## Working practices

- **Measure; an estimate is not a result**, the user's ideas included. Time with the real IRQ (headless jsbeeb),
  prove speed-ups exact, and show cycles saved and bytes spent, by memory region, before anything is committed.
  Faster *and* smaller goes in; anything else is the user's call.
- **Commit, push or release only when asked.** Don't chain a commit onto a measurement.
- **Test where the hardware is strict**: jsbeeb and real machines, not only b2; both DFSes for anything timed
  during loading; py65 never runs the IRQ.
- **Decision logs** (`docs/decisions.md` in Teddy Boy and Critical Mass): your recommendations stay Proposed until
  the user decides; record their words verbatim, with the date.
- **Write docs and commit messages plainly**: British spelling, short sentences, every claim with numbers
  (before → after, seeds, frames, bytes by region), no em dashes, mistakes recorded as mistakes.
- Baron problems become issues at `waitingforvsync/baron`, with a minimal repro.

## Building

`make`, or `baron -v --symbols build/x.json -o build/x.ssd --opt 2 src/boot.6502 src/<project>.6502 >
build/x.lst`. `baron --check` assembles without writing. The `-v` listing gives every address, every assignment's
value, each `ZA_AUTO`'s byte (`name = &70 [auto]`) and each pad's waste.
