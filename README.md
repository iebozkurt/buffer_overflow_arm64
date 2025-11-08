# Buffer Overflow Lab on ARM64

This repository adapts the classic SEED "Buffer Overflow Attack Lab" to 64-bit ARM (AArch64) systems so you can reproduce the exploit end-to-end on Apple silicon Macs and other ARM64 hardware.  The original lab assumes x86 binaries and relies on an older `strcpy` implementation whose overflow behavior no longer appears on hardened ARM64 libc builds.  To mirror the original walkthrough, this version ships a compatible copy routine (`scpy`) plus a containerized Fedora 29 environment where all relevant mitigations can be disabled for educational use.

## 1. Lab Overview

The goal of the lab is to craft an input file (`badfile`) that overflows a stack buffer in `stack.c`, overwrites the saved return address, and redirects execution into injected shellcode that spawns a shell.  You will:

1. Inspect the vulnerable program and confirm the absence of runtime protections.
2. Generate and validate the provided ARM64 shellcode.
3. Build an exploit payload that places the shellcode on the stack and forces control flow to jump to it.
4. Experiment with payload variations to solidify your understanding of stack layouts on AArch64.

### Deliverables (suggested)

* Annotated notes or screenshots that show how you derived offsets and addresses.
* The crafted `badfile` payload that successfully spawns a shell.
* Optional: A write-up discussing how modern mitigations such as Stack Canaries, ASLR, and non-executable stacks would block this attack.

## 2. Repository Layout

```
.
├── c_programs/
│   ├── stack.c            # Vulnerable program using custom scpy helper
│   ├── exploit.c          # Skeleton for constructing badfile
│   ├── exploit-modified.c # Reference solution from the original PDF
│   └── call_shellcode.c   # Harness for testing shellcode
├── Dockerfile             # Builds Fedora 29 lab environment
├── docker-compose.yml     # Convenience wrapper
└── README.md              # This guide
```

### About `scpy`

The upstream lab copies attacker input into a 12-byte stack buffer with `strcpy`.  On many current AArch64 platforms, `strcpy` was rewritten to use vectorized instructions that incidentally avoid the overflow sequence used in the original tutorial.  To keep the vulnerability intact, `stack.c` includes a custom helper named `scpy` that mimics the historical behavior: it copies eight bytes per iteration with no bounds checking.  When you compile `stack.c`, the helper is linked directly into the binary, guaranteeing that the overflow is observable regardless of your host C library.

## 3. Lab Environment

### Prerequisites

* Docker 20.10 or newer (Docker Desktop on macOS works on Apple silicon)
* `docker compose` plugin (v2 syntax works, but v1 is acceptable)

> The container runs in privileged mode so that you can disable kernel features like ASLR.  Review the `docker-compose.yml` file and ensure you are comfortable with the granted capabilities before running the lab.

### Building and Running the Container

```bash
git clone https://github.com/<your-account>/buffer_overflow_arm64.git
cd buffer_overflow_arm64
docker compose build
docker compose run --rm buffer-overflow-arm
```

The container drops you into a Fedora 29 root shell with the lab materials located at `/home/student/c_programs`.  Switch to the non-root `student` account if you want to mirror the original SEED instructions:

```bash
su - student  # password: password
cd c_programs
```

## 4. Disabling Protections

The classic buffer overflow exploit relies on weakening or disabling several modern safeguards.  The Docker image is pre-configured to make this easy, but you should verify each protection manually as part of the lab write-up.

1. **Stack canaries (StackGuard)** – Compile with stack protection disabled:
   ```bash
   gcc -g -fno-stack-protector -z execstack -no-pie stack.c -o stack
   ```
2. **Non-executable stack** – The `-z execstack` flag marks the stack as executable so the injected shellcode can run.
3. **Address Space Layout Randomization (ASLR)** – Disable ASLR inside the container session:
   ```bash
   echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
   ```
