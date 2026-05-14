# `binpatch` – Binary Patching Tool

---

## Table of Contents

1. [Introduction](#introduction)
2. [Synopsis](#synopsis)
3. [Options](#options)
4. [File Offsets vs. Virtual Addresses (`-o` vs `-va`)](#file-offsets-vs-virtual-addresses--o-vs--va)
5. [Hex Input Flexibility](#hex-input-flexibility)
6. [Find Limits & Terminal Flooding](#find-limits--terminal-flooding)
7. [Heuristic Find (`-fh`) – Advanced Search](#heuristic-find--fh--advanced-search)
8. [Entry Point & Main Resolution (`-e`, `-m`, `-r`)](#entry-point--main-resolution--e--m--r)
9. [Exit Codes](#exit-codes)
10. [Dependencies](#dependencies)
11. [Examples](#examples)
12. [License](#license)

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
| `-o`, `--offset` | OFFSET | **Raw File Offset** to patch or disassemble. (e.g., `112b` or `0x112b`). |
| `-va`, `--vaddr` | VMA | **Virtual Memory Address** (from `objdump`/IDA). Auto-translates to the correct File Offset. |
| `-e`, `--entry` | (none) | Automatically parse the ELF file to target the Entry Point (`_start`). Replaces `-o`. |
| `-m`, `--main` | (none) | Automatically target the `main()` function in compiled C/C++ binaries. Replaces `-o`. |
| `-h`, `--hex` | HEX_STRING | Hex bytes to write to the file (e.g., `"cb 10 00 00 05"`). |
| `-b`, `--backup` | (none) | Create a timestamped backup of the file before applying the patch. |
| `-f`, `--find` | HEX_STRING | Find an exact hex pattern in the file and print an `xxd`-style hex dump of occurrences. |
| `-fh`, `--find-heuristic`| HEX_STRING | Find the largest contiguous substring match (heuristic search). |
| `-d`, `--disassemble` | (none) | Disassemble instructions at the target offset (Intel syntax for x86/AMD64). |
| `-s`, `--size` | N | **Dual-purpose:** Number of instructions to disassemble (default: 1), OR max number of find results to display (default: 5). |
| `-r`, `--return` | (none) | Dynamically disassemble until a return instruction is hit (`ret`, `bx lr`, `pop {pc}`, etc.). Cannot be used with `-s`. |
| `-a`, `--all` | (none) | Show ALL find results, overriding the `-s` size limit. |
| `-q`, `--quiet` | (none) | Scripting mode. Suppresses all visual formatting and prints *only* raw hex offsets (`0x...`). |
| `--version` | (none) | Show version `binpatch 1.0.0`. |
| `--help` | (none) | Show the help message and exit. |

*(Note: `-h` maps to `--hex`, not help. Use `--help` for the manual).*

---

## File Offsets vs. Virtual Addresses (`-o` vs `-va`)

Reverse engineers often copy addresses from `objdump`, `Ghidra`, or `IDA Pro`. These tools output **Virtual Memory Addresses (VMAs)** (e.g. `0x4011e0`), whereas a file on a hard drive is accessed via **File Offsets** (e.g. `0x11e0`).

- If you have an exact File Offset from `-f` (Find), use **`-o`**.
- If you copy a memory address directly from disassembly output, use **`-va`**. `binpatch` will natively parse the ELF header and map it perfectly to the correct physical File Offset before patching.

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

## Find Limits & Terminal Flooding

To prevent your terminal from exploding when searching for common bytes (like `00 00`), `binpatch` automatically limits find results to the **first 5 matches**. 

If more matches exist, it will hide them and notify you at the bottom of the output. 
- Use `-a` or `--all` to force it to print every single match.
- Use `-s N` to change the limit to a specific number (e.g., `-s 20` shows 20 results).

---

## Heuristic Find (`-fh`) – Advanced Search

If a binary has been slightly updated or recompiled, an exact byte signature might break. The Heuristic Find (`-fh`) solves this by generating all contiguous substrings of your hex pattern and searching for the largest surviving chunks.

*(Note: Due to the substring search method, you may see adjacent overlapping matches. This is normal for a heuristic approach).*

---

## Entry Point & Main Resolution (`-e`, `-m`, `-r`)

`binpatch` contains advanced structural resolution for ELF binaries.

- **`-e` (Entry Point):** Natively parses the ELF header to resolve the Virtual Address of `_start`, translates it to a physical file offset, and targets it.
- **`-m` (Main):** Uses `objdump` to scan the binary's symbol table for the `main()` function, translating it to the physical file offset automatically. *(Fails gracefully on stripped binaries).*

If you use `-r` alongside `-d`, `binpatch` streams the disassembly line-by-line and will automatically stop when it hits an architecture-specific return instruction (e.g., `ret` for x86, `bx lr` or `pop {pc}` for ARM, `jr ra` for MIPS). 

---

## Backup Format

When the `-b` (`--backup`) flag is used alongside a write operation (`-h`), the original file is copied safely before modifications are made.

**Format:** `backup_YYYYMMDD_HHMMSS_originalfilename`

*Example:* `backup_20260513_141527_my_program`

---

## Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success. |
| `1` | Error (File not found, invalid offset, conflict, or match not found). |

---

## Dependencies

- **Python 3.6+** (Uses standard library only; no `pip` installs required).
- **`objdump`** (Optional). Required only if you use the `-d` (disassemble) or `-m` (main) flags. Usually installed by default on Linux via `binutils`.

---

## Examples

### 1. NOP out a Jump Instruction (Game Hacking)
Copy the Virtual Address of the instruction straight from IDA Pro or `objdump`, translate it safely with `-va`, and overwrite it with NOPs (`90 90`).
```bash
binpatch c_binary -va 4011cd -h "90 90"
```

### 2. Disassemble the `main()` function
Automatically resolve the `main()` symbol and disassemble the entire function until it returns.
```bash
binpatch c_binary -m -d -r
```

### 3. Patch at raw offset with backup
Write 5 bytes to file offset `0x112B` and create a timestamped backup first.
```bash
binpatch my_program -o 112B -h "cb 10 00 00 05" -b
```

### 4. Combine finding and patching (Scripting)
Using the `--quiet` (`-q`) flag, `binpatch` outputs *only* raw hexadecimal addresses. 
```bash
OFFSET=$(binpatch my_program -f "cb 10 00 00 05" -q | head -1)
binpatch my_program -o $OFFSET -h "90 90 90 90 90" -b
```

---

## License

This project is licensed under the [GNU General Public License v3.0 (GPLv3)](https://www.gnu.org/licenses/gpl-3.0.html).
