# gbs-copyRomDataToRamPlugin

**Version 4.3.1 — Requires GB Studio ≥ 4.3.0**

A GB Studio engine plugin that provides low-level access to arbitrary ROM data from scripts. It adds three events:

- **Copy ROM data to variable** — reads any number of bytes from a named ROM symbol, at a given offset, into script variables.
- **Compile tileset array** — generates a ROM array of tileset far pointers, addressable by index from scripts.
- **Compile scene array** — generates a ROM array of scene far pointers, addressable by index from scripts.

The two array events are build-time generators: they produce named ROM arrays. The copy event then reads individual entries from those arrays at runtime by index.

> **Warning:** This plugin requires an understanding of how ROM data and far pointers work in GB Studio. Incorrect symbol names, offsets or lengths produce silent garbage reads or memory corruption.

<img width="524" height="210" alt="image" src="https://github.com/user-attachments/assets/efb3f77d-75e5-4fbf-991b-875f8b83d6e2" />

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Size Limits and Restrictions](#size-limits-and-restrictions)
4. [Events Reference](#events-reference)
5. [Memory Footprint](#memory-footprint)

---

## Concepts

### ROM banks and far pointers

The Game Boy uses a banked ROM architecture — only one 16 KB bank is visible at a time. To reach data in an arbitrary bank you need both the bank number and the address within it. Together those two values are a **far pointer**, and they are what most GB Studio events that take "a bank and a pointer" expect.

A far pointer occupies **3 bytes**: one bank byte followed by a 2-byte address. That is why reading entry *i* of a far-pointer array means reading 3 bytes at offset `i × 3`.

### Script variables are 2 bytes each

GB Studio stores script variables as consecutive 16-bit slots. **Copy ROM data to variable** copies raw bytes into those slots starting at the variable you pick, so a read longer than 2 bytes spills into the next variable, and so on.

Reading a far pointer (3 bytes) therefore fills the first variable with the bank byte plus the low half of the address, and the low byte of the next variable with the rest — which is exactly the pair of values a far-pointer-taking event expects.

### Build-time array generation

**Compile tileset array** and **Compile scene array** run when the project is built, not at runtime. They generate a named ROM array of far pointers. Scripts then read from that array at runtime with **Copy ROM data to variable**, using the same symbol name.

<img width="589" height="497" alt="image" src="https://github.com/user-attachments/assets/7620f64a-3991-45ef-a357-3f85b32ccf7d" />

---

## Project Setup

1. Copy the plugin folder into your GB Studio project's `plugins/` directory. No additional configuration, engine fields, or compatibility variants are required.

### Reading a tileset far pointer by index

1. Add a **Compile tileset array** event to any script — a scene's init script works, since the event runs at build time. It only needs to exist in one script.
2. Set **Custom data symbol** to a unique name, e.g. `my_tileset_list`.
3. Set **Tileset count** and select each tileset in order.
4. Add a **Copy ROM data to variable** event where you want to read a far pointer at runtime:
   - **Custom data symbol:** `my_tileset_list`
   - **Custom data offset:** `index × 3`, where `index` is the 0-based tileset to select
   - **Variable:** the variable to receive the bank byte; the pointer fills the next variable automatically
   - **Variable offset:** `0`
   - **Data length:** `3`
5. Pass the two resulting variables to **Replace Tileset Tiles Ex**, or any other event that accepts a far pointer.

<img width="550" height="854" alt="image" src="https://github.com/user-attachments/assets/5d7c013a-740d-4fb3-a57e-817df2bbe0e0" />

### Reading a scene far pointer by index

The same workflow, using **Compile scene array** and passing the far pointer to the SubmappingExPlugin events or a scene-change instruction.

<img width="587" height="399" alt="image" src="https://github.com/user-attachments/assets/48a0446b-6004-4b72-a1a9-cd1948b87431" />

### Reading your own ROM data

Create a C file under `assets/engine/src/` declaring your data as a `const` array, with `#pragma bank 255` to let the linker place it and `BANKREF` so its bank symbol is exported:

```c
#pragma bank 255
#include "bankdata.h"
BANKREF(my_data)
const uint8_t my_data[] = { 10, 20, 30, 40 };
```

Then use **Copy ROM data to variable** with **Custom data symbol** = `my_data`, the offset of the byte you want, and the number of bytes to read.

<img width="551" height="1102" alt="image" src="https://github.com/user-attachments/assets/c8514a7b-6a9a-4bad-877c-5898724cc79b" />

---

## Size Limits and Restrictions

### Data length is in bytes, not variables

Each GB Studio variable is 2 bytes wide. A length of 1 reads one byte into the low half of the destination variable; 2 fills one whole variable; 3 fills one variable and the low byte of the next; and so on. Plan your variable layout accordingly.

### A far pointer is 3 bytes

To read one far pointer from an array, use offset `index × 3` and length `3`.

### Variable offset

**Variable offset** adds an extra slot offset to the destination, so you can write into the middle of a multi-variable buffer without declaring more locals. Use 0 for most cases.

### The symbol must exist in the build

If **Custom data symbol** doesn't resolve to a real symbol in the compiled ROM, the build fails with a link error. For the array events the symbol is generated for you; for hand-written data the file and its `BANKREF` declaration must be present before building.

### The array events are build-time only

They produce no runtime code. Changing an array's contents requires a full rebuild, and each event must appear in at least one script for the build to process it.

### No bounds checking

Exactly as many bytes as you request are copied. Reading past the end of a symbol, or misaligning the offset, silently reads adjacent ROM data. Verify your offsets and lengths.

### No engine files modified

The plugin only adds a new engine source file, so it has no compatibility conflicts with other engine plugins.

---

## Events Reference

---

### Copy ROM data to variable

**`EVENT_COPY_ROM_DATA_TO_RAM`** — groups: **Variables**, **Misc**

Copies a sequence of bytes from a named ROM symbol, at a given byte offset, into consecutive script variable slots. All parameters except the symbol name accept values, variables or expressions.

| Field | Default | Description |
|---|---|---|
| Custom data symbol | — | The symbol name of the ROM data to read from, e.g. `my_tileset_list`. |
| Custom data offset | 0 | Byte offset into the ROM data at which to begin reading. For far-pointer arrays use `index × 3`. |
| Variable | — | The first script variable to write into; it receives the first byte of the read. |
| Variable offset | 0 | Additional variable-slot offset applied to the destination. Use 0 for most cases. |
| Data length (byte) | 0 | Number of bytes to copy. 0 copies nothing; 1–2 fills one variable; 3–4 fills two; and so on. |

---

### Compile tileset array

**`EVENT_COMPILE_TILESET_ARRAY`** — group: **Scene → Tiles**

Generates a ROM array of tileset far pointers. Runs at build time and produces no runtime code.

| Field | Default | Description |
|---|---|---|
| Custom data symbol | — | Name of the generated symbol, e.g. `my_tileset_list`. Must be a valid C identifier. |
| Tileset count | 1 | Number of tilesets in the array (1–4096). |
| Tileset 1 … N | Last tileset | Each tileset entry, in order. Index 0 is the first entry. |

Read an entry with **Copy ROM data to variable** using offset `index × 3` and length `3`. The event only needs to appear once in any script; duplicates with the same symbol name regenerate the same array.

---

### Compile scene array

**`EVENT_COMPILE_SCENE_ARRAY`** — group: **Scene → Tiles**

Generates a ROM array of scene far pointers, for looking scenes up by index at runtime — for dynamic scene changes or submapping.

| Field | Default | Description |
|---|---|---|
| Custom data symbol | — | Name of the generated symbol. Must be a valid C identifier. |
| Scene count | 1 | Number of scenes in the array (1–4096). |
| Scene 1 … N | Last scene | Each scene entry, in order. |

---

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine (per-file SDCC compile with GB Studio's build flags, default engine settings). Values are the plugin's *delta* versus the stock engine; DMG build, with CGB noted where it differs. ROM cost lands in banked ROM (GB Studio's autobanker spreads it across switchable banks); using the plugin's events additionally compiles a few bytes of GBVM script per call into your project's script banks.

| | Cost |
|---|---|
| WRAM | +0 bytes |
| ROM | +218 bytes |

- **WRAM:** no fixed cost — the copy destination is whatever variables you point the event at, so any memory it fills is memory you have already set aside.
- **ROM:** the figure above is the plugin's code only. Each array you generate with the Compile events adds 3 bytes per entry on top.
- **Engine WRAM headroom:** the stock GB Studio 4.3.0 engine leaves about **854 bytes** of WRAM free (usable engine WRAM is 7,776 bytes at 0xC0A0–0xDF00; the stock engine uses 6,922 bytes). With this plugin installed roughly **854 bytes** remain. This figure does not depend on how many global variables your project defines: the script memory array has a fixed size of VM_HEAP_SIZE + (VM_MAX_CONTEXTS × VM_CONTEXT_STACK_SIZE) words — 768 + 16 × 64 = 1,792 words (3,584 bytes) with stock engine settings.
- **SRAM:** not used.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB non-switchable ROM bank that the GB Studio engine core,
the interrupt handlers and the GBDK runtime all share. Banked ROM is cheap
(add another bank), bank 0 is not, so it is usually the first thing a project
runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **0** |
| Bank 0 free with this plugin installed | **1,451** of 16,384 (91% used) |

**This plugin costs nothing in bank 0.** All of its code lives in a switchable
ROM bank; nothing it adds is resident in bank 0.

<details><summary>How this was measured</summary>

GB Studio 4.3.2, DMG target, default engine settings. Each module's bank 0
contribution is the `A _HOME size` record that SDCC writes into its `.rel`
object, summed over the engine sources this plugin provides. Stock sizes come
from building projects whose only plugin ships no engine C, so every module in
them is the untouched engine; two such builds were compared and agreed on all
73 shared modules.

The "free" figure is a stock project with this plugin and nothing else. Your
own number will differ: other plugins, and any engine settings that change what
the core compiles, move it independently of this plugin.

</details>
<!-- BANK0:END -->

## Changelog

Grouped by the date each change was merged into the official
[gb-studio-plugins](https://github.com/gb-studio-dev/gb-studio-plugins) repository.

Only bug fixes, new features and feature changes are listed. Engine version
bumps, patch regeneration, packaging fixes and documentation edits are omitted.

### 2026-06-14

- Added custom script parameter / stack support to the events.

### 2026-02-03

- New tileset / scene array compilation event.

### 2026-01-19

- Initial release.
