# Demo introduction to CHERI-QEMU development

## Status update

 - Completed merge to 6.2
 - Tracing infrastructure is there but will be updated
   - Need to write documentation for others to use (CHERI wiki?)

## Intro to CHERI development

This is already mentioned in the getting started guides, but worth repeating.
Most CHERI applications can be built using the `cheribuild` tool found in the
[repo](https://github.com/CTSRD-CHERI/cheribuild).

Also see cheribsd [getting started](https://www.cheribsd.org/)

When running, `cheribuild` will clone repositories and can build dependencies if required.
You can place the `cheribuild.py` tool in your PATH or use a bash `alias`, I do the latter.

Cheribuild cheatsheet
```
# List all available targets
cbuild --list-targets

# Do not `git pull` for updates
cbuild --skip-update <target>

# Clean build or re-run configuration step (e.g. ./configure in autotools)
cbuild --reconfigure --clean <target>

# Build LLVM for CHERIv9/RISC-V and the Morello LLVM respectively
# (at some point things will be merged)
cbuild llvm
cbuild morello-llvm

# Build QEMU
cbuild qemu
```

Note that `cheribuild` can be customised using a config file in ~/.config/cheribuild.json
for more advanced setups.

## Tour of CHERI QEMU
In the most reductive sense, QEMU is built around a loop of code translation and execution.
When new guest code needs to be executed, QEMU first translates it to host machine code via the TCG (Tiny Code Generator) subsystem,
and subsequently transfers control to the translated host machine code.
This is repeated for every TB (Translation Block), roughly corresponding to a BB (Basic Block) in the guest code.
The translation and execution loop is periodically interrupted to service I/O and interrupts.

### Running CheriBSD on QEMU
When QEMU is built with `cheribuild`, the binaries will be placed in `<sdkroot>/cherisdk/sdk/bin/qemu-system-<platform>`.
As a shortcut, `cheribuild` can run CheriBSD as follows:

```
# Build CheriBSD
cheribuild cheribsd-morello-purecap
cheribuild disk-image-morello-purecap

# Run CheriBSD in QEMU
cheribuild run-morello-purecap
```

### CHERI QEMU components
The CHERI-enabled QEMU implements CHERI support with a common CHERI target library and per-target implementations.
The common CHERI target support "library" is found at `target/cheri-common`; this includes

 - Generic instructon implementations (e.g. setbounds).
 - Tag memory.
 - Capability encoding library and helpers to manipulate capability fields from target-specific code.
 - TCG helpers for common operations(e.g. get capability tag).

Each CHERI-enabled target, implements its own decoding logic for instructions and generally contains:

 1. CHERI-specific instructions (e.g. setbounds).
 2. Legacy ISA instructions with a modified behaviour (e.g. load and stores of integer registers).
 3. Unchanged Legacy ISA instructions.

The decoder must also be aware of the program counter capability (PCC) and ensure that bounds checks occur.

Once an instruction is decoded, TCG translation occurs. The target will implement CHERI instructions with a mix of
TCG instructions and TCG helpers (C functions that are called during the execution phase to implement some instruction logic).
This code generation is generally split between the *cheri-common* target and the target-specific helpers logic.
Various other instructions are modified to mediate load and stores with capabilities and manage tags in the register file.

When a TB is fully translated, it can be executed. This mechansim is unchanged from the normal QEMU execution flow.


## Life of an Instruction

As an approachable way to introduce to QEMU, I want to show the CHERI implementation focusing on a single (simple)
ARM Morello instruction.
Relevant files in the target implementation:

 - cpu.h -- General CPU state description and utility functions.
 - cheri.decode -- contains bit patterns for instruction decoding.
   These files are parsed by the `scripts/decodetree.py` tool and documented in `docs/devel/decodetree.rst`
 - translate-a64.c -- top-level aarch64 instruction translation phase, contains the main translation logic.
 - translate-cheri.c.inc -- contains actual CHERI instruction TCG translations.
   Transforms a single instruction into TCG ops that will run on the host. This file is included into the top-level translation source.
 - helper.c -- General CPU helpers (e.g. `arm_cpu_do_interrupt()`)

For this example, we assume that we are in the translation loop, at point in which QEMU finds a new instruction stream
that needs to be translated.
The translation loop will invoke the target-specific translation logic for each guest instruction to be translated via the TCG.

For the sake of simplicity we focus on the ARM Morello instruction to set capability bounds: `scbnds`.

Condisder the following instruction:
```
# From llvm-objdump -d path/to/kernel.full
ffff0000000020a4: ef 01 ce c2   scbnds  c15, c15, x14
```

Note that the opcode above corresponds to:
```
31                  15                0
c    2    c    e    0    1    e    f
1100 0010 1100 1110 0000 0001 1110 1111
```

See `aarch64_tr_translate_insn()` in `target/arm/translate-a64.c`
```
    // disas_a64_insn()
    // Load the opcode from host memory
    uint32_t insn = arm_ldl_code(env, s->base.pc_next, s->sctlr_b);

    // Switch on bits [28:25] of the instruction
    switch (extract32(insn, 25, 4)) {
    ...
    case 0x1: /* Morello Instructions */
#ifdef TARGET_CHERI
        // Calls cheri-specific translation entrypoint generated by decodetree (cheri.decode)
        if (!disas_cheri(s, insn)) {
            unallocated_encoding(s);
        }
        break;
#endif
    ...
    }
```

The generated decoder will match the setbounds instruction.
```
// Note that there are two scbnds bit patterns, the first one is for the immediate variant
@op_SCBNDS    ........... imm6:6  S:1 .... Cn:5  Cd:5
@op_SC    ........... Rm:5 . opc:2 ... Cn:5  Cd:5

// We will match the second pattern specified as
SC 11000010110.....0..000.......... @op_SC

pattern: 11000010110.....0..000..........
instr:   11000010110011100000000111101111
```

As a result we will call the translation function for SC.
See `TRANS_F(SC)` in `target/arm/translate-cheri.c.inc`
```
// The translation functions are passed an arguments structure determined by the decodetree @op_xxx specification

    // in the function we check whether the SC-type instruction is a setbounds and generate TCG code accordingly.
    if (!(a->opc & 2)) {
        ...
        helper = &gen_helper_csetbounds;
        ...
        return gen_cheri_cap_cap_int(ctx, a->Cd, a->Cn, a->Rm, helper);
    }

```

The `gen_helper_csetbounds` will insert a sequence of instructions that will call the corresponding C
function to handle the execution of the instruction.
See `csetbounds` helper in `target/cheri-common/op_helper_cheri_common.c`, note that these functions use macros for declarations.
```
void CHERI_HELPER_IMPL(csetbounds(CPUArchState *env, uint32_t cd, uint32_t cb,
                                  target_ulong rt))
{
    // call into generic implementation, which will not get into
    do_setbounds(false, env, cd, cb, rt, GETPC());
}
```

Once the translation loop finds a function that signals the end of the TB (e.g. a branch), the TB is executed by jumping into it.
