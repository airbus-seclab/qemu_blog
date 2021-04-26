# A deep dive into QEMU: memory regions

In this post we'll have a glance at high level memory organisation in
QEMU: memory regions (MR).

We won't cover address spaces, because we usually manage memory
regions directly. However, have a look at
[docs/devel/memory](https://github.com/qemu/qemu/tree/v4.2.0/docs/devel/memory.rst)
and the code
([include/exec/memory.h](https://github.com/qemu/qemu/blob/v4.2.0/include/exec/memory.h)) for more details.

An high level external presentation of memory organisation is
available
[there](http://blog.vmsplice.net/2016/01/qemu-internals-how-guest-physical-ram.html). You
will also find a very interesting internal documentation at
[docs/devel/loads-stores](https://github.com/qemu/qemu/blob/v4.2.0/docs/devel/loads-stores.rst). This
is an enumeration of the available QEMU APIs for accessing memory.

When you want to play with memory regions in QEMU, you can either:

- get a direct pointer to the host buffer backing your VM memory
  region
- implement read/write callback functions to intercept every access (usually IO
  memory)
- use QEMU
  [`cpu_physical_memory_rw()`](https://github.com/qemu/qemu/tree/v4.2.0/exec.c#L3275) to safely access the region

In the blog post dedicated to the TCG, we will exactly see how
translated instructions access VM memory and how we can intercept at
this level.


## Looking at the memory tree (abbreviated)

Below is the tree of available memory regions once a `PowerPC 40p`
board is ready. As you can see, memory regions can contain other
memory regions (called `subregions`). This a clean way to organize
memory. Each memory region has its own properties and is attached to a
kind of view called the `address space`.

```
$ qemu-system-ppc -M 40p -s -S -monitor stdio
QEMU 4.2.0 monitor - type 'help' for more information

(qemu) info mtree

address-space: memory
  0000000000000000-ffffffffffffffff (prio 0, i/o): system
[...]
    0000000080000000-00000000807fffff (prio 1, i/o): pci-io-non-contiguous
    0000000080000000-00000000bf7fffff (prio 0, i/o): pci-io
      0000000080000000-0000000080000007 (prio 0, i/o): dma-chan
      0000000080000008-000000008000000f (prio 0, i/o): dma-cont
      0000000080000020-0000000080000021 (prio 0, i/o): pic
      0000000080000040-0000000080000043 (prio 0, i/o): pit
      0000000080000061-0000000080000061 (prio 0, i/o): pcspk
      0000000080000064-0000000080000064 (prio 0, i/o): i8042-cmd
      00000000800002f8-00000000800002ff (prio 0, i/o): serial
[...]
      0000000080000cf8-0000000080000cfb (prio 0, i/o): pci-conf-idx
      0000000080000cfc-0000000080000cff (prio 0, i/o): pci-conf-data
    0000000080800000-0000000080bfffff (prio 0, i/o): pciio
    00000000c0000000-00000000feffffff (prio 0, i/o): pci-memory
      00000000c00a0000-00000000c00bffff (prio 1, i/o): vga-lowmem
    00000000fff00000-00000000ffffffff (prio 0, rom): bios


address-space: cpu-memory-0
  0000000000000000-ffffffffffffffff (prio 0, i/o): system
[...]
    00000000fff00000-00000000ffffffff (prio 0, rom): bios


memory-region: pci-memory
  00000000c0000000-00000000feffffff (prio 0, i/o): pci-memory
    00000000c00a0000-00000000c00bffff (prio 1, i/o): vga-lowmem


memory-region: system
  0000000000000000-ffffffffffffffff (prio 0, i/o): system
[...]
    00000000fff00000-00000000ffffffff (prio 0, rom): bios

```

Default memory regions and address spaces are created by QEMU. The
most important is the `system memory region` which is created by
[`memory_map_init()`](https://github.com/qemu/qemu/blob/v4.2.0/exec.c#L2957)
from
[`cpu_exec_init_all()`](https://github.com/qemu/qemu/blob/v4.2.0/exec.c#L3402).

It can be seen as the top level one, and usually `subregions` are
added to the `system memory region`.


## Allocating system memory

This might be one of the most desired things when creating a new
machine : get RAM and load a firmware. The correct function to invoke
is
[`memory_region_allocate_system_memory()`](https://github.com/qemu/qemu/blob/v4.2.0/include/hw/boards.h#L14).

If we look at some other board implementations, for instance the
[`MIPS
R4K`](https://github.com/qemu/qemu/blob/v4.2.0/hw/mips/mips_r4k.c#L167):

```c
void mips_r4k_init(MachineState *machine)
{
    MemoryRegion *sys_mem = get_system_memory();
    MemoryRegion *ram = g_new(MemoryRegion, 1);

    memory_region_allocate_system_memory(ram, NULL, "mips_r4k.ram", ram_size);
    memory_region_add_subregion(sys_mem, 0, ram);
...
}
```

A new memory region for the RAM is created and directly added as a
`subregion` of the `system memory region`. From that point, accessing
physical addresses `0` to `ram_size` will hit the VM memory.

The `memory_region_allocate_system_memory` function actually calls
[memory_region_init_ram_shared_nomigrate()](https://github.com/qemu/qemu/blob/v4.2.0/memory.c#L1512)
from the QEMU memory API. It encapsulates RAM specific memory region
initialization.

The QEMU memory API allows you to create memory regions backed by file
descriptors, already allocated host buffers and callbacks as we will
see for IOs.

## IO memory regions

Getting back to our simple MIPS board example:

```c
void mips_r4k_init(MachineState *machine)
{
...
    MemoryRegion *iomem = g_new(MemoryRegion, 1);

    memory_region_init_io(iomem, NULL, &mips_qemu_ops, NULL, "mips-qemu", 0x10000);
    memory_region_add_subregion(sys_mem, 0x1fbf0000, iomem);
}
```

A new memory region `iomem` is created with
[`memory_region_init_io()`](https://github.com/qemu/qemu/blob/v4.2.0/memory.c#L1490)
and also added as a `subregion` of the `system memory`. This region is
not of RAM but IO type and has a special
[`MemoryRegionOps`](https://github.com/qemu/qemu/blob/v4.2.0/include/exec/memory.h#L144)
argument.

```c
static const MemoryRegionOps mips_qemu_ops = {
    .read = mips_qemu_read,
    .write = mips_qemu_write,
    .endianness = DEVICE_NATIVE_ENDIAN,
};

static void mips_qemu_write (void *opaque, hwaddr addr,
                             uint64_t val, unsigned size)
{
    if ((addr & 0xffff) == 0 && val == 42)
        qemu_system_reset_request(SHUTDOWN_CAUSE_GUEST_RESET);
...
}

static uint64_t mips_qemu_read (void *opaque, hwaddr addr,
                                unsigned size)
{
    return 0;
}
```

IO memory regions expose devices memory. They usually need special
interpretation during read/write accesses to simulate the expected
device behavior. Using `MemoryRegionOps` callback helps you implement
device operations.

In the previous example, the `iomem` region is mapped at `0x1fbf0000 -
0x1fc00000`. Whenever the VM accesses this memory range, the
read/write callbacks will be called. The `addr` argument is an offset
from the beginning of the related memory region.

So doing something like `*(u8*)0x1fbf0000 = 42` on this MIPS board
triggers a system reset.


## What about the CPIOM SDRAM

Sometimes you may like to combine both for a specific device. Part of
the device memory has no special meaning and is used for transactions,
while a small range might be interpreted as device registers.

```c
static void cpiom_sdram_init(Object *obj)
{
    cpiom_sdram_state_t *s = CPIOM_SDRAM(obj);
    SysBusDevice        *d = SYS_BUS_DEVICE(obj);
    MemoryRegion        *sysmem = get_system_memory();

    memory_region_allocate_system_memory(&s->ram, obj,
                                         CPIOM_SDRAM_NAME,
                                         CPIOM_MMAP_SDRAM_SIZE);
    memory_region_init_io(&s->reg, obj, &sdram_reg_ops, s,
                          CPIOM_SDRAM_NAME"-reg",
                          CPIOM_MMAP_SDRAM_REG_SIZE);
    memory_region_init_io(&s->pcode.mmio, obj, &sdram_pcode_ops, s,
                          CPIOM_SDRAM_NAME"-pcode",
                          CPIOM_MMAP_SDRAM_PCODE_SIZE);
    memory_region_init_io(&s->pdata.mmio, obj, &sdram_pdata_ops, s,
                          CPIOM_SDRAM_NAME"-pdata",
                          CPIOM_MMAP_SDRAM_PDATA_SIZE);
    memory_region_init_io(&s->raf, obj, &sdram_raf_ops, s,
                          CPIOM_SDRAM_NAME"-raf", 1*4);

    memory_region_add_subregion(sysmem, 0, &s->ram);
    sysbus_init_mmio(d, &s->reg);
    memory_region_add_subregion(sysmem,
                                CPIOM_MMAP_SDRAM_PCODE, &s->pcode.mmio);
    memory_region_add_subregion(sysmem,
                                CPIOM_MMAP_SDRAM_PDATA, &s->pdata.mmio);

    sysbus_init_irq(d, &s->irq_viol);
    sysbus_init_irq(d, &s->irq_err);
}
```

In this more complex example, the CPIOM SDRAM is a fully fledged QEMU
device object with several memory regions. Some memory regions (`reg`
and `raf`) are not directly added as subregions to the system
memory. We will explain why in the next post dedicated to device
creation.

The CPIOM SDRAM controller offers protection for code and data, as
well as refresh rate tunning. It is also able to raise interrupt
requests.
