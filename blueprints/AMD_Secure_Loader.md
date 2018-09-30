AMD Secure Loader
=================

## Summary

This blueprint describes the design of an AMD Secure Loader that is capable of
being launched with the SKINIT instruction and an associated hand-off protocol.

## Background

AMD Secure Startup SKINIT instruction expects a Secure Loader (SL) image as
described in the AMD64 Architecture Programmer's Manual, Volume 2. There are
several requirements that an SL image must comply, one in particular is the
size restriction of 64 KBytes. When considering the complex actions that must be
taken to securely hand-off to a more advance kernel, e.g. enabling memory
protections, hashing memory segments, etc., the 64 KBytes is quickly
deplenished. This project will develop a general purpose SL that implements a
hand-off protocol that target runtime's may adopt to enable being securely
launched on an AMD platform.

## Approach

The development will start with a prototype phase that is specifically designed
to boot a Linux kernel will be used to
demonstrate the solution working. Lessons learned from the prototype phase will
be used to refactor the SL into a more generic solution.

### Prototype

The loading assumption is that the boot loader will load the Linux kernel in
its preferred location. The boot loader will then load the SL at the first
immediate 4K boundary following the bzImage plus a 0x01000 offset. The boot
loader will reserve the SL's 64K memory range in the memory map. An
approximation of the memory layout can be seen below.

Memory Layout
```
 +---------------+ -0x01000 - PAGE_UP(bzImage)
 |               |
 |   bzImage     |
 |               |
 +---------------+ -0x01000
 |   Second      |
 |   Stage       |
 |   Stack       |
 +---------------+  0x00000
 |   SL Header   |
 | ------------- |  0x00004
 |   LZ Header   |
 | ------------- |  0x00020
 |   First       |
 |   Stage       |
 |   Stack       |
 | ------------- |  0x00200
 |               |
 |   LZ Code     |
 |               |
 | ------------- |  0x0A000
 |     DEV       |
 |   Tables      |
 | ------------- |  0x0D000
 |    Page       |
 |   Tables      |
 +---------------+  0x10000
```

#### Launch Sequence

The launch sequence for the SL is broken up in three stages, entry, setup,
secure hand-off.

The entry is assembly entry code and is responsible for,
* direct map paging for the first 4GB of memory
* adjust stack pointer to First Stage Stack
* switching to long mode
* jumping into setup

The setup stage is C code that is responsible for,
* setup DEV table for the first 48MB of memory
* DEV protecting the baImage and Second Stage Stack
* switching to Second Stage Stack
* calling the secure hand-off function

The secure hand-off is C code that is responsible for,
* hashing the bzImage configuration (zero page)
* hashing the bzImage command line
* hashing the bzImage from code32 entry point to image end
* returning to 32 bit protected mode
* jumping into bzImage code32 entry point
