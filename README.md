<!---
{
  "id": "6acc08b3-b4b8-402c-931b-26d29d81c063",
  "teaches": "C Libraries: Building and Loading a Shared Object Without Root",
  "depends_on": [
    "05e99c97-2969-45b9-b955-5b5a62a0786e"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-01",
  "keywords": [
    "C",
    "shared library",
    ".so",
    "-fPIC",
    "-shared",
    "ldd",
    "LD_LIBRARY_PATH",
    "rpath",
    "$ORIGIN",
    "dynamic linker"
  ]
}
--->

# C Libraries: Building and Loading a Shared Object Without Root

> In this exercise you will compile `vec2` into a shared object `libvec2.so`, link a program against it, and watch it fail at start-up because the dynamic linker cannot find the library. You will fix that three different ways, all without `sudo` and without `ldconfig`, and finally patch the library without rebuilding the program.

## Introduction

A **static** library is copied into every executable that uses it. The result is self-contained but large, and a bug in the library can only be fixed by relinking every program.

A **shared** library (`.so`, *shared object*) is not copied. The executable only records *that* it needs `libvec2.so`. When the program starts, the **dynamic linker** (`ld.so`) finds the file, loads it into memory and connects the calls. Several programs can share one copy in memory, and replacing the `.so` on disk patches all of them at once.

The price is a search problem: *where* is `libvec2.so` at run time? On a normal system the answer is `/usr/lib`, maintained with `ldconfig`, both of which need root. But the dynamic linker has several other places it looks, and three of them are fully under your control:

1. the environment variable `LD_LIBRARY_PATH`,
2. a search path baked into the executable, the **rpath**,
3. an rpath relative to the executable's own location, using the token `$ORIGIN`.

You will use all three.

### 1.1) Further Readings and Other Sources

* [ld.so(8) man page: the search order of the dynamic linker](https://man7.org/linux/man-pages/man8/ld.so.8.html)
* [ldd(1) man page](https://man7.org/linux/man-pages/man1/ldd.1.html)
* [Shared libraries (Linux Documentation Project)](https://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)
* [Drepper: How to Write Shared Libraries](https://www.akkadia.org/drepper/dsohowto.pdf)
* [Linkers, Loaders and Shared Libraries, CppCon 2023 (Moodle, chapter 32)](https://moodle.thga.de/course/view.php?id=2448&section=32)

## Tasks

### Task 1: Build the Shared Object

Start in `~/vec2` with the clean `vec2.h` and `vec2.c` from the static library exercise.

```bash
gcc -Wall -Wextra -fPIC -shared vec2.c -o libvec2.so -lm
ls -l libvec2.so
file libvec2.so
nm -D libvec2.so
```

Look up `-fPIC` in `man gcc`. `nm -D` shows the *dynamic* symbol table, the part that other programs can see.

### Task 2: Link Against It and Fail

```bash
gcc -Wall -Wextra main.c -I. -L. -lvec2 -lm -o main
./main
```

Read the error message carefully. It is not from `gcc` and not from `ld`. Which program printed it?

```bash
ldd ./main
```

Find the line for `libvec2.so`. Then compare with `libm` and `libc`, which are found. Where are they located?

### Task 3: Fix 1, the Environment Variable

```bash
LD_LIBRARY_PATH=. ./main
LD_LIBRARY_PATH=. ldd ./main
```

Now move the binary somewhere else and run it from there:

```bash
mkdir -p ~/bin-test && cp main ~/bin-test/ && cd ~/bin-test
LD_LIBRARY_PATH=. ./main
LD_LIBRARY_PATH=~/vec2 ./main
cd ~/vec2
```

Explain why the first attempt fails and the second works. Why is `LD_LIBRARY_PATH` considered a debugging aid rather than a deployment solution?

### Task 4: Fix 2, an Absolute rpath

Relink the program and bake the search path into the executable:

```bash
gcc -Wall -Wextra main.c -I. -L. -lvec2 -lm -Wl,-rpath,$HOME/vec2 -o main
./main
ldd ./main
readelf -d ./main | grep -i 'rpath\|runpath'
```

The `-Wl,` prefix passes the option through `gcc` to the linker. Copy `main` to `~/bin-test` and run it from there. It works now, without any environment variable.

### Task 5: Fix 3, a Relative rpath With `$ORIGIN`

Absolute paths break as soon as the directory moves. The token `$ORIGIN` means "the directory of the executable itself". Build a small relocatable tree:

```bash
mkdir -p ~/vec2-app/bin ~/vec2-app/lib
cp libvec2.so ~/vec2-app/lib/
gcc -Wall -Wextra main.c -I. -L. -lvec2 -lm -Wl,-rpath,'$ORIGIN/../lib' -o ~/vec2-app/bin/main
~/vec2-app/bin/main
mv ~/vec2-app ~/vec2-app-moved
~/vec2-app-moved/bin/main
```

The single quotes are important: the shell must not expand `$ORIGIN`. Check with `readelf -d` that the string `$ORIGIN/../lib` was stored literally.

### Task 6: Patch the Library, Not the Program

Change the behaviour of the library. In `vec2.c`, make `VEC2_len` print a message:

```c
#include <stdio.h>
/* ... */
double VEC2_len(vec2 a) {
    fprintf(stderr, "[libvec2 v2] computing length\n");
    return sqrt(VEC2_dot(a, a));
}
```

Rebuild **only** the shared object and copy it over the old one:

```bash
gcc -Wall -Wextra -fPIC -shared vec2.c -o libvec2.so -lm
cp libvec2.so ~/vec2-app-moved/lib/
~/vec2-app-moved/bin/main
```

The program was not recompiled and not relinked, yet it shows the new behaviour. This is the "patch a security hole in one place" advantage from the lecture, reproduced in your home directory.

### Task 7: Versioned Sonames (optional)

Real shared libraries carry a version in their *soname*. Build one:

```bash
gcc -fPIC -shared -Wl,-soname,libvec2.so.1 vec2.c -o libvec2.so.1.0 -lm
ln -sf libvec2.so.1.0 libvec2.so.1
ln -sf libvec2.so.1   libvec2.so
readelf -d libvec2.so.1.0 | grep SONAME
gcc main.c -I. -L. -lvec2 -lm -Wl,-rpath,'$ORIGIN' -o main
readelf -d main | grep NEEDED
```

Which name does `main` record as needed: `libvec2.so`, `libvec2.so.1` or `libvec2.so.1.0`? Why is that the right choice?

## Questions

- Which program prints "error while loading shared libraries", and at which point in time does it run?
- Why must a shared library be compiled with `-fPIC`, while a static library does not need it?
- List the search order of the dynamic linker as documented in `man ld.so`. Where do `LD_LIBRARY_PATH`, rpath and `/usr/lib` sit in that order?
- Why is `$ORIGIN` more robust than an absolute rpath? Name a situation where it still breaks.
- In Task 6 the program picked up the new library without relinking. What would happen if the new `libvec2.so` had removed the function `VEC2_len`? When would the error appear?
- What does `ldconfig` do that you replaced with `-rpath` in this exercise?

## Advice

Shared libraries are the normal case on every Linux system you will ever use: `ldd /bin/ls` shows a dozen of them. Knowing the search order of `ld.so` and the difference between `LD_LIBRARY_PATH` and rpath saves hours of debugging later, especially when you ship software to a machine you do not control.

Rule of thumb: `LD_LIBRARY_PATH` for experiments in the terminal, `$ORIGIN`-relative rpath for software you distribute as a directory tree, and the system paths with `ldconfig` only when you are the package maintainer.

Task 6 shows the big advantage of dynamic linking and at the same time its big risk. The program trusts that the library it finds at run time still has the same ABI as the one it was linked against. Sonames and versioned symbols exist to manage exactly that trust. If you ever maintain a library that other people's binaries depend on, read the Drepper paper from the further readings before you change a single prototype.
