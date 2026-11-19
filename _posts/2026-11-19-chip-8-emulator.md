---
title: "CHIP-8 Emulation: A Practical Guide"
description: Everything needed to build a CHIP-8 emulator, from scratch.
---

## About this guide

This guide explains how to build an emulator for CHIP-8, a small virtual machine from the 1970s, walking through each part of the machine with worked examples.

## Historical background

CHIP-8 was created in the mid-1970s by Joseph Weisbecker for RCA's COSMAC VIP. Over the following decades it was reimplemented for other machines, with small details changing along the way. That is why a handful of instructions have two "correct" behaviors today.

| Year | Event | Relevance |
| --- | --- | --- |
| 1977 | RCA sells the [COSMAC VIP](https://en.wikipedia.org/wiki/COSMAC_VIP), a $275 kit computer with an RCA 1802 CPU, shipped with 20 games written in CHIP-8. | The original behavior, often called "COSMAC VIP behavior", comes from here. |
| 1978 | Weisbecker describes CHIP-8 in BYTE magazine ("An Easy Programming System"); the VIPER newsletter prints the interpreter's machine code. | The earliest documentation, covering only 31 of the 35 instructions. |
| 1979–1981 | Kit computers such as the DREAM 6800 and ETI-660 get their own CHIP-8 versions. | Some used a different keypad layout. |
| 1990 | Andreas Gustafsson writes CHIP-48 for HP-48 graphing calculators. | It changed how a few instructions behave, probably by accident. |
| 1991 | Erik Bryntse builds SUPER-CHIP on top of CHIP-48: a 128×64 mode, scrolling, more instructions. | Hugely popular. Many documents labeled "CHIP-8" actually describe SUPER-CHIP behavior. |
| 2014 | John Earnest creates XO-CHIP with his Octo tool: color, real sound, 64 KB of memory. | Ten yearly Octojam game jams (the last in 2023) produced many modern games, mostly built and tested in Octo. |

Sources: [Wikipedia: CHIP-8](https://en.wikipedia.org/wiki/CHIP-8), [CHIP-8 extensions and compatibility](https://chip-8.github.io/extensions), [Wikipedia: COSMAC VIP](https://en.wikipedia.org/wiki/COSMAC_VIP).

### Consequences for an implementation

When the calculator versions changed a few instructions, later documents copied the new behavior under the CHIP-8 name. As a result, many modern games no longer run correctly on the original interpreter, and some old games break on modern emulators.

These differences are known as **quirks**, and about six of them matter (see [Quirks](#quirks-instructions-with-two-correct-answers)). The practical approach:

- Implement plain CHIP-8 first.
- Where a quirk exists, implement one behavior first and later make it a configurable setting.
- Use the test programs listed under [Testing and debugging](#testing-and-debugging), which report exactly which behavior is implemented.

### Scope

This guide covers the original 35-instruction CHIP-8 with its 64×32 screen. SUPER-CHIP and XO-CHIP are natural next steps (see [Where to go next](#where-to-go-next)).

## The machine at a glance

A CHIP-8 emulator is seven pieces of state plus one loop that reads and changes them. This picture shows how they connect; the following sections cover each piece in turn.

![CHIP-8 emulator architecture: core state, the loop, and the front end]({{ '/assets/img/chip-8-emulator/architecture.svg' | relative_url }})

*CHIP-8 emulator architecture: core state, the loop, and the front end*

The core (top) is the same in every CHIP-8 emulator. The front end (bottom) depends on the host computer and graphics library.

| Part | Size |
| --- | --- |
| Memory | 4096 bytes |
| V0–VF | 16 registers × 1 byte |
| I | 16 bits (12 used) |
| PC | 16 bits (12 used) |
| Stack | 12 to 16 return addresses, or more |
| Delay and sound timers | 1 byte each, counting down at 60 Hz |
| Screen | 64 × 32 pixels, each on or off |
| Font | 16 characters × 5 bytes = 80 bytes |
| Keypad | 16 keys, each down or up |

## Memory

CHIP-8 has 4096 bytes of memory, with addresses from `0x000` to `0xFFF`. Every byte can be read and written, and the program itself lives in this memory starting at `0x200`.

### Representation

An array of 4096 bytes, initialized to zero. Everything the program reads or writes by address goes through it: the code, the font, sprite data, and any values the program stores.

### Memory map

| Addresses | Size | Contents |
| --- | --- | --- |
| `0x000`–`0x1FF` | 512 bytes | On the original COSMAC VIP, the interpreter itself. In an emulator: empty, except for the built-in font (see [The built-in font](#the-built-in-font)). |
| `0x050`–`0x09F` | 80 bytes | The font, by a widespread convention. Any location below `0x200` works. |
| `0x200`–`0xFFF` | 3584 bytes | The loaded program, plus any data it uses. |

On the original machines, the interpreter also used the top of memory for its own variables and for the display. An emulator keeps registers, the stack and the screen in separate variables, so everything from `0x200` up is free for the program. The 512-byte gap at the bottom remains because programs expect to start at `0x200`.

### Loading a ROM

A CHIP-8 program file (usually `.ch8`) is raw bytes: no header, no title, no size field. The first byte of the file is the first byte of the program. Loading copies byte *n* of the file to address `0x200` + *n*.

**Worked example.** The IBM logo test program begins with these bytes:

| File position | Byte | Address |
| --- | --- | --- |
| 0 | `00` | `0x200` |
| 1 | `E0` | `0x201` |
| 2 | `A2` | `0x202` |
| 3 | `2A` | `0x203` |
| 4 | `60` | `0x204` |
| 5 | `0C` | `0x205` |

Each instruction is two consecutive bytes, with the first byte as the high half (big-endian). The first instruction is therefore `00E0` (clear the screen), the second `A22A` (not `2AA2`), the third `600C`, and so on.

Loading requirements:

- The file must be read in binary mode. In text mode, some systems alter bytes such as `0x0A` and `0x0D` (newline characters), corrupting the program.
- The largest program that fits is 4096 − 512 = 3584 bytes. Anything larger is not a plain CHIP-8 program.
- Code and data are mixed in the same file. The bytes that draw the IBM logo sit right after its instructions. The emulator cannot tell them apart and does not need to: it only executes what PC points at.

### Memory is writable

Several instructions write to memory, and any byte can be overwritten, including the program's own instructions. This is legal and some programs rely on it, so the program area must not be write-protected.

### Self-check

- After loading, memory at `0x200` equals the first byte of the file.
- Memory at `0x200` + (file size − 1) equals the last byte of the file.
- Everything after that, up to `0xFFF`, is still zero.

## Registers

Registers are a handful of named slots holding the values the program is currently working on. CHIP-8 has sixteen 1-byte registers named V0 to VF, one address register named I, and the program counter PC.

| Register | Size | Holds | Initial value |
| --- | --- | --- | --- |
| V0 – VE | 1 byte each (0–255) | Anything the program needs: positions, scores, loop counters | 0 |
| VF | 1 byte (0–255) | A **flag**: some instructions overwrite it with 0 or 1 to report a result | 0 |
| I | 16 bits (12 used) | A memory address, usually of a sprite or other data | 0 |
| PC | 16 bits (12 used) | The address of the next instruction to fetch | `0x200` |

### V0 to VF

V0–VF are effectively the program's only 16 variables, and almost every instruction reads or changes one of them. They are numbered in hex, so the sixteenth is VF, not V16.

**The X in an instruction is a register number, not a value.** In `7305`, the 3 selects register V3 and `05` is the number to add. If V3 held 10, it now holds 15. Treating the 3 as the value 3 is the most common decoding mistake.

All V registers hold 0–255, and results outside that range wrap around: 250 + 10 stores 4 (260 − 256), and 5 − 10 stores 251 (−5 + 256). Languages with unbounded integers, such as Python and JavaScript, do not wrap automatically, so values must be masked to 8 bits or stored in an 8-bit unsigned type.

### VF: the flag register

Some instructions write 0 or 1 into VF to report what happened:

- An addition that overflowed (carry).
- A subtraction that did **not** need to borrow.
- The bit shifted out during a shift.
- A sprite that erased a pixel already on screen (collision).

**Worked example.** V1 holds 200 and V2 holds 100. The instruction "add V2 to V1" runs. The true sum is 300, which does not fit in a byte, so V1 becomes 44 (300 − 256) and VF becomes 1 to signal the carry.

Because instructions overwrite VF, well-written programs do not keep data in it. The emulator must still treat VF as an ordinary register when an instruction names it as an input. [The instruction set section](#the-instruction-set-one-family-at-a-time) covers the case where VF is both an input and the flag.

### I: the address register

I holds a memory address. Instructions that draw sprites or read and write memory use the address in I. A typical sequence is "set I to the address of a sprite, then draw".

Addresses are 12 bits (`0x000`–`0xFFF`), so I needs a type at least 16 bits wide. A byte is too small: `0x22A` alone needs 10 bits.

### PC: the program counter

PC holds the address of the next instruction. It starts at `0x200`, where the program was loaded. Each instruction is 2 bytes, so PC normally advances by 2. Jumps, calls and skips change it in other ways.

### Self-check

Right after a ROM is loaded, the state is: PC = `0x200`, I = 0, V0–VF all 0, the stack empty, both timers 0, the screen all off.

## The stack and subroutines

The stack is a list of return addresses. A subroutine call pushes the address to come back to; a return pops it and continues from there.

### Subroutines

A subroutine is a piece of the program that can be run from several places, like a function. Two instructions handle it:

- `2NNN` **calls** the subroutine at NNN: push the current PC, then jump to NNN.
- `00EE` **returns**: pop the most recent address into PC.

### Why a stack

Subroutines can call other subroutines, and the return addresses must come back in reverse order: the last one stored is the first one used. A list that only grows and shrinks at one end does exactly that. This is called **last in, first out** (LIFO).

### Worked example: one call

| Address | Instruction | Meaning |
| --- | --- | --- |
| `0x200` | `2208` | Call the subroutine at `0x208` |
| `0x202` | `6105` | V1 = 5 |
| `0x204` | `1204` | Jump to `0x204` (loop forever) |
| `0x206` | `0000` | (unused) |
| `0x208` | `6007` | V0 = 7 |
| `0x20A` | `00EE` | Return |

PC advances by 2 as soon as an instruction is fetched, before it executes. The stack evolves as follows:

| Step | Fetched at | Instruction | PC after fetch | Effect | Stack after | PC after |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `0x200` | `2208` | `0x202` | Push `0x202`, jump | `[0x202]` | `0x208` |
| 2 | `0x208` | `6007` | `0x20A` | V0 = 7 | `[0x202]` | `0x20A` |
| 3 | `0x20A` | `00EE` | `0x20C` | Pop into PC | `[]` | `0x202` |
| 4 | `0x202` | `6105` | `0x204` | V1 = 5 | `[]` | `0x204` |
| 5 | `0x204` | `1204` | `0x206` | Jump | `[]` | `0x204` |

The address pushed in step 1 is `0x202`, the instruction **after** the call. It is simply the value PC already holds after the fetch. Pushing `0x200` (the call itself) would make the return land on the call again, which would call again, forever.

### Worked example: nested calls

If the subroutine at `0x208` itself called another subroutine from address `0x20A`, the stack would grow to `[0x202, 0x20C]`. The first return pops `0x20C` (back into the first subroutine); the second pops `0x202` (back to the main program).

### Size

The original COSMAC VIP interpreter had room for 12 return addresses. Many emulators allow 16, and allowing more is harmless. The stack lives in its own list, outside the 4096 bytes of memory.

### Error cases

- **Return with an empty stack** (`00EE` with nothing to pop): the program is broken, or the emulator is executing data as code. Stopping with an error message is more useful than a crash.
- **Stack overflow** (too many calls without returns): usually a bug in the call or return logic. Again, stopping with a clear message is the best response.

## Timers

CHIP-8 has two 1-byte timers, the delay timer and the sound timer. While a timer is above zero, it decreases by 1 exactly 60 times per second, independently of instruction speed.

| Timer | Set by the program | Read by the program | Side effect |
| --- | --- | --- | --- |
| Delay timer | Yes (`FX15`) | Yes (`FX07`) | None. Programs use it to wait or to pace themselves. |
| Sound timer | Yes (`FX18`) | No | A beep plays while it is above zero. |

### Worked example: a one-second wait

A program pauses for one second like this:

1. Set the delay timer to 60.
2. Read the delay timer into a register.
3. If that register is not 0 yet, jump back to step 2.

Instructions keep executing at full speed the whole time. Meanwhile, the timer logic decreases the delay timer 60 times per second: 60, 59, 58 … 1, 0. After one second, the read in step 2 returns 0 and the program continues.

The timer does not stop the program. It is a value that counts down on its own, and the program polls it.

### Worked example: a beep

The program sets the sound timer to 30. A tone starts immediately and stays on while the timer counts 30, 29 … 1. When it reaches 0, about half a second later (30 ÷ 60), the tone stops. CHIP-8 does not define the tone: any simple beep works.

On the original COSMAC VIP, a sound timer value of 1 produced no sound. Reproducing that detail is unnecessary.

### The most common timer bug

Timers tick at 60 Hz, but instructions run much faster. At a typical speed of about 700 instructions per second, each 1/60 s covers about 700 ÷ 60 ≈ 11.7 instructions.

Decreasing the timers once **per instruction** makes them run roughly 12 times too fast: a one-second pause lasts a twelfth of a second. Timers and instructions need separate pacing; [the main loop section](#the-main-loop-fetch-decode-execute) shows how both fit together.

### Self-check

- A delay timer set to 120 reaches 0 after 2 seconds of real time, regardless of instruction speed.
- A timer at 0 stays at 0. It never wraps around to 255.

## The display and sprites

The screen is 64 pixels wide and 32 tall, and each pixel is either on or off. Everything a program shows is drawn by one instruction, `DXYN`, which flips pixels using XOR and reports in VF whether anything was erased.

### The screen

- 64 × 32 = 2048 pixels, each on or off. No colors or shades.
- Coordinates start at the **top-left** corner: x = 0 is the left edge and x = 63 the right; y = 0 is the top and y = 31 the bottom.
- The emulator stores these 2048 values in a separate variable, not in memory.

```
 x=0                                                           x=63
 y=0  ┌──────────────────────────────────────────────────────────────┐
      │                                                              │
      │                  64 × 32 "logical" pixels                    │
      │                                                              │
 y=31 └──────────────────────────────────────────────────────────────┘
```

On a modern monitor, 64 × 32 is tiny, so each CHIP-8 pixel is normally drawn as a larger square: at 10 × 10 real pixels, the window is 640 × 320. The logical screen stays 64 × 32; only the presentation is scaled.

### Sprites

A sprite is a small picture stored in memory as bytes. **Each byte is one row, 8 pixels wide.** Each bit is one pixel: 1 means "flip this pixel", 0 means "leave it alone". The most significant bit (bit 7) is the leftmost pixel. A sprite is 1 to 15 rows tall.

Testing one pixel of a row means ANDing the byte with a mask: for column c (0–7, counted from the left), the mask is `0x80` shifted right by c. With `0x90` = `1001 0000`, the AND with `0x80` is non-zero (column 0 on) and the AND with `0x40` is zero (column 1 off).

**Worked example.** Three bytes stored at address `0x300`:

```
0x300:  0xFF = 1111 1111  →  ████████
0x301:  0x81 = 1000 0001  →  █······█
0x302:  0xFF = 1111 1111  →  ████████
```

That is a hollow box, 8 wide and 3 tall.

### How DXYN draws

`DXYN` draws the sprite stored at address **I**, **N** rows tall, with its top-left corner at **x = VX, y = VY**. X and Y name registers; the position is the **value inside** them.

For each row and each bit, the rule is XOR:

| Sprite bit | Screen pixel before | Screen pixel after | Sets VF? |
| --- | --- | --- | --- |
| 0 | off | off | no |
| 0 | on | on | no |
| 1 | off | **on** | no |
| 1 | on | **off** | **yes, VF = 1** |

VF is set to 0 at the start of each draw and becomes 1 if **any** pixel goes from on to off. Programs use this for collision detection.

Drawing changes neither I nor VX nor VY. Pixel positions advance across and down during the draw, but the registers do not.

### Worked example 1: drawing on an empty screen

V0 = 10, V1 = 5, I = `0x300` (the box above). The instruction `D013` runs: draw at (V0, V1), 3 rows tall.

- Row 0 (byte `0xFF`) lands on y = 5, pixels x = 10 to 17: all 8 turn on.
- Row 1 (byte `0x81`) lands on y = 6: only x = 10 and x = 17 turn on.
- Row 2 (byte `0xFF`) lands on y = 7, x = 10 to 17: all 8 turn on.

No pixel went from on to off, so **VF = 0**. Eighteen pixels are now lit.

### Worked example 2: drawing the same sprite again

Running `D013` again with nothing changed flips each of those 18 lit pixels, so they all turn off. The box disappears and **VF = 1**.

This is how CHIP-8 programs move objects: draw the sprite at its old position (erasing it), change the position, then draw it at the new one. It is also why CHIP-8 games flicker; the flicker is expected behavior, not an emulator bug.

### Worked example 3: a partial overlap

The screen is empty except pixel (12, 5), which is on. A one-row sprite with byte `0x30` = `0011 0000` is drawn at (10, 5). Its 1-bits are in columns 2 and 3 of the sprite, so they land on x = 12 and x = 13.

- (12, 5) was on and gets flipped → off. That is an erase, so **VF = 1**.
- (13, 5) was off and gets flipped → on.

Result: only (13, 5) is lit, and VF = 1.

### Worked example 4: positions off the screen

The **starting** position wraps around. If V0 = 70 and V1 = 40, the sprite starts at x = 70 − 64 = 6 and y = 40 − 32 = 8.

A sprite that **runs past** an edge, however, is cut off rather than wrapped. An 8-wide row starting at x = 60 draws pixels 60, 61, 62 and 63; the 4 pixels that would land on 64–67 are skipped and do not reappear on the left. Likewise, a 5-row sprite starting at y = 30 draws rows 30 and 31, and the other 3 rows are skipped.

Cutting off is called **clipping**. Some later interpreters wrap instead; see [Quirks](#quirks-instructions-with-two-correct-answers).

### Presenting the screen

The screen variable changes only when `00E0` (clear) or `DXYN` (draw) runs. It can be copied to the window every frame at 60 Hz, or only after one of those two instructions has run.

## The built-in font

The emulator must provide sprites for the 16 hex digits, 0 to F, stored in memory below `0x200`. Programs draw them like any other sprite, mostly to display scores and numbers.

### Why the emulator provides it

Almost every program needs to display numbers. Rather than each program carrying its own digit graphics, the original interpreter had them built in. Programs expect them to be present, so the emulator writes them into memory before the program starts.

### Font data

Each character is 5 bytes, one per row. This is the font in near-universal use:

| Char | Bytes |  | Char | Bytes |
| --- | --- | --- | --- | --- |
| 0 | `F0 90 90 90 F0` |  | 8 | `F0 90 F0 90 F0` |
| 1 | `20 60 20 20 70` |  | 9 | `F0 90 F0 10 F0` |
| 2 | `F0 10 F0 80 F0` |  | A | `F0 90 F0 90 90` |
| 3 | `F0 10 F0 10 F0` |  | B | `E0 90 E0 90 E0` |
| 4 | `90 90 F0 10 10` |  | C | `F0 80 80 80 F0` |
| 5 | `F0 80 F0 10 F0` |  | D | `E0 90 90 90 E0` |
| 6 | `F0 80 F0 90 F0` |  | E | `F0 80 F0 80 F0` |
| 7 | `F0 10 20 40 40` |  | F | `F0 80 F0 80 80` |

### Decoding a character

Each byte is one row; each 1-bit is a lit pixel.

```
'0':  0xF0 = 1111 0000  →  ████····
      0x90 = 1001 0000  →  █··█····
      0x90 = 1001 0000  →  █··█····
      0x90 = 1001 0000  →  █··█····
      0xF0 = 1111 0000  →  ████····
             ↑↑↑↑ ↑↑↑↑
            glyph  always zero
```

**Exercise:** decode `7` (`F0 10 20 40 40`) before reading on.

**Answer:** `0x10` = `0001 0000` lights column 3, `0x20` = `0010 0000` lights column 2, and `0x40` = `0100 0000` lights column 1. The result is a bar on top with a line going down and to the left:

```
████
···█
··█·
·█··
·█··
```

### Why only 4 of the 8 bits are used

The characters are 4 pixels wide, but **every CHIP-8 sprite row is a full byte** (8 pixels), because that is the only format `DXYN` reads. The glyph occupies the left 4 bits and the right 4 bits are zero. Zero bits leave the screen unchanged, so the character appears 4 wide; this is also why every font byte ends in `0`.

The glyph must sit in the **left** half, because the leftmost bit lands at the requested x position. A byte like `0x0F` would draw 4 pixels to the right of the intended position.

### Location

Anywhere in `0x000`–`0x1FF` works, since that area is otherwise empty and no program is loaded there. The conventional location is `0x050`–`0x09F`, and the sizes match exactly: 16 characters × 5 bytes = 80 bytes, and `0x09F` − `0x050` + 1 = `0x50` = 80. The font is copied in at startup, before the ROM is loaded.

### Finding a character

Programs do not know where the font is stored. They use `FX29`, which means "set I to the address of the character for the digit in VX", and the emulator supplies the right address based on where it stored the font. Only the low nibble of VX counts: if VX holds `0x2B`, the character is B.

**Hint:** every character is the same size, and they are stored in order from 0 to F.

### Worked example: displaying a score

For a score of 7, the program does:

1. Put 7 into V0.
2. `F029`: I = address of character 7.
3. Put the x and y position into V1 and V2.
4. `D125`: draw 5 rows at (V1, V2). A 7 appears.

For a score like 156, the program first splits it into the digits 1, 5 and 6 with `FX33` (see [The instruction set](#the-instruction-set-one-family-at-a-time)), then draws each digit this way, moving x a few pixels to the right each time.

## The keypad

CHIP-8 input is a keypad of 16 keys labeled 0 to F. The emulator keeps a pressed or released state for each key, and three instructions read that state.

### Original layout and the usual mapping

The COSMAC VIP keypad was a 4 × 4 grid. Emulators conventionally map it onto the left side of a QWERTY keyboard, preserving the shape:

| COSMAC VIP keypad |  |  |  |  | QWERTY keyboard |  |  |  |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2 | 3 | C | → | 1 | 2 | 3 | 4 |
| 4 | 5 | 6 | D | → | Q | W | E | R |
| 7 | 8 | 9 | E | → | A | S | D | F |
| A | 0 | B | F | → | Z | X | C | V |

**W** therefore means CHIP-8 key **5**, and **X** means key **0**. The shape matters more than the labels, because games were designed around the physical grid. Many games use 2, 4, 6 and 8 as up, left, right and down, which map to 2, Q, E and S.

If the graphics library exposes scancodes (physical key positions) as well as key names, scancodes keep the mapping correct on AZERTY and other layouts.

### Representation

A list of 16 boolean values, one per CHIP-8 key, updated whenever the window reports a key going down or up. Instructions only read this list.

### The three key instructions

| Instruction | Question it asks | Blocks? |
| --- | --- | --- |
| `EX9E` | Is the key whose number is in VX held down? If so, skip the next instruction. | No |
| `EXA1` | Is that key **not** held down? If so, skip the next instruction. | No |
| `FX0A` | Wait until a key is pressed, then store its number in VX. | **Yes** |

**Worked example.** V4 holds 5 and `E49E` runs. If **W** (CHIP-8 key 5) is held, the next instruction is skipped; otherwise it executes normally.

### FX0A in detail

`FX0A` pauses the program until a key is pressed. Two details matter:

- **The timers keep running while it waits.** Only instruction execution stops; the 60 Hz countdown continues.
- **On the original machine, the key counted only once it was released.** Accepting the key the moment it goes down can pick up a key held from earlier, which makes some games skip screens. The keypad test (see [Testing and debugging](#testing-and-debugging)) checks for exactly this.

[The instruction set section](#the-instruction-set-one-family-at-a-time) describes how to wait without freezing the whole emulator.

## The main loop: fetch, decode, execute

The core of the emulator is a loop that reads two bytes at PC, advances PC by 2, determines which instruction those bytes encode, and executes it. Separately, 60 times per second, it ticks the timers and presents the screen.

### Fetch

1. Read the byte at PC and the byte at PC + 1.
2. Combine them with the first byte as the high half: first × 256 + second.
3. Add 2 to PC **immediately**, before executing the instruction.

**Worked example.** PC = `0x208`. Memory holds `0xD0` at `0x208` and `0x1F` at `0x209`. The instruction is `0xD01F`, and PC is now `0x20A`.

Advancing PC before execution means every instruction can assume PC already points at the next one. A jump simply overwrites PC, a skip adds another 2, and a call pushes PC as it is (see [The stack and subroutines](#the-stack-and-subroutines)). No instruction needs to advance PC itself.

### Decode: the six fields

Every instruction consists of four nibbles (4-bit hex digits). Different instructions use different fields, and each field has a standard name:

| Name | Which part | Size | Typical use |
| --- | --- | --- | --- |
| (kind) | 1st nibble | 4 bits | Which family of instruction |
| X | 2nd nibble | 4 bits | A register number: VX |
| Y | 3rd nibble | 4 bits | A register number: VY |
| N | 4th nibble | 4 bits | A small number (sprite height, sub-type) |
| NN | 2nd byte | 8 bits | A number to load, add or compare |
| NNN | last 3 nibbles | 12 bits | A memory address |

All six fields can be extracted from every instruction up front with masks and shifts, and each instruction then uses only the ones it needs. For `0x8AB4`: AND with `0x0F00` then shift right by 8 gives X = `0xA`; AND with `0x00F0` then shift right by 4 gives Y = `0xB`; AND with `0x000F` gives N = `0x4`; AND with `0x00FF` gives NN = `0xB4`; AND with `0x0FFF` gives NNN = `0xAB4`.

**Worked examples.** Each row is one instruction, split into its fields:

| Instruction | Kind | X | Y | N | NN | NNN | Meaning |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `D01F` | D | 0 | 1 | F | `1F` | `01F` | Draw a 15-row sprite at (V0, V1) |
| `7009` | 7 | 0 | 0 | 9 | `09` | `009` | Add 9 to V0 |
| `A22A` | A | 2 | 2 | A | `2A` | `22A` | Set I to `0x22A` |
| `1228` | 1 | 2 | 2 | 8 | `28` | `228` | Jump to `0x228` |
| `8AB4` | 8 | A | B | 4 | `B4` | `AB4` | VA = VA + VB, carry into VF |
| `F233` | F | 2 | 3 | 3 | `33` | `233` | Split V2 into decimal digits |

For `A22A`, X and Y come out as 2 and 2, but they are meaningless: that instruction only uses NNN. Unused fields are simply ignored.

### Selecting the instruction

The first nibble identifies the family. For most families that is enough; four of them need a second look:

| First nibble | How its instructions are told apart |
| --- | --- |
| 0 | The whole value: `00E0` (clear screen) or `00EE` (return) |
| 1–7, 9–D | The first nibble alone decides |
| 8 | N: 0, 1, 2, 3, 4, 5, 6, 7 or E |
| E | NN: `9E` or `A1` |
| F | NN: `07`, `0A`, `15`, `18`, `1E`, `29`, `33`, `55` or `65` |

When no instruction matches, printing the instruction and its address and then stopping saves hours of debugging. A run of `0000` instructions usually means PC has wandered into empty memory or sprite data because of a bad jump or return.

### Execute

Each instruction does a small amount of work, usually a line or two. The next section describes all of them.

### Timing: two clocks

The emulator runs on two separate rhythms:

- **Instructions** run fairly fast. There is no single correct speed; about **700 instructions per second** suits most games, and the rate should be configurable because some games need more or less.
- **Timers and the screen** run at exactly **60 times per second**.

A common way to combine them is to work in **frames** of 1/60 s. In each frame:

1. Read the keyboard and update the 16 key states.
2. Run a batch of instructions: at 700 per second, about 11 or 12 per frame (700 ÷ 60).
3. Decrease each timer by 1 if it is above 0.
4. Draw the screen to the window.
5. Turn the beep on or off according to the sound timer.
6. Wait for the rest of the 1/60 s.

![One 1/60 s frame: six steps, with the fetch-decode-execute loop inside step 2]({{ '/assets/img/chip-8-emulator/main-loop-frame.svg' | relative_url }})

*One 1/60 s frame: six steps, with the fetch-decode-execute loop inside step 2*

The fetch-decode-execute loop from the start of this section lives inside step 2. Everything else happens exactly once per frame.

**Worked example.** At 600 instructions per second, each frame runs 10 instructions. One second gives 60 frames, 600 instructions and 60 timer ticks, so a delay timer set to 60 reaches 0 after exactly one second.

Many graphics libraries can wait for the monitor's refresh (often 60 Hz), which makes step 6 easy. On a 120 or 144 Hz monitor, the elapsed real time must be measured instead of assuming each refresh is 1/60 s.

## The instruction set, one family at a time

CHIP-8 has 35 instructions, and only six of them are needed to draw a first image: `00E0`, `1NNN`, `6XNN`, `7XNN`, `ANNN` and `DXYN`. This section explains every instruction with a before-and-after example. PC has already advanced by 2 when each instruction runs.

### Flow: jumps, calls and returns

| Instruction | What it does | Example |
| --- | --- | --- |
| `00E0` | Clear the screen: every pixel off. | Any screen → all 2048 pixels off. |
| `1NNN` | Jump: PC = NNN. | `1228` → PC = `0x228`. |
| `2NNN` | Call: push PC onto the stack, then PC = NNN. | At `0x20C`, PC is `0x20E` after fetch. `2300` → stack gets `0x20E`, PC = `0x300`. |
| `00EE` | Return: pop the stack into PC. | Stack top is `0x20E` → PC = `0x20E`, and it's removed from the stack. |
| `BNNN` | Jump to NNN + V0. Used for jump tables. | V0 = 4. `B300` → PC = `0x304`. |
| `0NNN` | On the original machine, run native machine code at NNN. | Not used by modern games. Treat it as unsupported. |

`BNNN` is one of the quirks: later interpreters read it as "jump to XNN + VX" instead. See [Quirks](#quirks-instructions-with-two-correct-answers).

### Skips: how CHIP-8 does "if"

These instructions skip the next instruction (PC + 2) when a condition is true.

| Instruction | Skip the next instruction if… | Example |
| --- | --- | --- |
| `3XNN` | VX equals NN | V2 = `0x10`. `3210` → skips. |
| `4XNN` | VX does **not** equal NN | V2 = `0x10`. `4210` → doesn't skip. |
| `5XY0` | VX equals VY | V1 = 7, V2 = 7. `5120` → skips. |
| `9XY0` | VX does **not** equal VY | V1 = 7, V2 = 7. `9120` → doesn't skip. |

**Worked example: an if-statement.** CHIP-8 has no "if … then", so games combine a skip with a jump:

| Address | Instruction | Meaning |
| --- | --- | --- |
| `0x300` | `3205` | Skip the next instruction if V2 = 5 |
| `0x302` | `1400` | Jump to `0x400` |
| `0x304` | … | Continue here |

If V2 = 5: PC is `0x302` after the fetch, the skip adds 2, and execution continues at `0x304`. The jump never happens. If V2 is anything else, no skip, so the jump to `0x400` runs. In plain words: "if V2 isn't 5, go to `0x400`".

### Setting registers

| Instruction | What it does | Example |
| --- | --- | --- |
| `6XNN` | VX = NN | `6A2F` → VA = `0x2F`. |
| `7XNN` | VX = VX + NN, wrapping past 255. VF is **not** changed. | V0 = `0xFF`. `7001` → V0 = `0x00`, VF unchanged. |
| `ANNN` | I = NNN | `A22A` → I = `0x22A`. |
| `CXNN` | VX = (a random byte) AND NN | Random byte `0xB7`. `C30F` → V3 = `0xB7` AND `0x0F` = `0x07`. |

**Watch out with `CXNN`:** it is not "a random number from 0 to NN". It's a random byte with some bits masked off. With NN = `0x06` (`0000 0110`), the only possible results are 0, 2, 4 and 6, never 1, 3 or 5.

### Arithmetic and logic: the 8XYN family

All of these combine VX and VY and store the result in VX. The last nibble picks the operation. VY is never changed.

| Instruction | What it does | VF afterwards |
| --- | --- | --- |
| `8XY0` | VX = VY | unchanged |
| `8XY1` | VX = VX OR VY | see [Quirks](#quirks-instructions-with-two-correct-answers) |
| `8XY2` | VX = VX AND VY | see [quirks](#quirks-instructions-with-two-correct-answers) |
| `8XY3` | VX = VX XOR VY | see [quirks](#quirks-instructions-with-two-correct-answers) |
| `8XY4` | VX = VX + VY | 1 if the sum went over 255, else 0 |
| `8XY5` | VX = VX − VY | 1 if VX ≥ VY (no borrow), else 0 |
| `8XY7` | VX = VY − VX | 1 if VY ≥ VX (no borrow), else 0 |
| `8XY6` | VX = VX shifted right by 1 | the bit that fell off (old bit 0) |
| `8XYE` | VX = VX shifted left by 1 | the bit that fell off (old bit 7) |

**Worked examples:**

| Before | Instruction | After |
| --- | --- | --- |
| V1 = 20, V2 = 30 | `8124` (add) | V1 = 50, VF = 0 |
| V1 = 200, V2 = 100 | `8124` (add) | V1 = 44 (300 − 256), VF = 1 |
| V1 = 10, V2 = 3 | `8125` (subtract) | V1 = 7, VF = 1 |
| V1 = 3, V2 = 10 | `8125` (subtract) | V1 = 249 (−7 + 256), VF = 0 |
| V1 = 5, V2 = 5 | `8125` (subtract) | V1 = 0, VF = 1 (equal means no borrow) |
| V1 = 3, V2 = 10 | `8127` (reverse subtract) | V1 = 7 (10 − 3), VF = 1 |
| V1 = `0x05` = `0000 0101` | `8106` (shift right) | V1 = `0x02`, VF = 1 |
| V1 = `0x81` = `1000 0001` | `810E` (shift left) | V1 = `0x02`, VF = 1 |

Note the subtraction flag is backwards from what many people expect: **VF = 1 means "no borrow"**.

**The VF trap.** What if VF is one of the inputs? Take VF = 200, V1 = 100, and run `81F4` (V1 = V1 + VF).

- Right: compute the sum (300) and the carry (1) from the original values, store 44 in V1, **then** store the carry in VF. Result: V1 = 44, VF = 1.
- Wrong: set VF = 1 first, then add. V1 becomes 100 + 1 = 101.

The rule for every instruction that sets a flag: work out both results first, write VX, and write VF **last**. Writing VF last also means that if X is F (VF is the destination), the flag wins.

The two shifts are quirks too: on the original machine, they first copied VY into VX, then shifted. See [Quirks](#quirks-instructions-with-two-correct-answers).

### Drawing

| Instruction | What it does |
| --- | --- |
| `DXYN` | Draw an N-row sprite from memory at I, at position (VX, VY), using XOR. VF = 1 if any pixel was erased, else 0. |

[The display and sprites section](#the-display-and-sprites) covers this one in detail, with four worked examples.

### Keys

| Instruction | What it does | Example |
| --- | --- | --- |
| `EX9E` | Skip the next instruction if the key numbered VX is held down. | V4 = 5, key 5 held. `E49E` → skips. |
| `EXA1` | Skip the next instruction if that key is **not** held down. | V4 = 5, key 5 not held. `E4A1` → skips. |
| `FX0A` | Wait for a key press, then VX = that key's number. | Player presses key 7 → VX = 7. |

**Hint for `FX0A`:** waiting in a loop inside the instruction would freeze the whole emulator, including the window and the timers. Instead, if no key has been pressed (and released) yet, PC moves back by 2, so the very same `FX0A` runs again on the next pass of the loop, while everything else keeps working.

### Timers

| Instruction | What it does | Example |
| --- | --- | --- |
| `FX07` | VX = delay timer | Delay timer is 42. `F307` → V3 = 42. |
| `FX15` | Delay timer = VX | V3 = 60. `F315` → delay timer = 60. |
| `FX18` | Sound timer = VX | V3 = 30. `F318` → beep for half a second. |

There's no instruction to read the sound timer.

### Memory and I

| Instruction | What it does | Example |
| --- | --- | --- |
| `FX1E` | I = I + VX | I = `0x300`, V2 = 5. `F21E` → I = `0x305`. |
| `FX29` | I = address of the font character for the low nibble of VX | V0 = 7 → I = where the font's "7" starts. |
| `FX33` | Store the decimal digits of VX at I, I+1, I+2 | V2 = 156. `F233` → memory\[I\] = 1, \[I+1\] = 5, \[I+2\] = 6. |
| `FX55` | Store V0 through VX (inclusive) into memory starting at I | I = `0x400`, V0 = 1, V1 = 2, V2 = 3. `F255` → memory\[`0x400`\] = 1, \[`0x401`\] = 2, \[`0x402`\] = 3. |
| `FX65` | Load V0 through VX (inclusive) from memory starting at I | The reverse of `FX55`. |

More `FX33` examples: V2 = 7 stores 0, 0, 7. V2 = 200 stores 2, 0, 0. These are values 0–9, not the characters "1", "5", "6". A game then uses `FX65` to load them into registers and `FX29` to draw each one.

**`FX55` and `FX65` include VX itself.** `F255` stores three registers (V0, V1, V2), not two. Stopping one short is a classic off-by-one bug.

**Two quirks live here** (see [Quirks](#quirks-instructions-with-two-correct-answers)): whether `FX55`/`FX65` leave I pointing past the last byte afterwards, and whether `FX1E` touches VF when I goes past `0xFFF`. A safe start for `FX1E` is to leave VF alone.

### Quick reference

All 35, in one place:

| Instruction | Short meaning | Changes VF? |
| --- | --- | --- |
| `0NNN` | Machine-code routine (ignore) | no |
| `00E0` | Clear screen | no |
| `00EE` | Return from subroutine | no |
| `1NNN` | Jump to NNN | no |
| `2NNN` | Call subroutine at NNN | no |
| `3XNN` | Skip if VX = NN | no |
| `4XNN` | Skip if VX ≠ NN | no |
| `5XY0` | Skip if VX = VY | no |
| `6XNN` | VX = NN | no |
| `7XNN` | VX += NN (no carry flag) | no |
| `8XY0` | VX = VY | no |
| `8XY1` | VX \|= VY | quirk |
| `8XY2` | VX &= VY | quirk |
| `8XY3` | VX ^= VY | quirk |
| `8XY4` | VX += VY | carry |
| `8XY5` | VX = VX − VY | no-borrow |
| `8XY6` | VX >>= 1 | bit shifted out |
| `8XY7` | VX = VY − VX | no-borrow |
| `8XYE` | VX <<= 1 | bit shifted out |
| `9XY0` | Skip if VX ≠ VY | no |
| `ANNN` | I = NNN | no |
| `BNNN` | Jump to NNN + V0 | no |
| `CXNN` | VX = random AND NN | no |
| `DXYN` | Draw sprite | collision |
| `EX9E` | Skip if key VX held | no |
| `EXA1` | Skip if key VX not held | no |
| `FX07` | VX = delay timer | no |
| `FX0A` | Wait for key, VX = key | no |
| `FX15` | Delay timer = VX | no |
| `FX18` | Sound timer = VX | no |
| `FX1E` | I += VX | quirk (usually no) |
| `FX29` | I = font address for VX | no |
| `FX33` | Decimal digits of VX at I | no |
| `FX55` | Store V0..VX at I | no |
| `FX65` | Load V0..VX from I | no |

## Quirks: instructions with two correct answers

Six behaviors differ between the original COSMAC VIP interpreter and the later calculator interpreters (CHIP-48 and SUPER-CHIP). Neither version is wrong; programs were written for one or the other. The robust solution is to make each one a boolean setting.

| Quirk | Original CHIP-8 (COSMAC VIP) | CHIP-48 / SUPER-CHIP | Worked example |
| --- | --- | --- | --- |
| **VF reset** | `8XY1`, `8XY2`, `8XY3` also set VF to 0 | VF is left alone | VF = 9. `8121` → original: VF = 0; later: VF = 9 |
| **Memory** | `FX55` / `FX65` leave I pointing just past the last byte: I + X + 1 | I is unchanged | I = `0x400`. `F255` → original: I = `0x403`; later: I = `0x400` |
| **Display wait** | `DXYN` waits for the next screen refresh, so at most about 60 sprites per second | No waiting | Ten draws in a row take about 1/6 s on the original; no time on later ones |
| **Clipping** | Sprites running past an edge are cut off | Also cut off (but XO-CHIP / Octo wraps them around) | Row at x = 60, 8 wide → cut: only 60–63 drawn; wrap: 60–63 and 0–3 |
| **Shifting** | `8XY6` / `8XYE` copy VY into VX, then shift VX | Shift VX in place; Y is ignored | V1 = 4, V2 = 3. `8126` → original: V1 = 1 (3 ÷ 2), VF = 1; later: V1 = 2 (4 ÷ 2), VF = 0 |
| **Jumping** | `BNNN` jumps to NNN + V0 | Read as `BXNN`: jump to XNN + VX | V0 = 1, V2 = 8. `B220` → original: `0x221`; later: `0x228` |

Sources: [Timendus' quirks test](https://github.com/Timendus/chip8-test-suite), [CHIP-8 extensions and compatibility](https://chip-8.github.io/extensions).

### Two smaller ones

- **`FX1E` and VF.** The original left VF alone when I + VX went past `0xFFF`. At least one interpreter (on the Amiga) set VF = 1 in that case, and one known game relies on it. The test suite expects VF to be left alone.
- **`FX0A` waits for release.** On the original, the key was accepted only once it was released. The keypad test checks this.

### Choosing defaults

Both behaviors have a large body of programs behind them:

- Many programs from the 1990s onward, and older test ROMs, expect the **later** behavior for `FX55`/`FX65` and for shifting.
- The quirks test, with "CHIP-8" selected, expects the **original** behavior: VF reset, memory, display wait and clipping on; shifting and jumping off.

A practical approach:

1. Start with the simplest behavior for each quirk. Most early test ROMs pass either way.
2. Once everything else works, add a boolean setting for each of the six quirks.
3. Run the quirks test (see [Testing and debugging](#testing-and-debugging)) to confirm each setting behaves as intended.
4. When a specific game misbehaves, look it up in the [CHIP-8 database](https://chip-8.github.io/database), which lists the quirks each known game needs.

**Hint for display wait:** real video timing is unnecessary. One simple approach is to end the current frame's batch of instructions as soon as a `DXYN` has executed, so the next draw happens in the next frame.

## Build order

Building in small steps, and testing each one before moving on, keeps bugs local. Each milestone below ends with a visible check.

1. **Skeleton.** Create the state: memory, V0–VF, I, PC, stack, timers, screen, keys. Load a ROM at `0x200` and the font at `0x050`.
   - Check: with the IBM logo ROM loaded, the six bytes from `0x200` are `00 E0 A2 2A 60 0C`.
2. **Tracer.** Implement fetch and decode. For each instruction, print its address and its six fields (see [The main loop](#the-main-loop-fetch-decode-execute)) without executing anything; just advance PC.
   - Check: the IBM logo prints `00E0`, `A22A`, `600C`, `6108`, `D01F`, `7009` … in that order.
3. **Window.** Open a window and draw the 64 × 32 screen variable as scaled squares. Set a few pixels directly, not through instructions, to test it.
   - Check: the pixels appear where expected, with (0, 0) at the top-left.
4. **Six instructions.** Implement `00E0`, `1NNN`, `6XNN`, `7XNN`, `ANNN` and `DXYN`. Everything else stays "unknown instruction → print and stop".
   - Check: the IBM logo appears. This is the classic first milestone.
5. **First two tests.** Run the test suite's CHIP-8 splash screen and IBM logo ROMs (see [Testing and debugging](#testing-and-debugging)).
   - Check: both match the screenshots in the test suite's README.
6. **Most of the rest.** Calls and returns, the skips, the 8XYN family, `9XY0`, `BNNN`, `CXNN`, `FX1E`, `FX29`, `FX33`, `FX55` and `FX65`.
   - Check: the Corax+ and flags tests show only checkmarks.
7. **Timing.** The 60-frames-per-second loop described in [The main loop](#the-main-loop-fetch-decode-execute), plus the timer instructions.
   - Check: a delay timer set to 120 reaches 0 after 2 real seconds. The quirks test can now run.
8. **Keys.** The keyboard mapping, then `EX9E`, `EXA1` and `FX0A`.
   - Check: the keypad test passes all three parts.
9. **Sound.** A beep while the sound timer is above 0.
   - Check: the beep test plays SOS in Morse code.
10. **Games.** Real games ([Where to go next](#where-to-go-next) lists sources), adjusting the instructions-per-second setting until they play correctly.
11. **Quirk settings.** The six boolean settings described in [Quirks](#quirks-instructions-with-two-correct-answers).
    - Check: the quirks test reports the selected behavior.

## Testing and debugging

Test programs written for emulator developers should come before games, because they point directly at the faulty instruction. The standard set is [Timendus' CHIP-8 test suite](https://github.com/Timendus/chip8-test-suite), and each test can also be run in a browser to see the correct output.

### The test suite, in order

| # | ROM | What it checks | Prerequisites |
| --- | --- | --- | --- |
| 1 | CHIP-8 splash screen | Drawing sprites at simple, aligned positions | `00E0`, `6XNN`, `ANNN`, `DXYN`, `1NNN` |
| 2 | IBM logo | The classic first test | Test 1's instructions plus `7XNN` |
| 3 | Corax+ | Most instructions, one checkmark or cross each | Test 2's instructions plus `3XNN`, `4XNN`, `5XY0` (used to check everything else) |
| 4 | Flags | The 8XYN arithmetic, and VF in every case, including VF as an input | The 8XYN family and `FX1E` |
| 5 | Quirks | Which quirk behaviors are implemented | Timers and the 60 Hz loop |
| 6 | Keypad | `EX9E`, `EXA1`, and whether `FX0A` really blocks and waits for release | Keys and timers |
| 7 | Beep | Sound | A working beep |

The quirks and keypad tests show a platform menu; key 1 selects CHIP-8.

A cross in Corax+ or the flags test names the instruction to fix. The VF rules in [The instruction set](#the-instruction-set-one-family-at-a-time) are the usual culprit.

### Debugging tools worth building early

- **Trace log.** An optional line per instruction with the address, the instruction and the registers. When something breaks, the last lines show what happened.
- **Single-step mode.** A key that executes exactly one instruction and pauses, so register changes can be compared with expectations.
- **Pause and dump.** A key that pauses and prints PC, I, V0–VF, the stack and the timers.
- **Stop on unknown instructions.** Print the instruction and its address, then stop (see [The main loop](#the-main-loop-fetch-decode-execute)).

### Symptom → likely cause

| Symptom | Usual suspects |
| --- | --- |
| Black screen, nothing happens | PC does not start at `0x200`; the window never redraws; `DXYN` is not implemented. |
| Garbage instead of the IBM logo | Bytes combined in the wrong order (see [Memory](#memory)); bits read right to left; X and Y used as numbers instead of register values. |
| Logo mirrored inside each 8-pixel block | Sprite bits read starting from bit 0 instead of bit 7. |
| Execution runs into a wall of `0000` | A bad jump or return: the call pushed the wrong address, or PC is not advanced at fetch. |
| Stuck forever after a call | The call pushed its own address, so the return lands on the call again. |
| Games run far too fast | Timers decreased once per instruction instead of 60 times per second, or too many instructions per frame. |
| Score shows nonsense | `FX33` stores digits in the wrong order, or `FX29` returns the wrong font address. |
| A register holds 300 or −5 | Values not wrapped to 0–255 (see [Registers](#registers)). |
| Collisions never register | `DXYN` does not set VF, or VF is not reset to 0 at the start of each draw. |
| One register is off after loading | `FX55`/`FX65` stop at VX − 1 instead of including VX. |
| The window freezes while a game waits for a key | `FX0A` waits in a loop instead of re-running itself (see [The instruction set](#the-instruction-set-one-family-at-a-time)). |

### Further help

- Running the same ROM in [Octo](https://johnearnest.github.io/Octo/), which has a built-in debugger, allows a step-by-step register comparison.
- The r/EmuDev subreddit and the #chip-8 channel of the Emulation Development Discord are active help forums. A trace log makes questions much easier to answer.

## Language and libraries

Any language works. CHIP-8 is slow by modern standards, so performance is never the constraint; what matters is a way to open a window, draw squares, read keys, play a tone and keep time.

| Language | Common choice for window, keys and sound | Notes |
| --- | --- | --- |
| Python | pygame | Quick to start. Integers never overflow, so 0–255 wrapping must be explicit. |
| JavaScript | Browser `<canvas>` + Web Audio | Runs anywhere and is easy to share. Same wrapping caveat as Python. |
| C or C++ | SDL or raylib | Fixed-size unsigned types (8 and 16 bits) make wrapping automatic. |
| Rust | SDL bindings, macroquad or minifb | The compiler requires explicit overflow handling, which suits this project. |
| C# | MonoGame or Raylib-cs | `byte` and `ushort` match CHIP-8's sizes. |
| Go | Ebitengine | `uint8` and `uint16` match CHIP-8's sizes. |

### Separate core and front end

Keeping the emulator **core** separate from the **front end** pays off:

- The core holds the state, executes one instruction and ticks the timers. It never touches a window, keyboard or speaker.
- The front end opens the window, turns key events into the 16 key states, draws the core's screen, plays the beep and keeps time.

The core can then be tested in isolation (load bytes, run 20 instructions, check the registers), and the front end can be swapped (terminal, browser, mobile) without touching the core.

### Graphics are optional at first

The first two build milestones need no window. Even the IBM logo can be checked by printing the 64 × 32 screen to a terminal, with `#` for on and `.` for off.

## FAQ

**The font characters are 4 pixels wide. Why do their bytes have 8 bits, with the right 4 always zero?** Every CHIP-8 sprite row is one byte (8 pixels), because that is the only format `DXYN` reads. The glyphs occupy the left half and the right half is blank. Zero bits leave the screen unchanged, so the glyphs appear 4 wide. See [The built-in font](#the-built-in-font).

**The screen is 64 × 32, but a character is only 4 × 5. Is that right?** Yes. A sprite is a small picture placed at a position on the 64 × 32 screen. A digit covers a small patch of it, and a two-digit score is two sprites side by side. The window is normally scaled up (see [The display and sprites](#the-display-and-sprites)).

**Is it necessary to emulate the original 1802 processor or count clock cycles?** No. CHIP-8 was an interpreted language, so only the effect of each instruction needs reproducing. About 700 instructions per second is close enough for nearly every game.

**How does a CHIP-8 program end?** Usually it does not. There is no halt instruction, so a finished program jumps to itself forever.

**Why do games flicker?** Games erase sprites by drawing them again with XOR, then redraw them elsewhere. The original machines flickered too. Some emulators fade pixels out gradually to hide it; that is optional.

**Do the registers, stack and screen live in the 4096 bytes of memory?** Not in an emulator; they are kept in separate variables. The original interpreter stored them in memory, but no program depends on that.

**Which documents are reliable?** The Timendus test suite reflects current community knowledge. The older "Cowgod's Chip-8 Technical Reference" is still widely linked, but some details it describes, such as sprites wrapping around the screen edges and shifts ignoring VY, match later interpreters rather than the original. When two sources disagree, the difference is probably a quirk (see [Quirks](#quirks-instructions-with-two-correct-answers)).

**Where can games for testing be found?** The [CHIP-8 Archive](https://johnearnest.github.io/chip8Archive/) is a curated collection released under the CC0 "no rights reserved" license. It can be filtered by platform; games marked plain CHIP-8 are the place to start.

## Where to go next

Once the emulator passes the test suite and runs games, three directions follow naturally: larger CHIP-8 variants, writing CHIP-8 programs, or emulating real hardware.

### Extending to SUPER-CHIP and XO-CHIP

| Variant | Additions | Why |
| --- | --- | --- |
| SUPER-CHIP (1991) | A 128 × 64 high-resolution mode, scrolling, larger 16 × 16 sprites, a few more instructions | Many popular 1990s games need it. The existing core grows rather than being rewritten. |
| XO-CHIP (2014) | A second drawing plane for color, a programmable sound pattern and pitch, up to 64 KB of memory | Most modern homebrew uses it, and it is a good exercise in extending a design cleanly. |

References: [Gulrak's CHIP-8 variant opcode table](https://chip8.gulrak.net/) (with only "CHIP-8" ticked at first), the [variant reference pages on chip8kb](https://chip8kb.gulrak.net/reference/variants/xo-chip/), and the [XO-CHIP specification](https://johnearnest.github.io/Octo/docs/XO-ChipSpecification.html). The test suite also includes a scrolling test for these variants.

### Writing a game

[Octo](https://johnearnest.github.io/Octo/) is a browser-based tool for writing CHIP-8 programs in a high-level assembly language. Writing a small game shows the machine from the other side, and the result runs in the emulator too. The Octojam game jam ran every October for ten years and has ended, but Octo and the [CHIP-8 Archive](https://johnearnest.github.io/chip8Archive/) remain available, with many open games to learn from.

### Real hardware

The usual next projects are the Game Boy and the NES. The core ideas carry over directly: state, a fetch-decode-execute loop, memory maps, timers, a screen buffer, input. What changes is that timing becomes critical: the CPU, graphics and sound chips run side by side, and games depend on their exact pace.

## Sources and further reading

### Sources

- [Guide to making a CHIP-8 emulator](https://tobiasvl.github.io/blog/write-a-chip-8-emulator/): specifications, instruction behavior, quirks, recommended speed.
- [CHIP-8 test suite](https://github.com/Timendus/chip8-test-suite), Timendus: the test ROMs, what each one checks, and the quirks it detects.
- [CHIP-8 Technical Reference](https://github.com/mattmikolay/chip-8/wiki/CHIP%E2%80%908-Technical-Reference), Matthew Mikolay: memory layout, display, timers and sound on the original machine.
- [CHIP-8 extensions and compatibility](https://chip-8.github.io/extensions): history of CHIP-48, SUPER-CHIP and XO-CHIP, and original COSMAC VIP behavior.
- [Wikipedia: CHIP-8](https://en.wikipedia.org/wiki/CHIP-8): history and an overview of the virtual machine.
- [End of Octojam](https://beyondloom.com/blog/octojam.html), John Earnest: the status of Octojam, Octo and the CHIP-8 Archive.

### Further reading

- [IBM logo, annotated disassembly](https://github.com/Timendus/chip8-test-suite/blob/main/src/tests/2-ibm-logo.8o): every instruction of the first test program, explained line by line. Best read after a first attempt, to stay spoiler-free.
- [CHIP-8 database](https://chip-8.github.io/database): the quirks and speed each known game needs.
- [Gulrak's CHIP-8 variant opcode table](https://chip8.gulrak.net/): every instruction across all variants, side by side.
- [Octo](https://johnearnest.github.io/Octo/) and the [CHIP-8 Archive](https://johnearnest.github.io/chip8Archive/): a CHIP-8 development tool with a debugger, and a library of CC0 games.
- [Awesome CHIP-8](https://chip-8.github.io/links/): a community list of guides, tools and games.