4. **Position-independent executables (PIE)** – The `-no-pie` flag keeps the binary at a fixed load address, simplifying return address calculations.

> Re-run the compilation command any time you modify the source.  If you restart the container, confirm ASLR is still disabled.

## 5. Lab Tasks

The tasks below parallel the steps in the original PDF while highlighting ARM64-specific details.

### Task 1: Understand the Vulnerable Program

1. Open `stack.c` and identify the `bof` function where `scpy` copies user input into the 12-byte buffer.
2. Compile the program with protections disabled (see Section 4).
3. Launch `gdb -q ./stack`, disassemble `bof`, and note the saved return address location relative to the buffer (use `info frame` and `x/40gx $sp`).
4. Record the buffer size, saved frame pointer, and saved program counter offsets; you will need them when crafting the payload.

### Task 2: Warm-Up Shellcode Test

1. Build the harness:
   ```bash
   gcc -g -fno-stack-protector -z execstack -no-pie call_shellcode.c -o call_shellcode
   ```
2. Run `./call_shellcode` to verify that the embedded ARM64 shellcode spawns a shell.  Exit the spawned shell with `exit`.
3. Inspect the shellcode bytes inside `call_shellcode.c` to understand how the payload sets up registers and invokes `execve`.

### Task 3: Crafting `badfile`

1. Study `exploit.c`.  The provided template already loads the shellcode into the buffer and pads the rest of the file with `0x90`-style AArch64 NOP instructions (`0xD503201F`).
2. Update the placeholder offsets:
   * Set `NOP_SIZE` and `RET_COUNT` based on your stack measurements from Task 1.
   * Replace the `/* RETURN ADDRESS HERE */` comment with the address you want execution to jump to—typically the start of your NOP sled.
3. Compile and run the generator:
   ```bash
   gcc exploit.c -o exploit
   ./exploit
   ```
   This produces `badfile` in the current directory.
4. Launch `./stack` (from the same directory).  When prompted, the program reads `badfile`, the overflow overwrites the saved return address, and execution should land in your shellcode.
5. If the program crashes before reaching the shellcode, inspect the stack in GDB to adjust padding and the target address.

### Task 4: Experimentation and Variations

* Replace the NOP sled with alternative instructions and observe how precise the return address must be on AArch64.
* Modify `scpy` to copy four bytes at a time and see how the required offsets change.
* Re-enable mitigations one at a time (Stack Protector, ASLR, NX stack) to understand which defense prevents which stage of the attack.
* Compare the final `badfile` with the reference `exploit-modified.c`, which mirrors the solution from the PDF.

## 6. Debugging Tips

* Use `hexdump -C badfile | head` to confirm your payload structure after each tweak.
* In GDB, set a breakpoint on `bof`, run the program, and dump the stack with `x/80gx $sp` right after `scpy` returns to ensure the saved return address is overwritten with your chosen pointer.
* Check register states with `info registers` to verify that the stack pointer (`sp`) and frame pointer (`x29`) match your calculations.

## 7. Common Issues

| Symptom | Suggested Fix |
| --- | --- |
| `Segmentation fault` before shell runs | Ensure the saved return address is overwritten with an aligned pointer (8-byte multiple) inside the executable region of your NOP sled. |
| `Illegal instruction` when shellcode executes | Confirm you compiled the harness and exploit on AArch64 and did not accidentally modify the shellcode bytes.  Rebuild `call_shellcode` and rerun Task 2. |
| Docker build fails on Apple silicon | Upgrade Docker Desktop and verify virtualization support (Rosetta/Virtualization.framework) is enabled. |
| ASLR appears active even after disabling | Run `cat /proc/sys/kernel/randomize_va_space` to confirm it is `0`.  Some host OSes may re-enable ASLR on container restart—rerun the disable command each session. |

## 8. License

This project is released under the MIT License.  See [LICENSE](LICENSE) for details.
