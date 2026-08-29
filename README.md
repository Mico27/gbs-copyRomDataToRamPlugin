# gbs-copyRomDataToRamPlugin

**Version 4.3.1. Requires GB Studio 4.3.0 or newer.**

Lets a script read data out of your game's ROM into variables, and builds lookup tables of tilesets
or scenes that a script can index into.

The usual reason to want this: an event asks for "a tileset bank and pointer" or "a scene bank and
pointer", and you want a script to choose which one. Build a list with **Compile tileset array**,
then read entry number 5 out of it with **Copy ROM data to variable** and hand the result to the
event.

- **Copy ROM data to variable** reads any number of bytes from a named block of data, at an offset
  you give, into variables.
- **Compile tileset array** builds a list of tilesets you can index into.
- **Compile scene array** builds a list of scenes you can index into.

> **This one is for advanced users.** A wrong name, offset or length reads whatever happens to sit
> nearby in the ROM, with no warning.

<img width="524" height="210" alt="image" src="https://github.com/user-attachments/assets/efb3f77d-75e5-4fbf-991b-875f8b83d6e2" />

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Size Limits and Restrictions](#size-limits-and-restrictions)
4. [Events Reference](#events-reference)
5. [FAQ](#faq)
6. [Memory Footprint](#memory-footprint)
7. [Bank 0 (HOME) Usage](#bank-0-home-usage)
8. [Changelog](#changelog)

---

## Concepts

### Banks and pointers

The Game Boy sees only one 16 KB slice of the ROM at a time. To reach anything, you need the bank
number and the address inside it. That pair is what GB Studio events mean when they ask for a bank
and a pointer.

The pair takes **3 bytes**: one for the bank and two for the address. That is why reading entry
number *i* out of a list means reading 3 bytes at offset `i × 3`.

### A variable holds 2 bytes

GB Studio variables sit next to each other, two bytes each. **Copy ROM data to variable** writes
bytes into them starting at the one you pick, so a read of more than 2 bytes spills into the next
variable.

Reading a 3 byte bank and pointer therefore fills your chosen variable and the low half of the next
one, which is exactly the pair of values the receiving event wants.

### The array events run at build time

**Compile tileset array** and **Compile scene array** do their work when the project is built, not
while the game runs. They create a named list in the ROM. Scripts read from it later with **Copy
ROM data to variable**, using the same name.

<img width="589" height="497" alt="image" src="https://github.com/user-attachments/assets/7620f64a-3991-45ef-a357-3f85b32ccf7d" />

---

## Project Setup

1. Copy the plugin folder into your project's `plugins` folder. There is nothing to configure.

### Looking up a tileset by number

1. Add **Compile tileset array** to any script. A scene's init script works, since it runs during
   the build. It only has to exist once.
2. Set **Custom data symbol** to a unique name, such as `my_tileset_list`.
3. Set **Tileset count** and pick each tileset in order.
4. Add **Copy ROM data to variable** where the lookup should happen:
   - **Custom data symbol:** `my_tileset_list`
   - **Custom data offset:** `index × 3`, counting from 0
   - **Variable:** the variable that receives the bank. The pointer fills the next one.
   - **Variable offset:** `0`
   - **Data length:** `3`
5. Pass the two variables to **Replace Tileset Tiles Ex**, or any other event that takes a bank and
   a pointer.

<img width="550" height="854" alt="image" src="https://github.com/user-attachments/assets/5d7c013a-740d-4fb3-a57e-817df2bbe0e0" />

### Looking up a scene by number

The same steps with **Compile scene array**, passing the result to the SubmappingEx events or a
scene change.

<img width="587" height="399" alt="image" src="https://github.com/user-attachments/assets/48a0446b-6004-4b72-a1a9-cd1948b87431" />

### Reading data you wrote yourself

Put a C file under `assets/engine/src/` declaring your data:

```c
#pragma bank 255
#include "bankdata.h"
BANKREF(my_data)
const uint8_t my_data[] = { 10, 20, 30, 40 };
```

Then use **Copy ROM data to variable** with **Custom data symbol** set to `my_data`, the offset of
the byte you want, and how many bytes to read.

<img width="551" height="1102" alt="image" src="https://github.com/user-attachments/assets/c8514a7b-6a9a-4bad-877c-5898724cc79b" />

---

## Size Limits and Restrictions

### Length counts bytes, not variables

Each variable holds 2 bytes. A length of 1 fills half a variable, 2 fills one, 3 fills one and a
half, and so on. Leave enough consecutive variables free.

### A bank and pointer is 3 bytes

To read one out of a list, use offset `index × 3` and length `3`.

### Variable offset

**Variable offset** shifts the destination along by whole variables, so you can write into the
middle of a buffer. Use 0 unless you need that.

### The name must exist in the build

A **Custom data symbol** that matches nothing in the built ROM stops the build with an error. The
array events create their name for you. Data you write yourself has to be in place before you
build.

### The array events run only at build time

They add nothing to your game's running code. Changing a list means rebuilding, and each event has
to appear in at least one script for the build to see it.

### Nothing is range checked

Exactly the number of bytes you ask for is copied. Reading past the end of a block, or using the
wrong offset, quietly picks up whatever is next to it in the ROM.

### No engine files are replaced

The plugin adds a new engine file and changes none of the existing ones, so it has no conflicts
with other engine plugins.

---

## Events Reference

### Copy ROM data to variable

Groups: **Variables** and **Misc**.

Copies bytes from a named block of ROM data into consecutive variables. Every field except the name
accepts values, variables and expressions.

| Field | Default | Description |
|---|---|---|
| Custom data symbol | none | Name of the data to read, such as `my_tileset_list`. |
| Custom data offset | 0 | Byte offset to start reading from. For a bank and pointer list, use `index × 3`. |
| Variable | none | The first variable to write into. It receives the first byte read. |
| Variable offset | 0 | Shifts the destination along by whole variables. Use 0 unless you need it. |
| Data length (byte) | 0 | How many bytes to copy. 0 copies nothing, 1 or 2 fill one variable, 3 or 4 fill two, and so on. |

### Compile tileset array

Group: **Scene**, under **Tiles**.

Builds a list of tilesets in the ROM. Runs at build time and adds no code to your game.

| Field | Default | Description |
|---|---|---|
| Custom data symbol | none | Name for the list, such as `my_tileset_list`. Letters, digits and underscores. |
| Tileset count | 1 | How many tilesets the list holds, from 1 to 4096. |
| Tileset 1 to N | Last tileset | Each entry in order. The first is number 0. |

Read an entry with **Copy ROM data to variable** using offset `index × 3` and length `3`. The event
only has to appear once. Repeating it with the same name builds the same list.

### Compile scene array

Group: **Scene**, under **Tiles**.

Builds a list of scenes in the ROM, for looking a scene up by number while the game runs. Useful
for a scene change whose destination is decided by a script, or for submapping.

| Field | Default | Description |
|---|---|---|
| Custom data symbol | none | Name for the list. Letters, digits and underscores. |
| Scene count | 1 | How many scenes the list holds, from 1 to 4096. |
| Scene 1 to N | Last scene | Each entry in order. |

---

## FAQ

**An event wants a "tileset bank and pointer". Where do I get one?**
Build a list with **Compile tileset array**, then read the entry you want with **Copy ROM data to
variable** at offset `index × 3`, length 3. The two variables you get are the bank and the pointer.

**How do I pick a scene to jump to from a variable?**
Build a list with **Compile scene array**, read the entry the same way, and pass the pair to a
scene change or to the SubmappingEx events.

**What offset do I use for entry number 5?**
15, because each entry takes 3 bytes.

**How many variables does a read use?**
One per 2 bytes, rounded up. A 3 byte read touches two variables, so leave the one after your
destination free.

**My build failed with an error about a symbol.**
The name in **Custom data symbol** does not exist in the build. Check the spelling against the
array event that creates it, and make sure that event is in a script that gets compiled.

**I got numbers back but they are nonsense.**
The offset or the length is wrong, and the read picked up neighbouring data. Nothing is range
checked. Recheck that the offset is a multiple of 3 for a list, and that the entry number is inside
the list.

**Where do I put the Compile array events?**
Any script that gets compiled, including a scene the player never visits. They run during the build
and add nothing to your game.

**Do I need to know C to use this?**
Not for the tileset and scene lists. Reading data of your own means writing a small C file, and the
example above is most of it.

**Can I change a list while the game runs?**
No. The lists are built into the ROM. Change the event and rebuild.

**Does it clash with other plugins?**
No. It adds a new engine file and replaces none of the stock ones.

---

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine at default engine settings, report of
2026-08-13. Figures are the difference against a stock project. Each event you use also compiles a
few bytes of script into your project, on top of the fixed cost below.

| Budget | Cost |
|---|---|
| Bank 0 (HOME) | 0 bytes |
| WRAM | 0 bytes |
| Banked ROM | +218 bytes |

- **Bank 0:** nothing. Everything the plugin adds is compiled into a switchable ROM bank.
- **WRAM:** no fixed cost. The copy lands in variables you already have.
- **Banked ROM:** the 218 bytes are the plugin's code. Each list you build adds 3 bytes per entry
  on top.
- **Engine WRAM headroom:** a stock GB Studio 4.3.0 project leaves about **854 bytes** of WRAM
  free (the engine has 7,776 bytes to work with and uses 6,922 of them). With this plugin
  installed roughly **854 bytes** remain. Adding more global variables to your project does not
  change that figure, because script memory is a fixed 3,584 byte block at stock engine settings.
- **SRAM:** not used.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB fixed ROM bank shared by the GB Studio engine core, the
interrupt handlers and the GBDK runtime. Extra banked ROM is cheap to add,
bank 0 is not, so bank 0 is usually the first thing a project runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **0** |

**This plugin costs nothing in bank 0.** Everything it adds is compiled into a
switchable ROM bank.
<!-- BANK0:END -->

## Changelog

Grouped by the date each change was merged into the official
[gb-studio-plugins](https://github.com/gb-studio-dev/gb-studio-plugins) repository.

Only bug fixes, new features and feature changes are listed. Engine version bumps, patch
regeneration, packaging fixes and documentation edits are omitted.

### 2026-06-14

- Added custom script parameter and stack support to the events.

### 2026-02-03

- Added the tileset and scene array events.

### 2026-01-19

- Initial release.
