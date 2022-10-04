# A deep dive into QEMU: a new machine

In this post, we will see how to create a new machine. Opportunity is
given to introduce the so called QOM.

## Available Machines and CPUs

The available machines for a given architecture can be listed with the
`-M ?` command line option.

For the PowerPC target architecture we have:

```
$ qemu-system-ppc -M ?
Supported machines are:
40p                  IBM RS/6000 7020 (40p)
bamboo               bamboo
g3beige              Heathrow based PowerMAC (default)
mac99                Mac99 based PowerMAC
mpc8544ds            mpc8544ds
none                 empty machine
ppce500              generic paravirt e500 platform
prep                 PowerPC PREP platform (deprecated)
ref405ep             ref405ep
sam460ex             aCube Sam460ex
taihu                taihu
virtex-ml507         Xilinx Virtex ML507 reference design
```

Once you picked-up your favorite machine, you can also choose between
available CPUs for the related machine.

```
$ qemu-system-ppc -M 40p -cpu ?
PowerPC 601_v0           PVR 00010001
PowerPC 601_v1           PVR 00010001
...
PowerPC 755_v2.8         PVR 00083208
PowerPC 755              (alias for 755_v2.8)
PowerPC goldfinger       (alias for 755_v2.8)
...
```

## The CPIOM machine

