# beeb-baron

A Claude Code skill for writing BBC Micro 6502 code in [Baron](https://github.com/waitingforvsync/baron) the way
Rich Talbot-Watkins does: Baron's syntax, sections and zero-page allocator; the house style and program structure;
6502 optimisation that pays on the Beeb; the hardware and memory map for taking the machine over; and how the code
is checked and measured.

It was written from Rich's projects (Portal, Teddy Boy, Critical Mass, Baron's examples) and what their
optimisation reviews measured.

## Installing

Clone it into your personal skills, or into a project's:

```
git clone https://github.com/waitingforvsync/beeb-baron.git ~/.claude/skills/beeb-baron
git clone https://github.com/waitingforvsync/beeb-baron.git .claude/skills/beeb-baron
```

Claude Code loads it when you work on `.6502` source, Baron builds, BBC boot discs and the like, or when you type
`/beeb-baron`.

## What's in it

| File | Covers |
|---|---|
| `SKILL.md` | The essentials, the model projects, working practices, building |
| `style.md` | Layout, naming, comments, zero page, program structure, a routine in house style |
| `baron.md` | The language, the zero-page allocator and its annotations, traps found in practice, the listing |
| `optimisation.md` | The cost model, techniques that paid, ones measured as losses, a waste checklist |
| `hardware.md` | Memory with the OS alive, the takeover, CRTC, Video ULA, the vertical rupture, VIAs, keyboard, SN76489, sideways RAM, the DFS, the Master 128 |
| `checking.md` | Emulators and harnesses, proving a change exact, measuring and reporting |

## What it expects

- **Baron 0.6 or later.** Before 0.6, an annotation in one of two sections at the same address (two sideways RAM
  banks) applied to both, and zero page was silently misallocated (Baron #29).
- **For checking and timing:** headless jsbeeb (the `jsbeeb-mcp` npm package also serves it to Claude over MCP),
  py65 for logic harnesses, and optionally b2 and beebjit.
- **The model projects, if you have them.** The skill looks for checkouts beside the current project
  (`../teddyboy`) and asks otherwise. Without them, its own files carry what they teach, and Baron's examples are
  public.
