# A deep dive into QEMU: a Brief History of Time

Ever wanted to play with general relativity ? QEMU is a simulation
environment, guess what ? We can control time as seen by the VM !

Some architectures directly provide a clock register inside the
CPU. However, a board usually needs extended time control through
dedicated devices. How would you implement such a device inside QEMU ?

## Time in QEMU

### QEMU Clocks

QEMU implements several `clocks` to get informed about time. Obviously
you can still directly use host OS interface to get time information.

Looking at
[`timer.h`](https://github.com/qemu/qemu/blob/v4.2.0/include/qemu/timer.h#L16)
we learn that there exists 4 clock types:
- realtime
- host
- virtual
- virtual_rt

The one that will be of interest for simulating basic timers is
`virtual`. It only runs alongside the VM, so it reflects time reality
in the context of the VM.

QEMU provides a `qemu_clock_xxx` API to control time for related clocks.

```c
int64_t now = qemu_clock_get_ms(QEMU_CLOCK_VIRTUAL);
```

This returns the current time in millisecond for the `virtual`
clock.

### QEMU Timers

QEMU provides a `timer_xxx` API to create, modify, reset, delete
`timers`, for different clocks and granularity (ms, ns). You can
attach timers to specific clocks. The main QEMU execution loop
controls the `virtual` clock and can disable timers when the VM vCPU
is stopped.

The following piece of code creates a `timer` with milliseconds
granularity, that runs only when the VM vCPU runs:

```c
QEMUTimer *user_timer = timer_new_ms(QEMU_CLOCK_VIRTUAL, user_timeout_cb, obj);
int64_t    now        = qemu_clock_get_ms(QEMU_CLOCK_VIRTUAL);

timer_mod(timer, now + duration);

static void user_timeout_cb(void *opaque)
{
  obj_t *obj = (obj_t*)opaque;
...
}
```

When `duration` milliseconds have elapsed in the `virtual clock` time,
the callback function `user_timeout_cb` is called.


## Creating a timer device

As any other device, and following the datasheet of the timer you
would like to simulate, you will have to expose IO memory regions to
reflect device register configuration to QEMU timers setup and raise
IRQs on those timers expiration.

So you will need both device specific hardware representation and QEMU
internal clock model.

### CPIOM tick timer

Under our CPIOM example implementation this may look like the following:

```c
typedef struct cpiom_clock
{
    QEMUTimer *qemu_timer;
    uint32_t  *trigger;
    int64_t    restart;
    double     duration;

} cpiom_clock_t;

typedef struct cpiom_timer_device_state
{
    /*< private >*/
    SysBusDevice      parent_obj;

    /*< public >*/
    MemoryRegion      iomem;
    cpiom_timer_reg_t reg;
    qemu_irq          irq;

    /* internal clock management */
    cpiom_clock_t     tick;

} cpiom_timer_state_t;
```

We have a standard `SysBusDevice` with `iomem` IO memory region and
underlying device registers `reg`. It also declares a `cpiom_clock`
called `tick`. The real CPIOM timers are actually more complex, but
for the sake of simplicity here we will only consider a `tick` timer.

```c
static void cpiom_timer_init(Object *obj)
{
    cpiom_timer_state_t *tm  = CPIOM_TIMERS(obj);
    SysBusDevice        *dev = SYS_BUS_DEVICE(obj);

    memory_region_init_io(&tm->iomem, obj, &cpiom_timer_reg_ops, tm,
                          CPIOM_TIMERS_NAME"-reg", CPIOM_MMAP_TIMERS_SIZE);
    sysbus_init_mmio(dev, tm->iomem);
    sysbus_init_irq(dev, &tm->irq);

    tm->tick.qemu_timer = timer_new_ns(QEMU_CLOCK_VIRTUAL, tick_expired, tm);
    tm->tick.trigger    = &tm->reg.base.tick;
...
}
```

We actually setup a device whose any access to `tm->iomem` will update
`tm->reg` thanks to the `cpiom_timer_reg_ops`
[`MemoryRegionOps`](https://github.com/qemu/qemu/blob/v4.2.0/include/exec/memory.h#L144). In
the meantime, a nano second `virtual clock` timer is created to call
`tick_expired`.


### Accessing the CPIOM tick timer

Let's say offset `0x0c` is a `R/W 32 bits TIME_COUNTER` register for
our imaginary timer device. The counter is decremented at a given
frequency (usually adjustable via a scale register). When it reaches
0, it raises an IRQ.

Eventually an OS driver running on our CPIOM board, and trying to
setup the timer device, will happen to write to this register.

A candidate implementation would be:

```c
static const MemoryRegionOps cpiom_timer_reg_ops = {
    .read  = cpiom_timer_reg_read,
    .write = cpiom_timer_reg_write,
    .endianness = DEVICE_NATIVE_ENDIAN,
};

static void cpiom_timer_reg_write(void *opaque, hwaddr addr, uint64_t data, unsigned size)
{
....
    cpiom_timer_state_t *tm = (cpiom_timer_state_t*)opaque;

    if (addr == 0x0c)
        write_counter(tm, data);
....
}

static void write_counter(cpiom_timer_state_t *tm, uint32_t new)
{
    if (!timer_is_active(tm))
        return;

    if (new == 0)
        tick_expired((void*)tm);
    else
        clock_setup(tm, &tm->tick, new);
}

static void tick_expired(void *opaque)
{
    cpiom_timer_state_t *tm = (cpiom_timer_state_t*)opaque;
    qemu_irq_raise(tm->irq);
}
```

If the driver modifies the device `counter`, we should check for
possible immediate expiration and raise an IRQ. Else we must update
our QEMU internal timer to trigger a call to `tick_expired` at the
expected `virtual` clock time.


### Time dilatation

Interestingly, the `clock_setup` might look like:

```c
static void clock_setup(cpiom_timer_state_t *tm, cpiom_clock_t *clk, uint32_t count)
{
    clk->duration = nsperiod * count;
    clk->restart  = qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL);

    uint64_t expire = clk->restart + (int64_t)floor(clk->duration);
    timer_mod(clk->qemu_timer, expire /* +/- speed factor */);
}
```

We compute the next expiration date in nano seconds based on the new
counter value and the timer frequency (expressed as `nsperiod`). This
period might be computed as follows:

```c
    nsperiod = (1/TIMER_FREQ_MHZ) * 1000 * scale;
```

Notice that we can also induce a *speed factor* effect to the
`virtual` clock.


### Elapsed time

Conversely, whenever a driver reads the device `counter` register,
your code must reflect the elapsed time in the VM and give back an
appropriate value. Something like:

```c
    now          = qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL);
    count        = (now - clk->restart)/nsperiod;
    clk->restart = now;
```
