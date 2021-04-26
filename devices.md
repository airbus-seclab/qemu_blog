# A deep dive into QEMU: adding devices

In this post, we will see how to create a simple new device. Other
posts will be dedicated to more complex devices such as PCI and
interrupt controllers.

## The QEMU device tree (abbreviated)

The QEMU monitor offers you different commands to inspect devices for a running instance:

```
$ qemu-system-ppc -M 40p -s -S -monitor stdio
QEMU 4.2.0 monitor - type 'help' for more information

(qemu) info qom-tree
/machine (40p-machine)
  /unattached (container)
    /device[15] (pc87312)
      /pc87312[0] (qemu:memory-region)
    /device[0] (604-powerpc-cpu)
    /io[0] (qemu:memory-region)
    /sysbus (System)
    /device[10] (i8257)
      /dma-page[1] (qemu:memory-region)
    /vga.mmio[0] (qemu:memory-region)
    /device[25] (pcnet)
      /pcnet-mmio[0] (qemu:memory-region)
      /pcnet.rom[0] (qemu:memory-region)
    /device[6] (isa-pit)
      /pit[0] (qemu:memory-region)
      /unnamed-gpio-in[0] (irq)
    /device[5] (isa-i8259)
      /unnamed-gpio-in[5] (irq)
      /unnamed-gpio-in[0] (irq)
      /pic[0] (qemu:memory-region)
  /raven (raven-pcihost)
    /pci-conf-idx[0] (qemu:memory-region)
    /pci-conf-data[0] (qemu:memory-region)
    /pci.0 (PCI)
    /unnamed-gpio-in[0] (irq)
    /pci-memory[0] (qemu:memory-region)
```

A lot of things there. From the machine itself, to the CPU objects,
DMA engine, PCI controller, PCI bus, system bus, network (pcnet),
timer (pit) and interrupt controllers (pic).

All of them are QEMU Objects. You can also use the `info qtree`
command to have a more detailled view.

## QEMU Monitor commands

