# Demo introduction to CheriBSD development

## Intro to CHERI development

Cheribuild cheatsheet
```
# Build cheribsd for a target
cbuild cheribsd-morello-purecap

# Build a custom kernel configuration (as the default kernel)
cbuild cheribsd-morello-purecap --cheribsd-morello-purecap/kernel-config GENERIC-MORELLO-PURECAP-NOSUBOBJECT

# Build only the kernel, not the base system (world)
cbuild cheribsd-morello-purecap --skip-buildword

# Build the disk image (required after updating the kernel or the base system)
# Note that this pull in files placed in <srcroot>/extra-files
# as well as anything you place in the <cherisdk>/rootfs-morello-purecap (but it will ask)
cbuild disk-image-morello-purecap

# Run on qemu
cbuild run-morello-purecap
```

On the morello box, you can clone directly the cheribsd repos and use standard BSD make commands
to build the kernel.
The [FreeBSD manual](https://docs.freebsd.org/en/books/developers-handbook/kernelbuild/) is a good resource.

These days we have bhyve support, so kernel development is much less painful. See `man 8 bhyve`.
 - See [vmrun](https://github.com/CTSRD-CHERI/cheribsd/blob/dev/share/examples/bhyve/vmrun.sh) example script to run bhyve
   - This should be available in `/usr/share/examples/bhyve/vmrun.sh` on a CheriBSD morello installation.
 - It is possible to attach gdb to bhyve guests for kernel debugging (same as QEMU).

## Intro to CheriBSD codebase

 1. Source code organisation overview (not sure how familiar are people)
    - Mostly focused on sys/
    - sys/sys -- MI includes
    - sys/compat -- compat layers for freebsd64 (and the upcoming linux PCuABI)
    - sys/kernel -- base kernel
      - notably kmalloc and vmem allocators
      - imgact_elf is the elf loader that is responsible of loading executables on `exec()`
    - sys/vm -- MI virtual memory
      - UMA allocator
      - vm_page -- resident pages management
      - vm_phys -- physical memory management
      - vm_map -- user and kernel mappings
      - vm_object -- abstraction for objects backing VM pages (e.g. files or anonymous memory)
      - vm_mmap -- mmap system call implementation and related functions
      - vm_kern -- low-level kernel memory allocation interface (backed by other allocators)
      - vm\_cheri\_revoke -- revocation implementation
    - sys/<arch>/pmap.c -- low level page table management
 2. Where does CHERI go in all of this?
    - Changes are spread to many places, can not be exhaustive here.
    - Generally all allocators have some changes to enforce bounds.
    - Machine-dependent code in sys/<arch> has changes to assembly files and relocation code.

## Life of a pointer

