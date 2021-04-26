# A deep dive into QEMU: VM running states

This blog post details how the different running states of the virtual
machine (guest) are handled internally.

As you may imagine, a virtual machine and especially its virtual CPU
goes through different running states during its lifetime:

```c
/* from QAPI types */
typedef enum RunState {
    RUN_STATE_DEBUG,
    RUN_STATE_INMIGRATE,
    RUN_STATE_INTERNAL_ERROR,
    RUN_STATE_IO_ERROR,
    RUN_STATE_PAUSED,
    RUN_STATE_POSTMIGRATE,
    RUN_STATE_PRELAUNCH,
    RUN_STATE_FINISH_MIGRATE,
    RUN_STATE_RESTORE_VM,
    RUN_STATE_RUNNING,
    RUN_STATE_SAVE_VM,
    RUN_STATE_SHUTDOWN,
    RUN_STATE_SUSPENDED,
    RUN_STATE_WATCHDOG,
    RUN_STATE_GUEST_PANICKED,
    RUN_STATE_COLO,
    RUN_STATE_PRECONFIG,
    RUN_STATE__MAX,
} RunState;
```

These
[`RunState`](https://github.com/qemu/qemu/blob/v4.2.0/qapi/run-state.json)
are handled in the QEMU
[`main_loop`](https://github.com/qemu/qemu/tree/v4.2.0/vl.c#L1801)
which executes in the QEMU startup thread, not the ones dedicated to
[virtual
CPUs](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L2134). The
function just wais for event requests to be processed in
[`main_loop_should_exit`](https://github.com/qemu/qemu/tree/v4.2.0/vl.c#L1743). They
are generally raised from the virtual CPU threads:

```c
static void main_loop(void)
{
...
    while (!main_loop_should_exit()) {
        main_loop_wait(false);
    }
}

static bool main_loop_should_exit(void)
{
    RunState r;
...
    if (qemu_debug_requested()) {
        vm_stop(RUN_STATE_DEBUG);
    }
    if (qemu_suspend_requested()) {
        qemu_system_suspend();
    }
...
    if (qemu_powerdown_requested()) {
        qemu_system_powerdown();
    }
    if (qemu_vmstop_requested(&r)) {
        vm_stop(r);
    }
...
}
```

If you remember the [breakpoints handling post](brk.md), while
handling the debug exception being raised, QEMU prepared a **debug
request** to the main loop from
[`cpu_handle_guest_debug`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L1141):

```c
static void cpu_handle_guest_debug(CPUState *cpu)
{
    gdb_set_stop_cpu(cpu);
    qemu_system_debug_request();
    cpu->stopped = true;
}
```

The
[`qemu_system_debug_request`](https://github.com/qemu/qemu/tree/v4.2.0/vl.c#L1737)
actually triggers an event notification to the main loop:

```c
void qemu_system_debug_request(void)
{
    debug_requested = 1;
    qemu_notify_event();
}
```

Back to the main loop, QEMU checks for a debug event with
[`qemu_debug_requested`](https://github.com/qemu/qemu/tree/v4.2.0/vl.c#L1527)
and in that case changes the virtual machine running state to that of
the related event with
[`vm_stop`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L2161):

```c
static bool main_loop_should_exit(void)
{
...
    if (qemu_debug_requested()) {
        vm_stop(RUN_STATE_DEBUG);
...
}

int vm_stop(RunState state)
{
    if (qemu_in_vcpu_thread()) {
        qemu_system_vmstop_request_prepare();
        qemu_system_vmstop_request(state);
        /*
         * FIXME: should not return to device code in case
         * vm_stop() has been requested.
         */
        cpu_stop_current();
        return 0;
    }

    return do_vm_stop(state, true);
}

```

The [`vm_stop`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L2161)
function checks where it is called from. If for instance a virtual CPU
calls it during the emulation of an instruction, a **stop request** is
raised instead of handling the run state transition directly. Because
state transitions **only happen** in the QEMU main loop thread.

Obviously, there exists the opposite service to start/resume a VM:
[`vm_start`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L2211).


## VMState change handlers

The real state transition service is
[`do_vm_stop`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L1096)
and we can see it implements all the low level mechanics:

```c
static int do_vm_stop(RunState state, bool send_stop)
{
    int ret = 0;

    if (runstate_is_running()) {
        cpu_disable_ticks();
        pause_all_vcpus();
        runstate_set(state);
        vm_state_notify(0, state);
        if (send_stop) {
            qapi_event_send_stop();
        }
    }

    bdrv_drain_all();
    ret = bdrv_flush_all();

    return ret;
}
```

QEMU stops the vCPU, tick counting and associated virtual clocks. It
also calls a special service:
[`vm_state_notify`](https://github.com/qemu/qemu/tree/v4.2.0/vl.c#L1423)
with the new running state of the VM as argument. Interestingly, we
are able to register callbacks to get notified of every VM running
state change thanks to
[`qemu_add_vm_change_state_handler`](https://github.com/qemu/qemu/tree/v4.2.0/vl.c#L1411):

```c
VMChangeStateEntry *qemu_add_vm_change_state_handler_prio(
        VMChangeStateHandler *cb, void *opaque, int priority)
{
    VMChangeStateEntry *e;
    VMChangeStateEntry *other;

    e = g_malloc0(sizeof(*e));
    e->cb = cb;
    e->opaque = opaque;
    e->priority = priority;

    /* Keep list sorted in ascending priority order */
    QTAILQ_FOREACH(other, &vm_change_state_head, entries) {
        if (priority < other->priority) {
            QTAILQ_INSERT_BEFORE(other, e, entries);
            return e;
        }
    }

    QTAILQ_INSERT_TAIL(&vm_change_state_head, e, entries);
    return e;
}

void vm_state_notify(int running, RunState state)
{
    VMChangeStateEntry *e, *next;

    trace_vm_state_notify(running, state, RunState_str(state));

    if (running) {
        QTAILQ_FOREACH_SAFE(e, &vm_change_state_head, entries, next) {
            e->cb(e->opaque, running, state);
        }
    } else {
        QTAILQ_FOREACH_REVERSE_SAFE(e, &vm_change_state_head, entries, next) {
            e->cb(e->opaque, running, state);
        }
    }
}
```

This is extremely convenient. As an example, the [GDB server
stub](https://github.com/qemu/qemu/tree/v4.2.0//gdbstub.c#L3365) does
register a callback to intercept debug events and check for gdb client
breakpoints:

```c
int gdbserver_start(const char *device)
{
...
    qemu_add_vm_change_state_handler(gdb_vm_state_change, NULL);
...
}

static void gdb_vm_state_change(void *opaque, int running, RunState state)
{
...
    switch (state) {
    case RUN_STATE_DEBUG:
        ...
        ret = GDB_SIGNAL_TRAP;
        break;
...
}
```

## Asynchronous handling of running states

We have seen that we can't change running state from every where. We
should rather request for a change.

This is especially true for some events. Consider you implement a
clock device which uses QEMU internal timers. The virtual clock timers
expiration is processed in the QEMU main loop and the associated
callbacks are called from that thread.

In that context, we **should not** try to stop the VM because we have
seen that the
[`do_vm_stop`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L1096)
service will try to disable all vCPU associated virtual clocks. And
*disabling the clocks will wait for related timerlists to stop* as
stated in the [~~documentation~~
code](https://github.com/qemu/qemu/tree/v4.2.0/util/qemu-timer.c#L148). This
will lead to a **dead-lock**.

The correct way to proceed is to request a state transition from the
timer callback itself:

```c
void my_user_timeout_cb(void *opaque)
{
   debug("--> vm timeout()\n");
   qemu_system_vmstop_request_prepare();
   qemu_system_vmstop_request(RUN_STATE_PAUSED);
}
```

And in your vm change state handler, **aysnchronously** deal with that
state thanks to
[`async_run_on_cpu`](https://github.com/qemu/qemu/tree/v4.2.0/cpus-common.c#L149):

```c
void my_vm_state_change(void *opaque, int running, RunState state)
{
   debug("vm state %s\n", RunState_str(state));

   if (state == RUN_STATE_PAUSED) {
      async_run_on_cpu(cpu, my_async_timeout_vm, RUN_ON_CPU_HOST_PTR(arg));
      return;
   }
```

This way, the `my_async_timeout_vm` function is added into the given
`cpu` work queue as a new
[`qemu_work_item`](https://github.com/qemu/qemu/tree/v4.2.0/cpus-common.c#L101)
and will be called out of the main loop context. It is safe to
consider your VM in the requested state (PAUSED) now and try to resume
it with
[`vm_start`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L2211)
for instance.