Notice that the monitor commands are implemented through the QMP API
and are referenced as `hmp commands` in the QEMU source code. The
commands subset is specific to a target. When building a PowerPC QEMU,
have a look at `ppc-softmmu/hmp-commands-info.h`. All the available
commands are located at
[`hmp-commands-info.hx`](https://github.com/qemu/qemu/blob/v4.2.0/hmp-commands-info.hx)

They look like the following:

```c
{
.name       = "mtree",
.args_type  = "flatview:-f,dispatch_tree:-d,owner:-o",
.params     = "[-f][-d][-o]",
.help       = "show memory tree (-f: dump flat view for address spaces;"
"-d: dump dispatch tree, valid with -f only);"
"-o: dump region owners/parents",
.cmd        = hmp_info_mtree,
},
```

Where
[`hmp_info_mtree()`](https://github.com/qemu/qemu/blob/v4.2.0/monitor/misc.c#L1076)
is the handler.

## A device is a QObject

As for the machine, we need to create the right
[`TypeInfo`](https://github.com/qemu/qemu/tree/v4.2.0/include/qom/object.h#L426),
[`DeviceClass`](https://github.com/qemu/qemu/tree/v4.2.0/include/hw/qdev-core.h#L37)
and
[`DeviceState`](https://github.com/qemu/qemu/tree/v4.2.0/include/hw/qdev-core.h#L141)
init functions.

Let's implement the minimum code for the CPIOM EDC device. We don't
care about its internals, we just need to know that :
- it has several IO memory mapped registers
- it can raise an interrupt
- it is attached to the system bus

```c
static void cpiom_edc_init(Object *obj)
{
    cpiom_edc_state_t *s = CPIOM_EDC(obj);
    SysBusDevice      *d = SYS_BUS_DEVICE(obj);

    memory_region_init_io(&s->reg1, obj, &edc_reg1_ops, s,
                          CPIOM_EDC_NAME"-reg1", CPIOM_MMAP_EDC_REG_SIZE);
    memory_region_init_io(&s->reg2, obj, &edc_reg2_ops, s,
                          CPIOM_EDC_NAME"-reg2", 6*4);
    memory_region_init_io(&s->err, obj, &edc_err_ops, s,
                          CPIOM_EDC_NAME"-err", CPIOM_MMAP_PPCERR_SIZE);

    sysbus_init_mmio(d, &s->reg1);
    memory_region_add_subregion(get_system_memory(), CPIOM_MMAP_PPCERR, &s->err);

    sysbus_init_irq(d, &s->irq);
}

static void cpiom_edc_class_init(ObjectClass *klass, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);
    dc->desc = CPIOM_EDC_NAME;
}

static const TypeInfo cpiom_edc_info = {
    .name = CPIOM_EDC_NAME,
    .parent = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(cpiom_edc_state_t),
    .instance_init = cpiom_edc_init,
    .class_init = cpiom_edc_class_init,
};

static void cpiom_edc_register_types(void)
{
    type_register_static(&cpiom_edc_info);
}

type_init(cpiom_edc_register_types)
```

Look familiar now. To be QOM compliant, we need a header file for our
device that may have the following content:

```c
#define CPIOM_MMAP_EDC_REG        0x20001000
#define CPIOM_MMAP_EDC_REG_SIZE   0x1000

#define CPIOM_MMAP_PPCERR         0x20002000
#define CPIOM_MMAP_PPCERR_SIZE    0x2000

#define CPIOM_EDC_NAME  "cpiom-edc"
#define CPIOM_EDC(obj)  OBJECT_CHECK(cpiom_edc_state_t,(obj),CPIOM_EDC_NAME)

typedef struct cpiom_edc_state
{
    /*< private >*/
    SysBusDevice     parent_obj;

    /*< public >*/
    MemoryRegion     reg1, reg2, err;
    qemu_irq         irq;

} cpiom_edc_state_t;
```

Everything here is essentialy device init boilerplate. Give name,
specific type, memory addresses ranges ...

No need to explain again that you should implement the associated
`MemoryRegionOps` for each IO memory region.

One important thing is that our EDC device is a
[`SysBusDevice`](https://github.com/qemu/qemu/tree/v4.2.0/include/hw/sysbus.h#L61)
thanks to its `TypeInfo` (`.parent = TYPE_SYS_BUS_DEVICE`). We will be
able to take benefit of the SysBus API.

## Instantiating our device

Back to the machine initialization code:

```c
cpiom_t* cpiom_init_mcs(MachineState *mcs)
{
...
    cpiom_init_dev(cpiom);
}

static void cpiom_init_dev(cpiom_t *cpiom)
{
...
    create_edc(cpiom);
....
}

static void create_edc(cpiom_t *cpiom)
{
    cpiom->edc = sysbus_create_varargs(
        "cpiom-edc", CPIOM_MMAP_EDC_REG,
        qdev_get_gpio_in_named(cpiom->intc, "ITN", INT_N_IT_EDC_ERR),
        NULL);

    cpiom_edc_state_t *s = CPIOM_EDC(cpiom->edc);
    memory_region_add_subregion(cpiom->config, 0xc, &s->reg2);
}
```

We have a specific EDC device init function that does complex
things. In short, it will :
- instantiate our device thanks to the system bus generic service
- attach its IRQ line to our CPIOM interrupt controller
  (`cpiom->intc`, more on this later)

The last memory region `reg2` is attached as a subregion to the CPIOM
board `cpiom->config` memory region. This is a CPIOM specific area
where several device configuration registers are located. Some of our
EDC device registers are thus mapped at `offset 0xc` into this
region. The `config` region itself is an IO memory subregion of the
`system memory`. You can segment and overlap already existing memory
regions with new subregions.


## Creating system bus devices

From the very low level, device creation is done through
[`qdev_create()`](https://github.com/qemu/qemu/blob/v4.2.0/hw/core/qdev.c#L117)
and
[`qdev_init_nofail()`](https://github.com/qemu/qemu/blob/v4.2.0/hw/core/qdev.c#L343). See
the documentation at
[`qdev-core.h`](https://github.com/qemu/qemu/blob/v4.2.0/include/hw/qdev-core.h#L50). They
essentialy find the correct device `TypeInfo` and instantiate the
`DeviceClass` accordingly, leading to the execution of our
`cpiom_edc_class_init/cpiom_edc_init` functions.

The[`sysbus_create_varargs`](https://github.com/qemu/qemu/blob/v4.2.0/hw/core/sysbus.c#L224)
function is a wrapper around the `qdev_xxx()` functions plus automatic
IO mappings and IRQ registration:

```c
DeviceState *sysbus_create_varargs(const char *name, hwaddr addr, ...)
{
    DeviceState *dev;
    SysBusDevice *s;
    va_list va;
    qemu_irq irq;
    int n;

    dev = qdev_create(NULL, name);
    s = SYS_BUS_DEVICE(dev);
    qdev_init_nofail(dev);
    if (addr != (hwaddr)-1) {
        sysbus_mmio_map(s, 0, addr);
    }
    va_start(va, addr);
    n = 0;
    while (1) {
        irq = va_arg(va, qemu_irq);
        if (!irq) {
            break;
        }
        sysbus_connect_irq(s, n, irq);
        n++;
    }
    va_end(va);
    return dev;
}
```

### Mapping device IO memory region

The `addr` argument of `sysbus_create_varargs` is the IO memory region
address `CPIOM_MMAP_EDC_REG` allocated in `cpiom_edc_init`. Remember
we didn't directly attached it as a subregion to the `system memory`
region, but rather did:

```c
static void cpiom_edc_init(Object *obj)
{
...
    memory_region_init_io(&s->reg1, obj, &edc_reg1_ops, ...);
    sysbus_init_mmio(d, &s->reg1);
...
}
```

Every `SysBusDevice` object has an internal `mmio` array of
`QDEV_MAX_MMIO` entries. The
[`sysbus_init_mmio()`](https://github.com/qemu/qemu/blob/v4.2.0/hw/core/sysbus.c#L190)
function will install such an entry. And then `sysbus_create_varargs`
function will call
[`sysbus_mmio_map()`](https://github.com/qemu/qemu/blob/v4.2.0/hw/core/sysbus.c#L130)
which will internally register the given memory region as a subregion
of `system memory`.


### Connecting IRQ lines

The remaining arguments of the function are variable length, NULL
terminated, `qemu_irq`. We will learn more about IRQ in the post on
interrupt controller. Just assume for now that a `qemu_irq` is a GPIO
whose `out` part is connected to the `in` part of another GPIO.

And this is exactly the case for our EDC device. First during EDC device init:

```c
static void cpiom_edc_init(Object *obj)
{
...
    sysbus_init_irq(d, &s->irq);
...
}
```

The
[`sysbus_init_irq()`](https://github.com/qemu/qemu/blob/v4.2.0/hw/core/sysbus.c#L179)
function will register the `out` part of our EDC irq object. Next,
during EDC device creation (`create_edc()`) the argument given to
`sysbus_create_varargs` is:

```c
qdev_get_gpio_in_named(cpiom->intc, "ITN", INT_N_IT_EDC_ERR)
```

Which in short is the `in` part of the `qemu_irq` number
`INT_N_IT_EDC_ERR` that belongs to the `cpiom->intc` device, which
happens to be an interrupt controller (a special device with a lot of
irq lines as you may guess).

The `sysbus_create_varargs` function will
[`sysbus_connect_irq()`](https://github.com/qemu/qemu/blob/v4.2.0/hw/core/sysbus.c#L113)
this irq with our EDC device `qemu_irq out` part. The GPIO connection
code looks like :


```c
qdev_connect_gpio_out_named( DEVICE(dev),
                             SYSBUS_DEVICE_GPIO_IRQ,
			     0,
                             qdev_get_gpio_in_named(cpiom->intc, "ITN", INT_N_IT_EDC_ERR)
			   );
```

Logically, when our device wants to raise an IRQ, it will
[`qemu_set_irq(irq,1)`](https://github.com/qemu/qemu/blob/v4.2.0/hw/core/irq.c#L39)
its own `qemu_irq` object so that the connected `qemu_irq` will
receive the signal.


## Building the device

Do not forget to fix `/hw/cpiom/Makefile.objs` to add our device
object file `edc.o` to be built.
