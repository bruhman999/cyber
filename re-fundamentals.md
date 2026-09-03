# Reverse Engineering Fundamentals — Beginner Notes

You can't skip around in RE — each layer explains the next one. Read this top to bottom the first time:

**Source code (C) → Compiler (GCC) → Binary file (ELF) → Protections on that binary → Disassembler/Debugger (GDB) → Assembly instructions**

That's the actual pipeline your code goes through, and it's also the order these notes are written in.

---

## Table of Contents
- [1. C Basics — Why RE Starts Here](#1-c-basics--why-re-starts-here)
- [2. Header Files — stdio.h and string.h](#2-header-files--stdioh-and-stringh)
- [3. int main vs void main](#3-int-main-vs-void-main)
- [4. Compiling With GCC](#4-compiling-with-gcc)
- [5. ELF — The Binary File Format](#5-elf--the-binary-file-format)
- [6. Exploit Mitigations (ASLR, NX, PIE, and friends)](#6-exploit-mitigations-aslr-nx-pie-and-friends)
- [7. Symbols and the strip Command](#7-symbols-and-the-strip-command)
- [8. GDB — The Debugger](#8-gdb--the-debugger)
- [9. Reading x86-64 Assembly](#9-reading-x86-64-assembly)
- [10. Full Worked Example](#10-full-worked-example)
- [11. What To Learn Next](#11-what-to-learn-next)

---

## 1. C Basics — Why RE Starts Here

Reverse engineering means taking a compiled binary and figuring out what source code produced it. Most RE targets (CTF challenges, malware, real software) are written in C or C++, and their compiled form is literally C logic translated into assembly. **If you don't know what the C looked like, you can't recognize it in the disassembly** — so we start with a tiny C program and follow it all the way down to machine instructions.

Here's the program we'll use throughout these notes:

```c
#include <stdio.h>

int main() {
    printf("Hello, world!\n");
    return 0;
}
```

Save this as `helloworld.c`. Every section below eventually comes back to this file.

---

## 2. Header Files — stdio.h and string.h

`#include <stdio.h>` at the top of a C file isn't code that runs — it's an instruction to the **preprocessor** (a step that runs before compiling) to paste in declarations for a set of functions before compiling continues.

- **`stdio.h`** = "standard input/output header." Declares functions like `printf` (print text), `scanf` (read input), `fopen`/`fread` (file I/O). Without including it, the compiler wouldn't know what `printf` even is.
- **`string.h`** = declares string-handling functions: `strcpy` (copy a string), `strlen` (get length), `strcmp` (compare strings), `strcat` (concatenate).

**Why this matters for RE specifically:** functions like `strcpy` and `gets` don't check whether the destination buffer is big enough. That's the root cause of most classic buffer overflow vulnerabilities. When you're reading disassembly later and see a call to `strcpy`, that's a flag worth investigating — it's often *the* vulnerability.

---

## 3. int main vs void main

```c
int main() { ... }     // correct, standard C
void main() { ... }    // technically non-standard
```

`main` is the entry point the operating system calls when your program starts. The C standard requires `main` to **return an `int`** — that integer becomes the program's **exit code**, which the shell can check (`echo $?` after running a program shows it). `0` conventionally means "success," non-zero means "something went wrong."

`void main()` (returning nothing) works on some compilers as a non-portable extension, but it's not standard C and you should avoid it — always use `int main()` and end with `return 0;`.

You'll also commonly see:
```c
int main(int argc, char *argv[]) { ... }
```
- `argc` = **arg**ument **c**ount — how many command-line arguments were passed (including the program name itself).
- `argv` = **arg**ument **v**alues — an array of those arguments as strings.

This matters in RE because when you disassemble `main`, you'll often see setup code handling `argc`/`argv` even in programs where they're unused — recognizing that pattern early saves confusion later.

---

## 4. Compiling With GCC

**GCC** (GNU Compiler Collection) turns your `.c` source file into a binary the CPU can actually execute.

```bash
gcc -o helloworld helloworld.c
```

Breaking that down:
- `gcc` — invoke the compiler
- `-o helloworld` — **o**utput file name; the resulting binary will be called `helloworld`. (If you omit `-o`, GCC names it `a.out` by default — an old Unix convention.)
- `helloworld.c` — the source file to compile

Run it: `./helloworld` → prints `Hello, world!`

**Flags you'll see constantly in RE tutorials:**
| Flag | Meaning |
|---|---|
| `-o <name>` | Set output filename |
| `-g` | Include debug symbols (variable/function names, line numbers) — makes GDB much more readable |
| `-no-pie` | Disable PIE (see section 6) — common in beginner exploit tutorials to make addresses predictable |
| `-fno-stack-protector` | Disable stack canaries (see section 6) — also common in beginner exploit practice |
| `-static` | Statically link libraries into the binary instead of relying on shared libraries at runtime |

Example you'll see a lot in intro binary exploitation material:
```bash
gcc -o vuln vuln.c -no-pie -fno-stack-protector -g
```
That's deliberately compiling with protections turned *off* so a beginner can practice exploiting a stack overflow without ASLR/canaries getting in the way first.

---

## 5. ELF — The Binary File Format

**ELF** = **E**xecutable and **L**inkable **F**ormat — the standard file format for executables, shared libraries (`.so` files), and object files on Linux. (Windows uses PE instead; macOS uses Mach-O.) When you compile `helloworld.c`, the output file *is* an ELF file, even though it has no file extension.

Confirm it:
```bash
file helloworld
# helloworld: ELF 64-bit LSB pie executable, x86-64, ...
```

An ELF file is made of:
- **ELF header** — identifies the file as ELF, target architecture, entry point address
- **Program headers** — tell the OS how to load segments into memory to run it
- **Section headers** — tell tools (like GDB) where each named section lives:
  - `.text` — the actual compiled machine code (your instructions)
  - `.data` — initialized global/static variables
  - `.bss` — uninitialized global/static variables
  - `.rodata` — read-only data (e.g. string literals like `"Hello, world!\n"`)
  - `.symtab` — symbol table (function/variable names) — see section 7

Useful commands to poke at an ELF file:
```bash
file helloworld          # quick summary of what kind of binary it is
readelf -h helloworld    # dump the ELF header
readelf -S helloworld    # list all sections
objdump -d helloworld    # disassemble it (see section 9)
checksec --file=helloworld   # check which exploit mitigations are enabled (needs pwntools/checksec installed)
```

---

## 6. Exploit Mitigations (ASLR, NX, PIE, and friends)

These are protections the OS and compiler add to make exploiting memory-corruption bugs (like buffer overflows) harder. You'll run into all of these constantly in binary exploitation:

### ASLR — Address Space Layout Randomization
A **kernel** feature (not compiled into the binary itself) that randomizes *where* things get loaded into memory each time a program runs — the stack, the heap, and shared libraries all get different addresses on every run. Without it, an attacker could hardcode a memory address in an exploit and reuse it every time. Check/toggle system-wide:
```bash
cat /proc/sys/kernel/randomize_va_space   # 0 = off, 2 = full ASLR (typical default)
```

### PIE — Position Independent Executable
This is the *compiler-side* counterpart to ASLR. A normal executable is compiled to always load at a fixed base address. A **PIE** binary is compiled so it can load at *any* base address — which means ASLR can randomize the binary's own code location too, not just the stack/heap/libraries. Compile without it using `-no-pie` (common for beginner practice, since it makes addresses predictable and easier to reason about while learning).

### NX — No-eXecute (also called DEP)
Marks memory regions like the stack as **non-executable**. Historically, a classic stack-overflow exploit would inject malicious machine code (shellcode) onto the stack and jump to it. With NX enabled, the CPU refuses to execute code sitting in a data region like the stack, forcing attackers toward more advanced techniques (like ROP — Return-Oriented Programming, a topic for later).

### Stack Canaries
A random value the compiler places on the stack right before the saved return address. Before a function returns, it checks whether that value has been overwritten. If a buffer overflow smashed through it on the way to overwriting the return address, the canary check fails and the program aborts instead of jumping to attacker-controlled memory. Disable for beginner practice with `-fno-stack-protector`.

### RELRO — RELocation Read-Only
Related to how the **GOT** (Global Offset Table — used to look up addresses of library functions at runtime) can be made read-only after the program loads, so an attacker who gets a write primitive can't overwrite GOT entries to hijack function calls. Comes in "Partial" and "Full" flavors.

**All four at a glance** (this is exactly what the `checksec` tool reports):

| Mitigation | Protects against | Where it comes from |
|---|---|---|
| ASLR | Predictable memory addresses | Kernel setting |
| PIE | Predictable *binary's own* code address | Compiler flag, needs ASLR to matter |
| NX | Executing injected shellcode on stack/heap | CPU feature + compiler/linker flag |
| Stack canary | Stack buffer overflows overwriting return address | Compiler flag |
| RELRO | GOT overwrite attacks | Linker flag |

---

## 7. Symbols and the strip Command

A **symbol** is a human-readable name (function name, variable name) that the compiler embeds in the binary, normally stored in the `.symtab` section. When you compile with `-g` and don't strip the binary, GDB can show you `main`, `printf`, your variable names, etc. instead of just raw addresses.

**`strip`** removes that symbol table (and other debug info) from the binary:
```bash
strip helloworld
```
Compare `nm helloworld` (lists symbols) before and after — after stripping, most of that output disappears. Malware and shipped commercial software are very often stripped specifically to make reverse engineering harder — you have to identify functions by *behavior* instead of by name. This is one of the core skills RE builds toward.

---

## 8. GDB — The Debugger

**GDB** (GNU Debugger) lets you run a binary step-by-step, inspect memory/registers, set breakpoints, and disassemble functions. It's the primary tool you'll live in for RE and binary exploitation.

**Starting it:**
```bash
gdb ./helloworld
```

**`gdb --nx`:** the `--nx` flag tells GDB to skip auto-loading any `.gdbinit` file in the current directory. Normally GDB will automatically run commands from a `.gdbinit` file if one exists — useful for your own config, but a security risk if you're analyzing an untrusted binary someone handed you (a malicious `.gdbinit` sitting next to it could run arbitrary commands the moment you launch GDB). `--nx` disables that auto-loading as a safety measure.

**Essential commands once inside GDB:**
| Command | What it does |
|---|---|
| `break main` (or `b main`) | Set a breakpoint at the start of `main` |
| `run` (or `r`) | Start the program |
| `next` (or `n`) | Step over the next line (doesn't enter function calls) |
| `step` (or `s`) | Step into the next line (enters function calls) |
| `continue` (or `c`) | Resume execution until the next breakpoint |
| `disassemble main` | Show the disassembly of `main` (see section 9) |
| `info registers` (or `i r`) | Show current CPU register values |
| `x/10i $pc` | Examine 10 instructions starting at the current instruction pointer |
| `print <var>` (or `p`) | Print a variable's value |
| `set disassembly-flavor intel` | Switch GDB's assembly output from AT&T to Intel syntax (see below) |

**Why `set disassembly-flavor intel` matters:** x86 assembly can be printed in two different notations that mean the exact same thing:

```
AT&T syntax:    mov    %eax, %ebx        (source, then destination — note the % prefixes)
Intel syntax:   mov    ebx, eax          (destination, then source — no % prefixes)
```
GDB defaults to AT&T syntax on Linux. Most tutorials, and most people learning RE for the first time, find **Intel syntax easier to read** (it's also what Windows/objdump-with-flags and most disassemblers like IDA/Ghidra show by default). Running `set disassembly-flavor intel` inside GDB switches you to it. You can also make it permanent by adding that line to your `~/.gdbinit` file.

---

## 9. Reading x86-64 Assembly

Once you disassemble `main`, you'll see a wall of short instructions. Here's what the recurring ones mean — using Intel syntax (destination first).

### Registers you'll see constantly
| Register | Meaning |
|---|---|
| `rbp` | **Base pointer** — marks the base of the current function's stack frame |
| `rsp` | **Stack pointer** — points to the current top of the stack |
| `rax` | **Accumulator** — commonly holds a function's return value |
| `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9` | Hold the 1st–6th arguments passed into a function (Linux x86-64 calling convention) |
| `rip` | **Instruction pointer** — address of the *next* instruction to execute |

(The `e`-prefixed versions like `eax`, `ebp` are the 32-bit halves of these same registers, left over from x86-32 — you'll see both depending on build.)

### Instructions
| Instruction | Meaning | Example |
|---|---|---|
| `push` | Push a value onto the stack (and decrement `rsp`) | `push rbp` — save the caller's base pointer before this function starts using the stack |
| `pop` | Pop the top value off the stack into a register (and increment `rsp`) | `pop rbp` — restore it before returning |
| `mov` | Copy a value from source to destination | `mov eax, 0` — set `eax` to 0 |
| `lea` | **L**oad **E**ffective **A**ddress — compute an address and store it, *without* reading the memory at that address | `lea rax, [rbp-0x8]` — put the *address* of a local variable into `rax` (not its value). Also frequently used as a fast way to do simple arithmetic (`lea eax, [rax+rax*2]` computes `rax*3`), which trips up beginners the first time they see it |
| `call` (often shown as `callq` in older AT&T output) | Call a function — pushes the return address onto the stack, then jumps to the target | `call printf` |
| `ret` (often `retq`) | Return from the current function — pops the return address off the stack and jumps back to it | `ret` |
| `nop` | **N**o **Op**eration — does literally nothing, just consumes one cycle | Used for instruction alignment/padding by the compiler, and (in exploit development) as a "NOP sled" — a run of `nop`s an attacker jumps into so imprecise jumps still land safely before the real shellcode |

The `q` suffix you sometimes see (`callq`, `retq`, `movq`) is AT&T syntax's way of marking the operand size as a **q**uadword (64-bit). You'll see it disappear if you switch to Intel syntax, since Intel infers size from the registers used instead of suffixing the mnemonic.

### A typical function prologue
Almost every non-optimized function starts with this pattern — memorize it, you'll see it hundreds of times:
```asm
push   rbp           ; save caller's base pointer
mov    rbp, rsp       ; set up this function's own base pointer
sub    rsp, 0x10      ; allocate 16 bytes of local stack space
```
And a matching **epilogue** at the end:
```asm
leave                 ; equivalent to: mov rsp, rbp  +  pop rbp  (undo the prologue)
ret                    ; return to caller
```

### Opcodes and the `/r` notation
The instructions above (`mov`, `push`, `call`...) are the human-readable **mnemonics**. Underneath, each one is actually encoded as raw bytes the CPU decodes — those raw bytes are the **opcode**. When you read Intel's official instruction reference manual, you'll see entries like:

```
ADD r/m32, r32     01 /r
```
That `/r` means: "the ModRM byte that follows the opcode byte encodes a register operand in its `reg` field." It's low-level encoding trivia mainly relevant once you start reading raw hex bytes instead of GDB's disassembled mnemonics (e.g. with `objdump -d` in hex mode, or writing shellcode by hand). As a beginner you don't need to memorize opcode tables — just know the `/r` notation means "there's a register operand here," and that GDB/objdump translate all of this into the mnemonics above for you automatically.

---

## 10. Full Worked Example

Putting it all together, start to finish:

```bash
# 1. Write the source
cat > helloworld.c << 'EOF'
#include <stdio.h>

int main() {
    printf("Hello, world!\n");
    return 0;
}
EOF

# 2. Compile it (with debug symbols, protections off for easier reading as a beginner)
gcc -o helloworld helloworld.c -g -no-pie -fno-stack-protector

# 3. Confirm what kind of file it is
file helloworld

# 4. Open it in GDB
gdb --nx ./helloworld
```

Inside GDB:
```
(gdb) set disassembly-flavor intel
(gdb) break main
(gdb) run
(gdb) disassemble main
(gdb) info registers
(gdb) next
(gdb) continue
```

You'll see `disassemble main` print something like:
```asm
push   rbp
mov    rbp, rsp
sub    rsp, 0x10
lea    rax, [rip+0xe94]        ; address of "Hello, world!\n" string
mov    rdi, rax                 ; set up 1st argument to printf
call   printf
mov    eax, 0x0
leave
ret
```
Reading it: save the base pointer, set up the stack frame, load the address of the string literal, move it into `rdi` (the 1st-argument register), call `printf`, set the return value (`eax`) to 0, tear down the stack frame, and return. That's the entire `int main() { printf(...); return 0; }` you wrote — now in assembly.

---

## 11. What To Learn Next

Once the above feels comfortable, the natural next steps in most RE learning paths are:
- **Buffer overflows** — actually exploiting the `strcpy`-style bugs from section 2, using what you now know about the stack layout from section 9
- **`objdump`** — a command-line disassembler alternative to GDB, good for quickly dumping a whole binary without an interactive session
- **Ghidra or IDA** — free (Ghidra) or paid (IDA) tools that decompile assembly back into C-like pseudocode, which is much faster to read than raw assembly once binaries get bigger
- **ROP (Return-Oriented Programming)** — the technique attackers use to bypass NX by chaining together existing code fragments instead of injecting new code
- **`pwntools`** — a Python library built specifically for writing exploits against binaries, widely used in CTFs
