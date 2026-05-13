# Installation Guide for `binpatch`

## Prerequisites
`binpatch` requires **Python 3.6+** (Standard library only, no pip installs required). 

For the disassemble feature (`-d`), you will need `objdump` installed on your system. This is usually installed by default on Linux, but if you don't have it, install `binutils`:
```bash
# Arch / Artix Linux
sudo pacman -S binutils

# Debian / Ubuntu
sudo apt install binutils
```

## System-wide Installation (Linux / macOS)

To install `binpatch` so you can run it from anywhere, follow these steps:

1. **Move the script to your local bin:**
   ```bash
   sudo cp binpatch /usr/local/bin/binpatch
   ```

2. **Make it executable:**
   ```bash
   sudo chmod +x /usr/local/bin/binpatch
   ```

3. **Verify the installation:**
   ```bash
   binpatch --help
   ```

## One-liner Installation (Direct from GitHub)
You can also install it directly without cloning the whole repo:
```bash
sudo wget https://raw.githubusercontent.com/TheMaster1127/binpatch/main/binpatch -O /usr/local/bin/binpatch && sudo chmod +x /usr/local/bin/binpatch
```

## Windows Installation
1. Copy `binpatch` to a folder (e.g., `C:\bin`).
2. Add `C:\bin` to your system **PATH** environment variable.
3. You can then run it via `python binpatch`.
