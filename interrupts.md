# A deep dive into QEMU: interrupt controller

In this post, we will see how to create an interrupt controller.


## Yet another SysBusDevice

Now that you understand how to create a SysBusDevice. Let's assume
that our controller is one of those.

It's association to the system bus is done at board level
(`hw/cpiom/board.c`) with :

```c
static void create_intr(cpiom_t *cpiom)
{
    cpiom->intc = sysbus_create_varargs(
        "cpiom-intc", CPIOM_MMAP_INTR_CTL,
        cpiom->cpu->env.irq_inputs[PPC6xx_INPUT_SRESET],
        cpiom->cpu->env.irq_inputs[PPC6xx_INPUT_MCP],
        cpiom->cpu->env.irq_inputs[PPC6xx_INPUT_INT],
        NULL);
}
```

Remember, the `sysbus_create_varargs` function takes a variable number
of `qemu_irq` as arguments. These irqs are the one to be connected
with the device being created.

We are creating an interrupt controller, whose main purpose is to
receive interrupt requests from other devices. But it also has to be
connected to the CPU IRQ pins. That way, the CPU will be able to raise
interrupts to the running operating system kernel and call its
`interrupt service routines` or `vector handlers` whatever you call
them.

In our case, the MPC 755 PowerPC CPU has three available IRQ pins for
:
- System Reset Exception
- Machine check Exception
- External Interrupt Exception

If you are not familiar whith these concepts, please
[RTFM](https://www.nxp.com/docs/en/reference-manual/MPC750UM.pdf) !


## Preparing the receiving IRQ lines

Looking at the device specific code in `/hw/cpiom/intr.c`:

```c
static void cpiom_intc_init(Object *obj)
{
    DeviceState         *dev = DEVICE(obj);
    cpiom_intc_state_t *intc = CPIOM_INTC(obj);
    SysBusDevice        *sbd = SYS_BUS_DEVICE(obj);

    memory_region_init_io(&intc->iomem, obj, &cpiom_intc_io_ops, intc,
                          CPIOM_INTC_NAME, CPIOM_MMAP_INTR_CTL_SIZE);

    sysbus_init_mmio(sbd, &intc->iomem);

    intc->act[INTC_RST].name = "RST";
...
    qdev_init_gpio_in_named_with_opaque(dev, rst_handler, dev,
                                        intc->act[INTC_RST].name, 32);

    intc->act[INTC_MCP].name = "MCP";
...
    qdev_init_gpio_in_named_with_opaque(dev, mcp_handler, dev,
                                        intc->act[INTC_MCP].name, 32);

    intc->act[INTC_ITN].name = "ITN";
...
    qdev_init_gpio_in_named_with_opaque(dev, itn_handler, dev,
                                        intc->act[INTC_ITN].name, 32);

    sysbus_init_irq(sbd, &intc->irq[CPIOM_IRQ_RST]);
    sysbus_init_irq(sbd, &intc->irq[CPIOM_IRQ_MCP]);
    sysbus_init_irq(sbd, &intc->irq[CPIOM_IRQ_ITN]);

    cpiom_intc_reset(dev);
}
```

The CPIOM interrupt controller is a simple device with an IO memory
region and three identical *inner* interrupt controllers, orchestrated
by a global freeze register. Each *inner* interrupt controller has a
simple logic of `enable, mask, priority` registers. We won't enter the
details of its logic, it's out of scope.

The fundamental element in the previous code, is that we are able to
register 32 IRQ lines per interrupt controller, for each of the three
CPU IRQ pins: RST, MCP and ITN. We do so thanks to:

```c
static void itn_handler(void *opaque, int irq, int level);

qdev_init_gpio_in_named_with_opaque(dev, itn_handler, dev, intc->act[INTC_ITN].name, 32);
```

We associate the `itn_handler` callback to the ITN interrupt
controller for its 32 available lines. GPIO `in/out` registration is
string based. And if you remember the previous post, the EDC device
chose to attach its IRQ line to the ITN interrupt controller this way:

```c
qdev_get_gpio_in_named(cpiom->intc, "ITN", INT_N_IT_EDC_ERR),
```

Upon IRQ events (*raised or lowered*) on this controller, the
`itn_handler` will be called with the related IRQ number and level as
argument.

## A bit more about IRQs

If you want to learn more about the internal representation of
[`qemu_irq`](https://github.com/qemu/qemu/blob/v4.2.0/hw/core/irq.c#L31)
in the QEMU code, they are defined as :

```c
typedef struct IRQState *qemu_irq

struct IRQState {
    Object parent_obj;

    qemu_irq_handler handler;
    void *opaque;
    int n;
};
```

It's a simple object with a number, a callback and its avaible generic
argument (usually the *interrupt controller* device whose handler is
being called). And if you look at
[`qdev_init_gpio_in_named_with_opaque`](https://github.com/qemu/qemu/blob/v4.2.0/hw/core/qdev.c#L412):

```c
void qdev_init_gpio_in_named_with_opaque(DeviceState *dev,
                                         qemu_irq_handler handler,
                                         void *opaque,
                                         const char *name, int n)
{
...
    gpio_list->in = qemu_extend_irqs(gpio_list->in, gpio_list->num_in, handler, opaque, n);
...
}

qemu_irq *qemu_extend_irqs(qemu_irq *old, int n_old, qemu_irq_handler handler,
                           void *opaque, int n)
{
    qemu_irq *s;
    int i;
...
    for (i = n_old; i < n + n_old; i++) {
        s[i] = qemu_allocate_irq(handler, opaque, i);
    }
...
}

qemu_irq qemu_allocate_irq(qemu_irq_handler handler, void *opaque, int n)
{
    struct IRQState *irq;

    irq = IRQ(object_new(TYPE_IRQ));
    irq->handler = handler;
    irq->opaque = opaque;
    irq->n = n;

    return irq;
}

```

And whenever an IRQ is raised or lowered:

```c
static inline void qemu_irq_raise(qemu_irq irq) { qemu_set_irq(irq, 1); }
static inline void qemu_irq_lower(qemu_irq irq) { qemu_set_irq(irq, 0); }

void qemu_set_irq(qemu_irq irq, int level)
{
    if (!irq)
        return;

    irq->handler(irq->opaque, irq->n, level);
}
```
