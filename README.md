# Introduction

This is a series of posts about **QEMU internals**. It won't cover
everything about QEMU, but should help you understand how it works and
foremost how to hack into it for fun and profit.

We won't explain usage and other things that can be found in the
official documentation. The following topics will be addressed:

- [Creating a new machine](machine.md)
- [Controlling memory regions](regions.md)
- [Creating a new device](devices.md)
- [Interrupts controller](interrupts.md)
- [Timers](timers.md)
- [PCI controller](pci.md)
- [PCI devices](pci_slave.md)
- [Options](options.md)
- [Execution loop](exec.md)
- [Breakpoints handling](brk.md)
- [VM running states](runstate.md)
- TCG internals [part 1](tcg_p1.md), [part 2](tcg_p2.md) and [part 3](tcg_p3.md)
- [Snapshots](snapshot.md)

The official code and documentation can be found here:

- https://github.com/qemu/qemu
- https://www.qemu.org/documentation/

# Terminology

## Host and target

The host is the plaform and architecture which QEMU is running
on. Usually an x86 machine.

The target is the architecture which is emulated by QEMU. You can
choose at build time which one you want:

```
./configure --target-list=ppc-softmmu ...
```

As such, in the source code organisation you will find all supported
architectures in the `target/` directory:

```
(qemu-git) ll target
drwxrwxr-x 2 xxx xxx 4.0K alpha
drwxrwxr-x 2 xxx xxx 4.0K arm
drwxrwxr-x 2 xxx xxx 4.0K cris
drwxrwxr-x 2 xxx xxx 4.0K hppa
drwxrwxr-x 3 xxx xxx 4.0K i386
drwxrwxr-x 2 xxx xxx 4.0K lm32
drwxrwxr-x 2 xxx xxx 4.0K m68k
drwxrwxr-x 2 xxx xxx 4.0K microblaze
drwxrwxr-x 2 xxx xxx 4.0K mips
drwxrwxr-x 2 xxx xxx 4.0K moxie
drwxrwxr-x 2 xxx xxx 4.0K nios2
drwxrwxr-x 2 xxx xxx 4.0K openrisc
drwxrwxr-x 3 xxx xxx 4.0K ppc
drwxr-xr-x 3 xxx xxx 4.0K riscv
drwxrwxr-x 2 xxx xxx 4.0K s390x
drwxrwxr-x 2 xxx xxx 4.0K sh4
drwxrwxr-x 2 xxx xxx 4.0K sparc
drwxrwxr-x 2 xxx xxx 4.0K tilegx
drwxrwxr-x 2 xxx xxx 4.0K tricore
drwxrwxr-x 2 xxx xxx 4.0K unicore32
drwxrwxr-x 9 xxx xxx 4.0K xtensa
```

The `qemu-system-<target>` binaries are built into their respective `<target>-softmmu` directory:

```
(qemu-git) ls -ld *-softmmu
drwxr-xr-x  9 xxx xxx 4096 i386-softmmu
drwxrwxr-x 11 xxx xxx 4096 ppc-softmmu
drwxr-xr-x  9 xxx xxx 4096 x86_64-softmmu
```


## System and user modes

QEMU is a system emulator. It offers emulation of a lot of
architectures and can be run on a lot of architectures.

It is able to emulate a full system (cpu, devices, kernel and apps)
through the `qemu-system-<target>` command line tool. This is the mode we
will dive into.

It also provides a *userland* emulation mode through the `qemu-<target>`
command line tool.

This allows to directly run `<target>` architecture Linux binaries on
a Linux host. It mainly emulates `<target>` instructions set and
forward system calls to the host Linux kernel. The emulation is only
related to user level cpu instructions, not system ones, no device
nore low level memory handling.

We won't cover qemu user mode in this blog post series.


## Emulation, JIT and virtualization

Initially QEMU was an emulation engine, with a Just-In-Time compiler
(TCG). The TCG is here to dynamically translate `target` instruction
set architecture (ISA) to `host` ISA.

We will later see that in the context of the TCG, the `tcg-target`
becomes the architecture to which the TCG has to generate final
assembly code to run on (which is host ISA). Obvious !

There exists scenario where `target` and `host` architectures are the
same. This is typically the case in classical virtualization
environment (VMware, VirtualBox, ...) when a user wants to run Windows
on Linux for instance. The terminology is usually Host and Guest
(*target*).

Nowadays, QEMU offers virtualization through different
**accelerators**. Virtualization is considered an accelerator because
it prevents unneeded emulation of instructions when host and target
share the same architecture. Only system level (aka
*supervisor/ring0*) instructions might be emulated/intercepted.

Of course, the QEMU virtualization capabilities are tied to the host
OS and architecture. The x86 architecture offers hardware
virtualization extensions (Intel VMX/AMD SVM). But the host operating
system must allow QEMU to take benefit of them.

Under an x86-64 Linux host, we found the following accelerators:

```
$ qemu-system-x86_64 -accel ?
Possible accelerators: kvm, xen, hax, tcg
```

While on an x86-64 MacOS host:

```
$ qemu-system-x86_64 -accel ?
Possible accelerators: tcg, hax, hvf
```

The supported accelerators can be found in
[`qemu_init_vcpu()`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L2134):

```c
void qemu_init_vcpu(CPUState *cpu)
{
...
    if (kvm_enabled()) {
        qemu_kvm_start_vcpu(cpu);
    } else if (hax_enabled()) {
        qemu_hax_start_vcpu(cpu);
    } else if (hvf_enabled()) {
        qemu_hvf_start_vcpu(cpu);
    } else if (tcg_enabled()) {
        qemu_tcg_init_vcpu(cpu);
    } else if (whpx_enabled()) {
        qemu_whpx_start_vcpu(cpu);
    } else {
        qemu_dummy_start_vcpu(cpu);
    }
...
}
```

To make it short:

- `kvm` is the *Linux Kernel-based Virtual Machine* accelerator;
- `hvf` is the MacOS *Hypervisor.framework* accelerator;
- `hax` is the cross-platform Intel HAXM accelerator;
- `whp` is the *Windows Hypervisor Platform* accelerator.

You can take benefit of the speed of x86 hardware virtualization under
the three major operating systems. Notice that the TCG is also
considered an accelerator. We can enter a long debate about
terminology here ...


## QEMU APIs

There exists a lot of APIs in QEMU, some are obsolete and not well
documented. Reading the source code still remains your best
option. There is a good overview
[available](https://habkost.net/posts/2016/11/incomplete-list-of-qemu-apis.html).

The posts series will mainly address QOM, qdev and VMState. The QOM is
the more abstract one. While QEMU is developped in C language, the
developpers chose to implement the QEMU Object Model to provide a
framework for registering user creatable types and instantiating
objects from those types: device, machine, cpu, ... People used to
[OOP](https://en.wikipedia.org/wiki/Object-oriented_programming)
concepts will find their mark in the QOM.

We will briefly illustrate how to make use of it, but won't detail its
underlying implementation. Stay pragmatic !

The interested reader can have a look at
[include/qom/object.h](https://github.com/qemu/qemu/tree/v4.2.0/include/qom/object.h).

# Disclaimer

It shall be noted that Airbus does not commit itself on the
exhaustiveness and completeness regarding this blog post series. The
information presented here results from the author knowledge and
understandings as of [QEMU
v4.2.0](https://github.com/qemu/qemu/tree/v4.2.0).
