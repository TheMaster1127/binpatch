# `binpatch` – Binary Patching Tool

---

## Table of Contents

1. [Introduction](#introduction)
2. [Synopsis](#synopsis)
3. [Options](#options)
4. [Hex Input Flexibility](#hex-input-flexibility)
5. [Heuristic Find (`-fh`) – Advanced Search](#heuristic-find--fh--advanced-search)
6. [Backup Format](#backup-format)
7. [Exit Codes](#exit-codes)
8. [Dependencies](#dependencies)
9. [Examples](#examples)
    - [Patching with Backup](#1-patch-at-offset-with-backup)
    - [Exact Find](#2-exact-find-all-locations)
    - [Heuristic Find](#3-heuristic-find-largest-substring-match)
    - [Disassembly](#4-disassemble-at-offset)
    - [Scripting / Automation](#5-combine-finding-and-patching-scripting)
10. [License](#license)

---

## Introduction

`binpatch` is a powerful, dependency-free command-line utility designed to patch, find, and disassemble bytes directly inside binary files. It simplifies low-level binary editing by combining the best features of `dd`, `xxd`, and `objdump` into a single, scriptable interface. 

Features include automatic timestamped backups, highly flexible hex string parsing, and an advanced heuristic search engine for finding partial byte matches.

---

## Synopsis

```bash
binpatch <file> [OPTIONS]
```

---

## Options

| Option | Argument | Description |
|--------|----------|-------------|
| `-o`, `--offset` | OFFSET | Offset in file to patch or disassemble. Accepts decimal (`4395`) or hex (`0x112B`). |
| `-h`, `--hex` | HEX_STRING | Hex bytes to write to the file (e.g., `"cb 10 00 00 05"`). |
| `-b`, `--backup` | (none) | Create a timestamped backup of the file before applying the patch. |
| `-f`, `--find` | HEX_STRING | Find an exact hex pattern in the file and print an `xxd`-style hex dump of all occurrences. |
| `-fh`, `--find-heuristic`| HEX_STRING | Find the largest contiguous substring match (heuristic search). Useful when exact offsets have shifted. |
| `-d`, `--disassemble` | (none) | Disassemble instructions at the given `-o` offset. |
| `-s`, `--size` | N | Number of instructions to disassemble when using `-d` (default: 1). |
| `-q`, `--quiet` | (none) | Scripting mode. Suppresses all visual formatting and prints *only* raw hex offsets (`0x...`). |
| `--help` | (none) | Show the help message and exit. |

*(Note: `-h` maps to `--hex`, not help. Use `--help` for the manual).*

---

## Hex Input Flexibility

The `-h`, `-f`, and `-fh` flags are designed to accept hexadecimal input copied from almost any reverse-engineering tool. It normalizes the string by converting to lowercase, stripping `0x` prefixes, removing commas, and ignoring whitespace.

All of these inputs are treated as exactly the same byte sequence:
- `"cb 10 00 00 05"` *(space-separated)*
- `"cb10000005"` *(no spaces)*
- `"0xcb 0x10 0x00 0x00 0x05"` *(with 0x prefixes)*
- `"CB 10 00 00 05"` *(uppercase)*
- `"cb, 10, 00, 00, 05"` *(stray commas)*

---

## Heuristic Find (`-fh`) – Advanced Search

If a binary has been slightly updated or recompiled, an exact byte signature might break. The Heuristic Find (`-fh`) solves this by generating all contiguous substrings of your hex pattern and searching for the largest surviving chunks.

If you search for `"cb 10 00 00 05"`, `binpatch` will attempt to find the full string. If it fails, it recursively slices it down, checking:
1. `cb 10 00 00 05` (5 bytes)
2. `cb 10 00 00` / `10 00 00 05` (4 bytes)
3. `cb 10 00` / `00 00 05` ... (3 bytes)

It collects all matches across the binary, sorts them by size (largest first), and filters the output to prevent terminal flooding (e.g., ignoring millions of 1-byte `00` matches if a 4-byte match was found).

---

## Backup Format

When the `-b` (`--backup`) flag is used alongside a write operation (`-h`), the original file is copied safely before modifications are made. Metadata is preserved.

**Format:** `backup_YYYYMMDD_HHMMSS_originalfilename`

- `YYYY` – Year
- `MM` – Month (01–12)
- `DD` – Day (01–31)
- `HH` – Hour (00–23, 24-hour)
- `MM` – Minute (00–59)
- `SS` – Second (00–59)

*Example:* `backup_20260513_141527_my_program`

---

## Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success. |
| `1` | Error (File not found, invalid offset, hex parsing error, or exact match `-f` not found). |
| `2` | No match found during heuristic search (`-fh`). |

---

## Dependencies

- **Python 3.6+** (Uses standard library only; no `pip` installs required).
- **`objdump`** (Optional). Required only if you use the `-d` (disassemble) flag. Usually installed by default on Linux via `binutils`.

---

## Examples

### 1. Patch at offset with backup
Write 5 bytes to offset `0x112B` and create a timestamped backup first.
```bash
binpatch my_program -o 0x112B -h "cb 10 00 00 05" -b
```

### 2. Exact find (all locations)
Search for a byte sequence. `binpatch` generates an independent, `xxd`-style hex dump with `^^` carets highlighting the exact location in memory.
```bash
binpatch my_program -f "cb 10 00 00 05"
```
*Output:*
```text
Found at offset 0x112b:
00001120: 12 34 cb 10 00 00 05 90  00 00 00 00 00 00 00 00  |.4.............|
                ^^ ^^ ^^ ^^ ^^ 
```

### 3. Heuristic find (Largest substring match)
Find the largest surviving chunk of a broken byte signature.
```bash
binpatch my_program -fh "cb 10 00 00 05"
```
*Output:*
```text
Largest match: 4 bytes at offset 0x112b
00001120: 12 34 cb 10 00 00 90 90  00 00 00 00 00 00 00 00  |.4.............|
                ^^ ^^ ^^ ^^ 
```

### 4. Disassemble at offset
Disassemble 10 instructions starting at `0x112B`. If combined with `-h`, it writes the patch *first*, then disassembles so you can instantly verify your injected assembly.
```bash
binpatch my_program -o 0x112B -d -s 10
```

### 5. Combine finding and patching (Scripting)
Using the `--quiet` (`-q`) flag, `binpatch` outputs *only* raw hexadecimal addresses. This makes it incredibly powerful for bash scripting.

*Find a signature, grab the first offset, and overwrite it with NOPs (`90`):*
```bash
OFFSET=$(binpatch my_program -f "cb 10 00 00 05" -q | head -1)
binpatch my_program -o $OFFSET -h "90 90 90 90 90" -b
```

---

## License

This project is licensed under the [GNU General Public License v3.0 (GPLv3)](https://www.gnu.org/licenses/gpl-3.0.html).
