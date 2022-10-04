# A deep dive into QEMU: PCI host bridge controller

The QEMU PCI subsystem is really interesting to understand. Because
PCI is a [specification](https://pcisig.com/specifications), QEMU
implements everything to simplify your life of *new PCI device*
developper.

## Some basics about PCI

A lot of ressources are available on
[PCI](https://wiki.osdev.org/PCI). Please take time to get familiar
wih the concepts involved.

Now that you are a PCI expert, you know what we have to do to expose a
PCI host bridge. Fortunately, the PCI config space is a standard
and device specific IO memory regions are located through config space
`BAR` registers. Seems we just have to setup config space values for
each simulated PCI device.

Our example PCI host bridge implementation is largely inspired by the
[RAVEN](https://github.com/qemu/qemu/blob/v4.2.0/hw/pci-host/prep.c)
PCI host bridge available in QEMU source code.

Implementing a PCI host bridge is not a tremendous task. An
interesting part is related to QEMU internal PCI API, where PCI
devices dynamically expose their IO memory regions through PCI config
space BAR. Apart from that, a PCI host bridge is just another device.

We will see in the implementation that a PCI host bridge is also a PCI
device.

## CPIOM PCI subsystem

Below is an overview of an imaginary PCI organisation in our CPIOM. We
will mainly focus on the PCI host bridge `master` and PCI slave device
`mxfc` which exposes through BAR registers the different flash
memories and a serial controller.

<img alt="CPIOM PCI subsystem" src="/assets/images/6d54ba55-f68f-43d1-a020-aca2e3a9468b.png">

The code flash holds the firmware which is used to bootstrap the CPIOM
board. It must be available to the very first instruction fetched by
the CPU. We will detail flash simulation in another blog post.

The CPIOM machine initialization code is responsible for PCI devices
instantiation:

```c
static void cpiom_init_dev(cpiom_t *cpiom)
{
...
    pci_init(cpiom);
}

static void pci_init(cpiom_t *cpiom)
{
    DeviceState        *d;
    MasterPCIHostState *h;

    /* Master PCI Host controller */
    d = qdev_create(NULL, "master-pcihost");
    object_property_add_child(qdev_get_machine(), "master", OBJECT(d), NULL);

    h = MASTER_PCI_HOST_BRIDGE(d);
    memory_region_add_subregion(cpiom->config, 0x24, &h->pci_reg);

    qdev_init_nofail(d);

    PCIBus *pci_bus = (PCIBus *)qdev_get_child_bus(d, "pci.0");
    if (pci_bus == NULL) {
        cpiom_error("Couldn't create PCI host controller");
    }

    /* Slave PCI Device */
    pci_create_simple(pci_bus, PCI_DEVFN(1, 0), "mxfc");
}
```

As you can see, we first create the PCI host bridge `master` and then
attach our slave device `mxfc` to its bus. We are using the low-level
`qdev` API to create the host controller. Also notice the
`cpiom->config` subregion mapping some of the PCI host bridge IO
registers.

Now let's have a look at the master implementations.


## PCI Master

This component is the PCI host bridge. The device type declaration is
equivalent to other QOM devices:

```c
static void master_pcihost_class_init(ObjectClass *klass, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);

    set_bit(DEVICE_CATEGORY_BRIDGE, dc->categories);
    dc->realize = master_pcihost_realizefn;
    dc->props = master_pcihost_properties;
    dc->fw_name = "pci";
}

static const TypeInfo master_pcihost_info = {
    .name = TYPE_MASTER_PCI_HOST_BRIDGE,
    .parent = TYPE_PCI_HOST_BRIDGE,
    .instance_size = sizeof(MasterPCIHostState),
    .instance_init = master_pcihost_initfn,
    .class_init = master_pcihost_class_init,
};
```

We declare ourself as a child to `TYPE_PCI_HOST_BRIDGE` and
`DEVICE_CATEGORY_BRIDGE` category. The following
[categories](https://github.com/qemu/qemu/blob/v4.2.0/include/hw/qdev-core.h#L18)
are available inside QEMU:

```c
typedef enum DeviceCategory {
    DEVICE_CATEGORY_BRIDGE,
    DEVICE_CATEGORY_USB,
    DEVICE_CATEGORY_STORAGE,
    DEVICE_CATEGORY_NETWORK,
    DEVICE_CATEGORY_INPUT,
    DEVICE_CATEGORY_DISPLAY,
    DEVICE_CATEGORY_SOUND,
    DEVICE_CATEGORY_MISC,
    DEVICE_CATEGORY_CPU,
    DEVICE_CATEGORY_MAX
} DeviceCategory;
```

### The host bridge controller part

The most important part of host init code is located in
`master_pcihost_initfn` and `master_pcihost_realizefn`
functions. `Realized` is a property of QOM device state, usually set
after device instantiation. Have a look at
[DeviceClass](https://github.com/qemu/qemu/blob/v4.2.0/include/hw/qdev-core.h#L47)
for more details.

But let's have a quick overview of our host bridge type in the first
place. It's a little bit more complex than a simple QOM device:

```c
typedef struct MasterPCIDevState {
    PCIDevice    dev;
    MemoryRegion pci_cfg;
} MasterPCIDevState;

typedef struct MasterPCIHostState {
    PCIHostState         parent_obj;

    PCIBus               pci_bus;
    MemoryRegion         pci_reg;
    MemoryRegion         pci_mem;
    MasterPCIDevState    pci_dev;

} MasterPCIHostState;
```

It has an associated `PCIBus`, memory mapped IO registers (`pci_reg`),
a memory region overlapping system memory (`pci_mem`) and a dedicated
device `pci_dev`. We will also see that its parent type
(`PCIHostState`) holds important things.

The `pci_mem` is a full-span (4GB for a 32 bits system) memory overlap
of the system memory region with a lower access priority. It's used
merely by the QEMU PCI subsystem to dynamically configure BAR devices
(more on this later).

The `pci_reg` is specific to our host bridge implementation and allows
to intercept register accesses as for any device. It also helps to
intercept standard PCI config space IO operations through `CONF_ADDR`
and `CONF_DATA` registers.


### Accessing the PCI config space

When you want to access the PCI config space you usually make use of
`in/out` instructions targetting ports `0xcf8/0xcfc`. However, for the
PowerPC architecture there is no such `in/out` instructions, the PCI
config space is directly memory mapped (ala `PCI-express`).

As this mechanism is part of the PCI specification, QEMU provides
generic handlers to ease config space access:
[`pci_host_{data/conf}_{le/be}_ops`](https://github.com/qemu/qemu/blob/v4.2.0/hw/pci/pci_host.c#L190).

If you dig into the QEMU code, you will discover that these handlers
end up accessing PCI device `config_read/config_write`
callbacks. Hence the need for our controller to hold a `pci_dev`
object to expose its own PCI config space.

We will see in the init functions below how we connect these standard
handlers to the right IO memory area of our host brigde controller.


### The host bridge device part

This device is present to expose the PCI config space part of our host
bridge controller: vendor id, device id, revision and so on.

You usually don't need to reimplement the `config_read/config_write`
handlers. QEMU provides you with default ones
[`pci_default_read_config`](https://github.com/qemu/qemu/blob/v4.2.0/hw/pci/pci.c#L1380).

During PCI device initialization, you might setup its config space
values as the following:

```c
static void master_realize(PCIDevice *d, Error **errp)
{
    MasterPCIDevState *s = MASTER_PCI_DEVICE(d);

    pci_set_byte(d->config + PCI_LATENCY_TIMER, 0x80);
    pci_set_word(d->config + PCI_COMMAND, 0x1d6);
    pci_set_long(d->config + PCI_BASE_ADDRESS_0, 0x80000000);
}
```


### Putting it all together


The association of these components to the appropriate handlers and
mappings is done in the aforementioned initialization functions:

```c
static void master_pcihost_realizefn(DeviceState *d, Error **errp)
{
    PCIHostState       *h = PCI_HOST_BRIDGE(d);
    MasterPCIHostState *s = MASTER_PCI_HOST_BRIDGE(d);

    /* CONFIG_ADDR & CONFIG_DATA registers */
    memory_region_init_io(&h->conf_mem, OBJECT(h), &pci_host_conf_be_ops, s,
                          "pci-conf-addr", 4);

    memory_region_init_io(&h->data_mem, OBJECT(h), &pci_host_data_le_ops, s,
                          "pci-conf-data", 4);

    memory_region_add_subregion(&s->pci_reg, 0, &h->conf_mem);
    memory_region_add_subregion(&s->pci_reg, 4, &h->data_mem);

    object_property_set_bool(OBJECT(&s->pci_bus), true, "realized", errp);
    object_property_set_bool(OBJECT(&s->pci_dev), true, "realized", errp);
}

static void master_pcihost_initfn(Object *obj)
{
    PCIHostState       *phs = PCI_HOST_BRIDGE(obj);
    MasterPCIHostState *hhs = MASTER_PCI_HOST_BRIDGE(obj);
    DeviceState        *dev;

    memory_region_init(&hhs->pci_mem, obj, "pci", UINT32_MAX);
    memory_region_add_subregion_overlap(get_system_memory(),
                                        0, &hhs->pci_mem, -1);

    memory_region_init_io(&hhs->pci_reg, obj, &master_reg_ops, hhs,
                          "master-reg", 4*4);

    pci_root_bus_new_inplace(&hhs->pci_bus, sizeof(hhs->pci_bus),
                             DEVICE(obj), NULL,
                             &hhs->pci_mem,
                             &hhs->pci_reg,
                             0, TYPE_PCI_BUS);

    /* Create Master pci device 0.0 */
    phs->bus = &hhs->pci_bus;
    object_initialize(&hhs->pci_dev, sizeof(hhs->pci_dev),
                      TYPE_MASTER_PCI_DEVICE);

    dev = DEVICE(&hhs->pci_dev);
    qdev_set_parent_bus(dev, BUS(&hhs->pci_bus));
    object_property_set_int(OBJECT(&hhs->pci_dev), PCI_DEVFN(0,0), "addr", NULL);
    qdev_prop_set_bit(dev, "multifunction", false);
}
```

Notice how we attached the PCI device to the host bridge controller.