Let's create a new machine to support
[CPIOM](http://www.artist-embedded.org/docs/Events/2007/IMA/Slides/ARTIST2_IMA_Itier.pdf)
in QEMU.

In Integrated Modular Avionics (IMA), the Core Processing Input Output
Modules (CPIOMs) are PowerPC MPC-755 CPU based embeded equipements
with RAM, Flash, NVM, Timers, Interrupt controller, DMA engine, PCI
controller and others components we won't cover (AFDX, DSP).

![CPIOM architecture](./docs/assets/images/34A45ACD-6EBF-4C3F-93E4-DF4EE3009469.png)

## Code organisation

First let's prepare our environment by isolating our cpiom code in a
dedicated directory. Usually, every single object you add to QEMU
should be placed in the right directory. They are organized by nature,
not target usage or association.

- a new PowerPC based machine should land in `hw/ppc/`
- a new serial device in `hw/char/`
- a new PCI host controller in `hw/pci-host/`
- a new network controller in `hw/net/`
- ...

So let's do the opposite and put everything at the same place in
`hw/cpiom`. The first thing we have to do is tell QEMU how to build
our new machine and futur devices:

```
$ cat hw/cpiom/Makefile.objs
obj-y += board.o dev1.o dev2.o

$ cat hw/Makefile.objs
...
devices-dirs-y += cpiom/
```

## Creating a new machine: `hw/cpiom/board.c`

To implement a new machine in QEMU we will need to create QOM
[`TypeInfo`](https://github.com/qemu/qemu/tree/v4.2.0/include/qom/object.h#L426)
and its associated
[`MachineClass`](https://github.com/qemu/qemu/tree/v4.2.0/include/hw/boards.h#L107)
and
[`MachineState`](https://github.com/qemu/qemu/tree/v4.2.0/include/hw/boards.h#268)
initialization functions.

You can find documentation about [QOM
conventions](https://wiki.qemu.org/Documentation/QOMConventions).

### The TypeInfo

We can use the predefined macro:

```c
DEFINE_MACHINE("cpiom", cpiom_machine_init)
```

Which generates the following code:

```c
static void cpiom_machine_init_class_init(ObjectClass *oc, void *data)
{
    MachineClass *mc = MACHINE_CLASS(oc);
    cpiom_machine_init(mc);
}

static const TypeInfo cpiom_machine_init_typeinfo = {
    .name       = MACHINE_TYPE_NAME("cpiom"),
    .parent     = TYPE_MACHINE,
    .class_init = cpiom_machine_init_class_init,
};

static void cpiom_machine_init_register_types(void)
{
    type_register_static(&cpiom_machine_init_typeinfo);
}
type_init(cpiom_machine_init_register_types)
```

This will register a new machine class type for our CPIOM. This is the
highest level definition.

### The MachineClass

As the API states, our `class_init` function will be called after all
parent class init methods have been called. The class is initialized
before any object instance of the class.

Let's write it :

```c
void cpiom_machine_init(MachineClass *mc)
{
    mc->desc = "CPIOM board";
    mc->init = cpiom_init;
    mc->default_cpu_type = POWERPC_CPU_TYPE_NAME("755_v2.8");
    mc->default_ram_size = CPIOM_MMAP_SDRAM_SIZE;
}
```

At this point we are able to define specific properties of the
machine, such as the CPU model, the RAM size, and most importantly the
instance initialization method `cpiom_init`.


### The MachineState

Once the class is ready, an object instance will be created. We
previously provided an `mc->init = cpiom_init` function.

```c
static void cpiom_init(MachineState *mcs)
{
    cpiom_t *cpiom = g_new0(cpiom_t, 1);

    cpiom_init_cpu(cpiom, mcs);
    cpiom_init_dev(cpiom);
    cpiom_init_boot(cpiom, mcs);
}
```
The core features of our board implementation will lie here.


## Initializing a CPU for the CPIOM: PowerPC 755

The cpu instantiation will look like the following:

```c
static void cpiom_init_cpu(cpiom_t *cpiom, MachineState *mcs)
{
    cpiom->cpu = POWERPC_CPU(cpu_create(mcs->cpu_type));

    /* Set time-base frequency to 16.6 Mhz */
    cpu_ppc_tb_init(&cpiom->cpu->env, CPIOM_TBFREQ);

    cpiom->cpu->env.spr[SPR_HID2] = 0x00040000;

    CPUClass *cc = CPU_GET_CLASS( CPU(cpiom->cpu) );
    PowerPCCPUClass *pcc = POWERPC_CPU_CLASS(cc);
    CPUPPCState *env = &cpiom->cpu->env;

    /* Default mmu_model for MPC755 is POWERPC_MMU_SOFT_6xx (no softmmu) */
    pcc->mmu_model = POWERPC_MMU_32B;
    env->mmu_model = POWERPC_MMU_32B;
    pcc->handle_mmu_fault = ppc_hash32_handle_mmu_fault;
}
```

### Obtaining a PowerPC CPU state object

The generic
[`cpu_create()`](https://github.com/qemu/qemu/tree/v4.2.0/hw/core/cpu.c#L58)
function will create a CPU object of the proper type. We want to
instantiate a PowerPC CPU whose specific type has been defined during
the class init (`POWERPC_CPU_TYPE_NAME("755_v2.8")`).

The code is equivalent to:

```c
CPUState *cpu = CPU( object_new( POWERPC_CPU_TYPE_NAME("755_v2.8") ));
```

The QOM type instantiation is complex and not that much
interesting. We should just remember that we want a PowerPC cpu from
the 755 family. The requested CPU model and family are defined in
[target/ppc/cpu-models.c](https://github.com/qemu/qemu/tree/v4.2.0/target/ppc/cpu-models.c#L642)
and
[target/ppc/translate_init.inc.c](https://github.com/qemu/qemu/tree/v4.2.0/target/ppc/translate_init.inc.c#L6462)
:


```c
POWERPC_DEF("755_v2.8", CPU_POWERPC_7x5_v28, 755, "PowerPC 755 v2.8")

POWERPC_FAMILY(755)(ObjectClass *oc, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(oc);
    PowerPCCPUClass *pcc = POWERPC_CPU_CLASS(oc);

    dc->desc = "PowerPC 755";
    pcc->init_proc = init_proc_755;
...
    pcc->mmu_model = POWERPC_MMU_SOFT_6xx;
    pcc->excp_model = POWERPC_EXCP_7x5;
    pcc->bus_model = PPC_FLAGS_INPUT_6xx;
    pcc->bfd_mach = bfd_mach_ppc_750;
    pcc->flags = POWERPC_FLAG_SE | POWERPC_FLAG_BE |
                 POWERPC_FLAG_PMM | POWERPC_FLAG_BUS_CLK;
}

static void init_proc_755(CPUPPCState *env)
{
...
    init_excp_7x5(env);
    ppc6xx_irq_init(ppc_env_get_cpu(env));
....
}
```

This is the very low level and detailled CPU init code. We can see MMU
type, interrupts and exceptions initialization code.


### Accessing your PowerPC CPU

Along the QEMU code, you will usually find dynamic type cast macro
such as `DEVICE_CLASS, CPU_CLASS, CPU or POWERPC_CPU`. They navigate
with safe type checking through the inherited classes the device
belongs to. They help you retrieve the correct definition for a given
runtime object.

```c
#define POWERPC_CPU(obj)  OBJECT_CHECK(PowerPCCPU, (obj), TYPE_POWERPC_CPU)
```

Below are some use cases:

```c
CPUState        *cs     = CPU(cpiom->cpu);
CPUClass        *cc     = CPU_GET_CLASS(cs);
PowerPCCPU      *ppc    = POWERPC_CPU(cs);
PowerPCCPUClass *ppc_cc = POWERPC_CPU_CLASS(cc);
CPUPPCState     *ppc_cs = &ppc->env;
```

In the
[`CPUClass`](https://github.com/qemu/qemu/tree/v4.2.0/include/hw/core/cpu.h#L77)
you will find function pointers related to the CPU family such as
execution, interrupt, mmu fault handling. While the
[`CPUState`](https://github.com/qemu/qemu/tree/v4.2.0/include/hw/core/cpu.h#L297)
will hold generic state of the CPU such as its running state,
configured breakpoints, last exception raised, ...

They are both related to overall QEMU CPU design. The part you might
be interested in is the *environment pointer* which keeps track of the
CPU registers. In our case, look for
[`CPUPPCState`](https://github.com/qemu/qemu/tree/v4.2.0/target/ppc/cpu.h#L962).


```c
struct PowerPCCPU {
...
    CPUPPCState env;
...
}

struct CPUPPCState {
...
    /* general purpose registers */
    target_ulong gpr[32];
    /* Storage for GPR MSB, used by the SPE extension */
    target_ulong gprh[32];
...
}
```

So if you want to access `GPR[14]` of your CPIOM CPU, you will end
doing something like:

```c
cpiom->cpu->env.gpr[14] = xxxx;
```

### Forcing software MMU for the CPIOM

Once the CPU object is created, we can proceed with some tuning for
instance on its core frequency or model specific register default
values. Our init function is also responsible for enabling software
MMU support for the MPC755 family which is not the case by default:

```c
static void cpiom_init_cpu(cpiom_t *cpiom, MachineState *mcs)
{
...
    CPUClass *cc = CPU_GET_CLASS( CPU(cpiom->cpu) );
    PowerPCCPUClass *pcc = POWERPC_CPU_CLASS(cc);
    CPUPPCState *env = &cpiom->cpu->env;

    /* Default mmu_model for MPC755 is POWERPC_MMU_SOFT_6xx (no softmmu) */
    pcc->mmu_model = POWERPC_MMU_32B;
    env->mmu_model = POWERPC_MMU_32B;
    pcc->handle_mmu_fault = ppc_hash32_handle_mmu_fault;
}
```

Under the QEMU terminology, `softmmu` stands for software memory
management unit. This is the software implementation of CPU hardware
MMU. Several implementations are available in the QEMU source code,
for different architectures.

In the post dedicated to QEMU memory internals, we will explain how
address translation is operated and why we had to do this.


### Conclusion

This is not exhaustive, but you got the point. If you need more
control over your CPU init, have a look at its
[definition](https://github.com/qemu/qemu/tree/v4.2.0/target/ppc/cpu.h#L1180).
