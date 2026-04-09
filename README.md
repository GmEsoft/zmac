# zmac

A classic **Z80 / 8080 / Z180 macro cross-assembler**, preserved and modernized for contemporary use.

This repository contains the source for `zmac`, a long-lived assembler originally written in **1978** and later expanded with modern tooling support, richer expression handling, multiple output formats, compatibility modes, and relocatable `.REL` output. It remains especially useful for **CP/M**, **TRS-80**, **ZX Spectrum**, and other retro-computing workflows built around the **Zilog Z80** family.

- **GitHub repository:** <https://github.com/GmEsoft/zmac>
- **Historic/project page by George Phillips:** <http://48k.ca/zmac.html>
- **Z80 background:** <https://en.wikipedia.org/wiki/Zilog_Z80>

---

## ✨ Highlights

`zmac` includes support for:

- powerful **macro assembly** and **conditional assembly**
- **Z80**, **8080**, and **Z180** modes
- multiple output formats including `.cmd`, `.hex`, `.cim`, `.cas`, `.tap`, and `.rel`
- listing files with **cycle counts** and symbol information
- compatibility-oriented options for older assembler dialects such as **MRAS**, **MACRO-80**, and **DRI** styles
- generated documentation from `src/doc.txt` / `src/doc.c`

For the reverse-engineered `.REL` object format used by this tree, see [`REL_FORMAT.md`](./REL_FORMAT.md).

---

## 📁 Project layout

| Path | Purpose |
|---|---|
| `src/zmac.y` | Main assembler grammar and core implementation |
| `src/doc.txt` | End-user documentation source |
| `src/doc.c` | Documentation generator (`HTML` / man page) |
| `src/zi80dis.c`, `src/zi80dis.h` | Z80 instruction decoding/timing support |
| `src/mio.c`, `src/mio.h` | Memory/file I/O helpers |
| `REL_FORMAT.md` | Notes on the `.REL` structure emitted by zmac |

---

## 🔧 Building

### Windows
From `src/`:

```bat
build.bat
```

This uses:

- **GCC:** <https://gcc.gnu.org/>
- **GNU Bison:** <https://www.gnu.org/software/bison/>

### Unix-like systems
From `src/`:

```sh
make
```

---

## 🚀 Quick usage

```sh
zmac --help
zmac myprog.z
zmac --rel mymodule.z
```

The built-in documentation in `src/doc.txt` describes the supported directives, options, compatibility modes, and output formats in much more detail.

---

## 🙏 Acknowledgements

This project reflects work by many people over several decades. Based on the in-source history and documentation, thanks are due to:

### Original foundation
- **Bruce Norskog** — original author of `zmac` in 1978
- **Ken Borgendale** — author of the Intel 8080 macro cross-assembler that inspired the original design

### Major maintainers and contributors named in the project docs
- **John Providenza**
- **Colin Kelley**
- **Russell Marks**
- **Mark RISON**
- **Chris Smith**
- **Matthew Phillips**
- **Tim Mann**
- **George Phillips** — extensive modernization, cycle counting, additional output formats, `.REL` support, compatibility work, and documentation updates

### Additional contributors and people thanked in the source history
- **Mark Galanger**
- **Sergey Erokhin**
- **Pedro Gimeno**
- **Tim Halloran**
- **Paulo Silva**
- **Peter Wilson**
- **Douglas Miller**
- **Blair Robins**
- **Al Petrofsky**
- **tjh**
- **abog**

If a name has been missed, it is unintentional; this list was assembled from the comments and credits currently present in `src/zmac.y` and `src/doc.txt`.

---

## 📚 Related links

- **George Phillips' zmac page:** <http://48k.ca/zmac.html>
- **TRS-80GP emulator** (referenced in the docs): <https://www.48k.ca/trs80gp.html>
- **Creative Commons CC0 1.0** (referenced by `src/doc.c`): <https://creativecommons.org/publicdomain/zero/1.0/>
- **GNU Bison:** <https://www.gnu.org/software/bison/>
- **GCC:** <https://gcc.gnu.org/>

---

## 📜 License note

This repository now includes a top-level [`LICENSE`](./LICENSE) file identifying the project as **CC0 1.0 Universal** (`CC0-1.0`).

That matches the public-domain dedication reflected in `src/doc.c` and on George Phillips' zmac project page: to the extent possible under law, copyright and related rights in this work have been waived. For full details, see the local `LICENSE` file or the official CC0 text at <https://creativecommons.org/publicdomain/zero/1.0/>.
