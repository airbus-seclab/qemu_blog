# A deep dive into QEMU: snapshot API

This blog post gives some insights on the QEMU snapshot API.

## Overview

The QEMU monitor exposes commands to create and restore snapshots of
your running VM: `savevm` and `loadvm`. QEMU offers advanced features
such as live migration that we won't deal with in this article.

However, the principle is based on the ability to save a complete and
restorable state of your virtual machine including its vCPU, RAM and
devices.

If we look at the service involved internally when invoking the
[`savevm`](https://github.com/qemu/qemu/tree/v4.2.0/monitor/hmp-cmds.c)
command we will land in
[`save_snapshot`](https://github.com/qemu/qemu/tree/v4.2.0/migration/savevm.c#L2619):

```c
void hmp_savevm(Monitor *mon, const QDict *qdict)
{
    Error *err = NULL;

    save_snapshot(qdict_get_try_str(qdict, "name"), &err);
    hmp_handle_error(mon, &err);
}


int save_snapshot(const char *name, Error **errp)
{
...
    if (!bdrv_all_can_snapshot(&bs)) {
        error_setg(errp, "Device '%s' is writable but does not support "
                   "snapshots", bdrv_get_device_name(bs));
        return ret;
    }

...
    ret = global_state_store();
    if (ret) {
        error_setg(errp, "Error saving global state");
        return ret;
    }
    vm_stop(RUN_STATE_SAVE_VM);

...
    f = qemu_fopen_bdrv(bs, 1);
    if (!f) {
        error_setg(errp, "Could not open VM state file");
        goto the_end;
    }
    ret = qemu_savevm_state(f, errp);
    vm_state_size = qemu_ftell(f);
    qemu_fclose(f);
    if (ret < 0) {
        goto the_end;
    }

...
    if (saved_vm_running) {
        vm_start();
    }
}
```

A lot of interesting things there. First the snapshot API is a file
based API. Second, your board' block devices (if any) must be
*snapshotable*. Third, there exists special [running
states](runstate.md) related to snapshoting.

The block device *snapshot-ability* is specific and mainly related to
device read-only mode. However, the most important part of the
[`save_snapshot`](https://github.com/qemu/qemu/tree/v4.2.0/migration/savevm.c#L2619)
function is tied to
[`qemu_savevm_state`](https://github.com/qemu/qemu/tree/v4.2.0/migration/savevm.c#L1501)
which will proceed through every device
[`VMStateDescription`](https://github.com/qemu/qemu/tree/v4.2.0/include/migration/vmstate.h#L176)
thanks to
[`vmstate_save_state_v`](https://github.com/qemu/qemu/tree/v4.2.0/migration/vmstate.c#L323)


## Preparing your devices

To be snapshotable, a device must expose through a
[`VMStateDescription`](https://github.com/qemu/qemu/tree/v4.2.0/include/migration/vmstate.h#L176),
all of its internal state fields that must be saved and restored
during snapshot handling.

Obviously, it's a device based implementation. Usually, configured IO
registers are saved to preserve what drivers may have done during
initialisation.

Let's take an example based on [our timer device](timers.md) implementation:

```c
static const VMStateDescription vmstate_cpiom_timer = {
    .name = CPIOM_TIMERS_NAME,
    .version_id = 1,
    .minimum_version_id = 1,
    .post_load = cpiom_timer_post_load,
    .fields = (VMStateField[]) {
        VMSTATE_UINT32(reg.base.ctrl, cpiom_timer_state_t),
        VMSTATE_UINT32(reg.base.prescal, cpiom_timer_state_t),
        VMSTATE_UINT32(reg.base.period, cpiom_timer_state_t),
        VMSTATE_UINT32(reg.base.counter, cpiom_timer_state_t),
        VMSTATE_UINT32(reg.base.cycle, cpiom_timer_state_t),
        VMSTATE_UINT32(reg.base.slice_end, cpiom_timer_state_t),
        VMSTATE_UINT32(reg.base.slice_out, cpiom_timer_state_t),
        VMSTATE_UINT32(reg.base.tick, cpiom_timer_state_t),
        VMSTATE_UINT32(reg.base.win_begin, cpiom_timer_state_t),
        VMSTATE_UINT32(reg.base.win_end, cpiom_timer_state_t),
        VMSTATE_UINT32(reg.base.win_sts, cpiom_timer_state_t),
        VMSTATE_UINT32(reg.base.win_sav_sts, cpiom_timer_state_t),
        VMSTATE_UINT32(reg.conf.rtc_period, cpiom_timer_state_t),
        VMSTATE_UINT32(reg.conf.cpt_rtc, cpiom_timer_state_t),
        VMSTATE_END_OF_LIST()
    },
};

static void cpiom_timer_class_init(ObjectClass *klass, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);

    dc->vmsd = &vmstate_cpiom_timer;
    dc->reset = cpiom_timer_reset;
    dc->desc = CPIOM_TIMERS_NAME;
}
```

As you can see, the `VMStateDescription` exposes all the internal
registers of the timer so that they get automatically saved and
restored during snapshots.

The `vmsd` field of the `DeviceClass` is used during device
realization by QEMU
[`qdev`](https://github.com/qemu/qemu/tree/v4.2.0/hw/core/qdev.c#L893)
API with
[`vmstate_register`](https://github.com/qemu/qemu/tree/v4.2.0/migration/savevm.c#L787).

If you look at QEMU git tree existing devices implementation, you will
find more complex definitions with
[`VMSTATE_PCI_DEVICE`](https://github.com/qemu/qemu/tree/v4.2.0/include/hw/pci/pci.h#L847)
or
[`VMSTATE_STRUCT`](https://github.com/qemu/qemu/tree/v4.2.0/include/migration/vmstate.h#L816)
declarations. Devices may also inherit higher-level/generic
`VMStateDescription` such as
[`serial-isa`](https://github.com/qemu/qemu/tree/v4.2.0/hw/char/serial-isa.c#L85):

```c
const VMStateDescription vmstate_serial = {
    .name = "serial",
    .version_id = 3,
    .minimum_version_id = 2,
    .pre_save = serial_pre_save,
    .pre_load = serial_pre_load,
    .post_load = serial_post_load,
    .fields = (VMStateField[]) {
        VMSTATE_UINT16_V(divider, SerialState, 2),
        VMSTATE_UINT8(rbr, SerialState),
        VMSTATE_UINT8(ier, SerialState),
        VMSTATE_UINT8(iir, SerialState),
        VMSTATE_UINT8(lcr, SerialState),
        VMSTATE_UINT8(mcr, SerialState),
        VMSTATE_UINT8(lsr, SerialState),
        VMSTATE_UINT8(msr, SerialState),
        VMSTATE_UINT8(scr, SerialState),
        VMSTATE_UINT8_V(fcr_vmstate, SerialState, 3),
        VMSTATE_END_OF_LIST()
    },
    .subsections = (const VMStateDescription*[]) {
        &vmstate_serial_thr_ipending,
        &vmstate_serial_tsr,
        &vmstate_serial_recv_fifo,
        &vmstate_serial_xmit_fifo,
        &vmstate_serial_fifo_timeout_timer,
        &vmstate_serial_timeout_ipending,
        &vmstate_serial_poll,
        NULL
    }
};

static const VMStateDescription vmstate_isa_serial = {
    .name = "serial",
    .version_id = 3,
    .minimum_version_id = 2,
    .fields = (VMStateField[]) {
        VMSTATE_STRUCT(state, ISASerialState, 0, vmstate_serial, SerialState),
        VMSTATE_END_OF_LIST()
    }
};
```

## Lower level handlers

The internals of the VMState save/load API is backed by
[`SaveStateEntry`](https://github.com/qemu/qemu/tree/v4.2.0/migration/savevm.c#L233)
fields. When you register a `vmsd` with
[`vmstate_register`](https://github.com/qemu/qemu/tree/v4.2.0/migration/savevm.c#L787),
a new [`SaveStateEntry` is
created](https://github.com/qemu/qemu/tree/v4.2.0/migration/savevm.c#L798):

```c
int vmstate_register_with_alias_id(DeviceState *dev, int instance_id,
                                   const VMStateDescription *vmsd,
                                   void *opaque, int alias_id,
                                   int required_for_version,
                                   Error **errp)
{
    SaveStateEntry *se;
...
    se = g_new0(SaveStateEntry, 1);
    se->version_id = vmsd->version_id;
    se->section_id = savevm_state.global_section_id++;
    se->opaque = opaque;
    se->vmsd = vmsd;
...
    savevm_state_handler_insert(se);
    return 0;
}
```

The
[`qemu_savevm_state`](https://github.com/qemu/qemu/tree/v4.2.0/migration/savevm.c#L1525)
function will iterate through the list of `SaveStateEntries` and call
their associated [`SaveVMHandlers
*ops`](https://github.com/qemu/qemu/tree/v4.2.0/include/migration/register.h#L17)
respectively from
[`qemu_savevm_state_setup`](https://github.com/qemu/qemu/tree/v4.2.0/migration/savevm.c#L1148)
and
[`qemu_savevm_state_iterate`](https://github.com/qemu/qemu/tree/v4.2.0/migration/savevm.c#L1210).

```c
typedef struct SaveVMHandlers {
...
    void (*save_cleanup)(void *opaque);
    int (*save_live_complete_postcopy)(QEMUFile *f, void *opaque);
    int (*save_live_complete_precopy)(QEMUFile *f, void *opaque);
...
} SaveVMHandlers;
```

Depending on the fact that a device has a `vmsd` or not when
registering to the `vmstate` API or loading a snapshot file, QEMU
might call either the `SaveVMHandlers` from the `SaveStateEntry` or
lower level functions such as
[`vmstate_save_state_v`](https://github.com/qemu/qemu/tree/v4.2.0/migration/vmstate.c#L323)
or
[`vmstate_load_state`](https://github.com/qemu/qemu/tree/v4.2.0/migration/vmstate.c#L78)
as we can see for instance in
[`vmstate_load`](https://github.com/qemu/qemu/tree/v4.2.0/migration//savevm.c#L851):

```c
static int vmstate_load(QEMUFile *f, SaveStateEntry *se)
{
    trace_vmstate_load(se->idstr, se->vmsd ? se->vmsd->name : "(old)");
    if (!se->vmsd) {         /* Old style */
        return se->ops->load_state(f, se->opaque, se->load_version_id);
    }
    return vmstate_load_state(f, se->vmsd, se->opaque, se->load_version_id);
}
```

## How the RAM is snapshoted ?

Sometimes it's hard to find your way into the QEMU code, with all that
function pointers initialized you don't know where depending on some
other fields value :)

Using a debugger might speed-up the proces ! Let's use it to
understand how RAM is snapshoted.

First,
[`memory_region_allocate_system_memory`](https://github.com/qemu/qemu/tree/v4.2.0/numa.c#L557)
registers something related to a vmstate with
[`vmstate_register_ram`]():

```c
void vmstate_register_ram(MemoryRegion *mr, DeviceState *dev)
{
    qemu_ram_set_idstr(mr->ram_block,
                       memory_region_name(mr), dev);
    qemu_ram_set_migratable(mr->ram_block);
}
```

And ... *cool story bro' !*


We should better try to break into
[`qemu_savevm_state_setup`](https://github.com/qemu/qemu/tree/v4.2.0/migration//savevm.c#L1501):

```shell
Breakpoint 2, qemu_savevm_state_setup (f=0x555556bde230)

(gdb) print se->idstr 
$6 = "ram", '\000' <repeats 252 times>

(gdb) print se->vmsd
$7 = (const VMStateDescription *) 0x0

(gdb) print se->opaque
$8 = (void *) 0x5555565fabd8 <ram_state>

(gdb) print se->is_ram 
$9 = 1

(gdb) print se->ops 
$10 = (SaveVMHandlers *) 0x55555648eb20 <savevm_ram_handlers>
(gdb) print se->ops->save_cleanup
$11 = (void (*)(void *)) 0x55555580fc9e <ram_save_cleanup>
(gdb) print se->ops->has_postcopy 
$12 = (_Bool (*)(void *)) 0x5555558125a5 <ram_has_postcopy>
(gdb) print se->ops->save_setup 
$13 = (int (*)(QEMUFile *, void *)) 0x555555810b61 <ram_save_setup>
(gdb) print se->ops->save_state 
$14 = (SaveStateHandler *) 0x0
(gdb) print se->ops->load_state 
$15 = (LoadStateHandler *) 0x555555811f3f <ram_load>
```

The RAM `SaveStateEntry` does not have any `vmsd` and its `opaque`
field is initialized to a
[`RAMState`](https://github.com/qemu/qemu/tree/v4.2.0/migration/ram.c#L306)
object that will be used by the specific [`RAM
SaveVMHandlers`](https://github.com/qemu/qemu/tree/v4.2.0/migration/ram.c#L4577).

Now we have function names to look at `migrate/ram.c`. We won't detail
the code from here. The interested reader might deep-dive into it and
discover the QEMU strategy looking for dirty memory pages to save to
optimize footprint.
