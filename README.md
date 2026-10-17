# static-gdb-mips32

Statically-linked [GDB](https://www.gnu.org/software/gdb/) that runs **natively on
MIPS32 Linux machines** — single portable binaries with no shared-library
dependencies, for copying onto routers, CPE, embedded boards, or any MIPS32 box
where you can't install packages.

Built for **both endiannesses** and **two variants** each:

| Binary | ~Size | Description |
|---|---|---|
| `gdb-<ver>-mips32-static-nopython` | ~15 MB | Big-endian, TUI, no Python scripting |
| `gdb-<ver>-mips32-static-python` | ~20 MB | Big-endian + embedded CPython 3.12 |
| `gdb-<ver>-mips32el-static-nopython` | ~15 MB | Little-endian, TUI, no Python scripting |
| `gdb-<ver>-mips32el-static-python` | ~20 MB | Little-endian + embedded CPython 3.12 |

Target profile: **o32 ABI, MIPS32r2 baseline, glibc 2.40** (needs a reasonably
modern kernel on the device). Check your device with `readelf -h /bin/sh |
grep -E 'ELF|flags'` — `ELF32` + MSB/LSB tells you the ABI class and endianness.

## Quick start

Requirements: Ubuntu/Debian-ish **x86_64** host (the build cross-compiles), with
`build-essential`, `curl`, and:

```bash
sudo apt-get install -y \
    libgmp-dev libmpfr-dev libexpat1-dev zlib1g-dev libncurses-dev texinfo \
    gcc-14-mips-linux-gnu g++-14-mips-linux-gnu \
    libc6-dev-mips-cross binutils-mips-linux-gnu \
    gcc-14-mipsel-linux-gnu g++-14-mipsel-linux-gnu \
    libc6-dev-mipsel-cross binutils-mipsel-linux-gnu \
    qemu-user-static
```

Then:

```bash
./build.sh
```

Binaries land in `dist/`, full install trees under `install/<triplet>-<variant>/`.
A full build takes roughly 30–45 minutes on a 20-core machine. The script is
re-runnable: sources, host Python, cross libraries, and cross Pythons are
skipped when already present.

### Options

```bash
GDB_VERSION=16.3     PYTHON_VERSION=3.12.7    # upstream versions
TRIPLETS="mips-linux-gnu mipsel-linux-gnu"    # subset to build
VARIANTS="nopython python"                    # subset to build
CROSS_GCC_VER=14    # cross gcc package series on the host
JOBS=$(nproc)       # parallel jobs
BASE=/path/to/dir   # where to place src/deps/build/install/dist (default: repo dir)
```

## Deploying to a device

```bash
scp dist/gdb-16.3-mips32el-static-nopython root@device:/usr/local/bin/gdb
```

For the **python** variants also copy the data-directory (its path is baked in
at build time; keep the relative layout or pass `--data-directory`):

```bash
scp -r install/mipsel-linux-gnu-python/share root@device:/usr/local/
```

## Verifying on an x86_64 host (qemu)

The build script sanity-checks every binary under qemu-user. You can also run a
full end-to-end debug session on your workstation:

```bash
# target program under qemu's gdbserver stub
qemu-mipsel-static -g 12345 ./my-mips32el-binary &
qemu-mipsel-static ./dist/gdb-16.3-mips32el-static-nopython \
    -batch -ex 'file ./my-mips32el-binary' -ex 'target remote :12345' \
    -ex 'break main' -ex continue -ex 'info registers pc'
```

## Caveats

- **Thread debugging is limited**: a glibc-static executable cannot `dlopen`
  `libthread_db` (same limitation as any statically linked GDB).
- **Python variant**: extension modules relying on `dlopen` cannot load
  (`import ctypes` fails); pure-Python GDB scripts work. pwndbg/gef will not
  work with a static build.
- Native target only: these debug **o32** userland on the matching endianness.
  To cross-debug MIPS32 from x86_64 instead, a `--enable-targets=all` x86_64
  build is a better fit.
- Static glibc 2.40 binaries need a kernel of roughly 3.2 or newer on the device.

## How it works / notable gotchas

1. **Cross toolchain**: Ubuntu's `gcc-14-<triplet>` + `libc6-dev-*-cross`
   packages (static glibc archives included). GDB's support libraries
   (GMP/MPFR — hard requirements of GDB 16 — plus expat, zlib, ncurses) are not
   packaged for cross use, so they are cross-built statically from source.
2. **libtool swallows `-static`**: GDB's final link runs through libtool, which
   consumes a plain `-static` flag and silently produces a dynamic binary. The
   build relinks with `make -C gdb LDFLAGS="-all-static"`.
3. **qemu-user as the Python bridge**: GDB's configure must *execute* the
   Python it links against. The cross-built Python (static libpython, dynamic
   glibc) is wrapped in a `qemu-<arch>-static -L /usr/<triplet>` script passed
   to `--with-python=` — configure runs the target interpreter under emulation
   and link-tests the real cross `libpython3.12.a`.
4. **CPython 3.12 cross quirks**: `--with-build-python=<host python>` is
   required; `ac_cv_buggy_getaddrinfo=no` (a runtime test false-fails when
   cross-compiling); out-of-tree builds require a pristine source tree, which
   is why every Python build here (host and cross) is out-of-tree.
5. **ncurses widec**: cross ncurses installs only the widec flavor
   (`libncursesw.a`), while readline/TUI look for `-lncurses`/`-ltinfo`; the
   build provides symlink aliases (ABI-compatible for readline's use).

Build layout (all gitignored): `src/`, `deps/` (host + cross Pythons and
libraries), `build-*/`, `install/`, `dist/`.

## License

GDB is GPLv3; statically-linked GDB binaries are derivative works, so if you
distribute the binaries you must offer the corresponding source (this repo plus
the pinned upstream versions is a reasonable way to do that). No license is
claimed over these build files — add one of your choosing before publishing.
