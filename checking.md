# Checking and measuring

How Rich's projects prove code right and measure it. Portal's rules are its `codingstandards.md` "Checking"
section; Teddy Boy's tools are in its `CLAUDE.md`.

## Emulators

| Tool | Use | Watch out |
|---|---|---|
| **b2** (github.com/tom-seddon/b2) | Running by hand; scripted through its HTTP API (port 48075: reset into a config, mount a disc, boot with SHIFT). Configs "B/Acorn 1770" (Model B with sideways RAM), "Master 128 (MOS 3.20)". | One instance only: two harnesses at once reset each other. More lenient than the hardware (it let the VIA win over the keyboard on PA7). Don't time by polling it from the host clock. |
| **jsbeeb, headless** (node, from the `jsbeeb-mcp` npm package, github.com/mattgodbolt/jsbeeb-mcp, which also serves it to an agent over MCP) | Deterministic runs with the real IRQ and VIA timers; real keys; capturing the sound chip's writes; frame-by-frame screen checks; 8271 and 1770 DFSes (`--model B1770`). The timing reference. | Each instance is 80-200 MB: size a parallel run to the memory free (one timing set of about 42 emulators at a time fitted 12 GB), and leave room on a machine others share. |
| **beebjit** | Fast, accurate; its `bbc.c` is a good reference for bus timing. | |
| **py65** (Python) | Logic harnesses: boot the game, poke state, run to an address. Teddy's `tools/machine.py` adds the sideways RAM banks. | Never runs the IRQ: keys, sound and anything timed must be checked in jsbeeb. Charge the IRQ's cycles if timing in it. Only ever slice its memory (`bytes(mem[a:b])`): copying it whole never ends and ran the machine out of memory. Stop at a bank address only with that bank paged. |

## Making checks (Portal's rules)

- **The 6502 matches a model byte for byte.** Portal's C models the BBC build (same types, same tables, built the
  same way) and every engine part has a check against it; Teddy Boy's Python reads the original ROM and models
  the screen.
- **Run a routine or macro over every input** where that's feasible; sum the answers where they won't fit (Fletcher's,
  mod 255: mod 256 can't see an error in a byte that's the same in every frame of a row of 256).
- **Each check is its own program** with only the files it needs (`-D TEST=TRUE`, `IF DEFINED(TEST)`), leaving its
  answers in memory for the host to compare.
- **Break each check on purpose** to see it fail: each path, and each way a macro expands. Break a fast path laxly
  (letting through what it should catch), since making it stricter only hands work to the exact path. Restore and
  rebuild before measuring anything afterwards: a build left from a break measures the break.
- **Made-up cases mustn't repeat with the cases' structure**: take the systematic parts from the case's number and
  the rest from an LFSR stepped a byte (eight steps) per value, the same in the model and the 6502.
- **Count the 6502's paths** on a cycle-exact model: every instruction run, every branch taken both ways. Code
  unrolled by a macro or `FOR` can leave one copy unreached. A branch that never falls through is an `(always)` or
  a test that can go.
- **Paths the demo never takes** get a stress scene, and caps get checks with the cap lowered.

## Proving a change exact (Teddy Boy)

- A speed-up or a move must make the same decisions: hash the screen and the state after each part of a frame,
  with a clock that doesn't depend on speed (`tools/diffpick.py BASE CAND`), and every set-up
  (`tools/setuphash.py`).
- **Anything that reads a free-running timer** (`random` reading User VIA T1) diverges with any cycle change: hook
  the timer read to a function of frame and read count before comparing runs.
- Brute-force rewritten arithmetic over every input.
- After anything that changes what's drawn, check every frame against the model with the real IRQ
  (`jsbeeb_rate.mjs --model`); after anything touching the IRQ, keys or sound, `make jsbeeb`; after the loader or
  loading screen, `title_check.mjs` on both DFSes.
- Watch the stack: a sentinel under it, checked every frame (`--stack`). "No frames" from a timing run is a crash
  until shown otherwise.

## Measuring

- Time in jsbeeb with the real IRQ, several seeds, the busy rounds; report the distribution (median, 90%, 99%)
  and the share over budget, not just the mean.
- Report **before → after** with the tool, seeds, rounds and frame counts, and **bytes by memory region** with what's
  free after. Thousands separators (4,494), units (cycles a frame, ms).
- Show the results and wait: don't chain a commit onto a measurement.

## Harness habits

- Harnesses that poke workspace must wait until the code is in place (IRQ1V points at the game's handler and the
  unpack is done): during the load those addresses are inside the packed image.
- Sync comparisons on the game's own frames (breakpoints at the frame loop), not emulator frame counts: builds
  load at different speeds.
- Parallel workers write to files, not pipes (a full 64K pipe deadlocked them for hours). Wait on an output file's
  last line, not `pgrep -f` (it matches the waiting shell itself).
- Keep scratch work somewhere that survives a reboot (a `<project>-work/` directory beside the project), not
  only `/tmp`.
- A tool that patches the trees it's given (Teddy's `screens.py --nodefer`) must be given copies.
