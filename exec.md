# A deep dive into QEMU: The execution loop

This is the beginning of the second part of the blog post series. We
will go deeper into QEMU internals this time to give insights to hack
into core components. Let's look at the virtual CPU execution loop.


## The Big picture

In the very [first blog post](README.md) we explained how
accelerators were started, through
[`qemu_init_vcpu()`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L2134). Suppose
we run QEMU with a single threaded TCG and no hardware assisted
virtualization backend, we will end up running our virtual CPU in a
dedicated thread:

```c
static void qemu_tcg_init_vcpu(CPUState *cpu)
{
...
    qemu_thread_create(cpu->thread, thread_name,
                       qemu_tcg_rr_cpu_thread_fn,
                       cpu, QEMU_THREAD_JOINABLE);
...
}

static void *qemu_tcg_rr_cpu_thread_fn(void *arg)
{
...
    while (1) {
        while (cpu && !cpu->queued_work_first && !cpu->exit_request) {

            qemu_clock_enable(QEMU_CLOCK_VIRTUAL, ...);
            
            if (cpu_can_run(cpu)) {
                r = tcg_cpu_exec(cpu);

                if (r == EXCP_DEBUG) {
                    cpu_handle_guest_debug(cpu);
                    break;
                }
            }
            cpu = CPU_NEXT(cpu);
        }
    }
}
```

This is a very simplified view but we can see the big picture. If the
vCPU is in a *runnable* state then we execute instructions via the
TCG. We will detail how it handles asynchronous events such as
interrupts and exceptions, but we can already see there is a special
handling for `EXCP_DEBUG` in the previous code excerpt.

There is nothing architecture dependent at this level, we are still in
a generic part of the QEMU engine. The debug exception special
treament here is usually triggered by underlying architecture
dependent events (ie. *breakpoints*) and require particular attention
from QEMU to be forwarded to other subsystems such as a GDB server
stub out of the context of the VM. We will also cover breakpoints
handling in a dedicated post.

## Entering the TCG execution loop

The interesting function to start with is
[`tcg_cpu_exec`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L1461)
and more specifically the
[`cpu_exec`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cpu-exec.c#L661)
one. We will cover (definitely) in a future blog post the internals of
the TCG engine, but for now we only give an overview of the VM
execution. Simplified, it looks like:

```c
int cpu_exec(CPUState *cpu)
{
    cc->cpu_exec_enter(cpu);

    /* prepare setjmp context for exception handling */
    sigsetjmp(cpu->jmp_env, 0);

    /* if an exception is pending, we execute it here */
    while (!cpu_handle_exception(cpu, &ret)) {
        while (!cpu_handle_interrupt(cpu, &last_tb)) {
            tb = tb_find(cpu, last_tb, tb_exit, cflags);
            cpu_loop_exec_tb(cpu, tb, &last_tb, &tb_exit);
        }
    }

    cc->cpu_exec_exit(cpu);
}
```

QEMU makes use of `setjmp/longjmp` C library feature to implement
exception handling. This allows to get out of deep and complex TCG
translation functions whenever an event has been triggered, such as a
CPU interrupt or exception. The corresponding functions to exit the
CPU execution loop are
[`cpu_loop_exit_xxx`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cpu-exec-common.c#L65):

```c
void cpu_loop_exit(CPUState *cpu)
{
    /* Undo the setting in cpu_tb_exec.  */
    cpu->can_do_io = 1;
    siglongjmp(cpu->jmp_env, 1);
}
```

The vCPU thread code execution goes back to the point it called
`sigsetjmp`. Then QEMU tries to deal with the event as soon as
possible. But if there is no pending one, it executes the so-called
*Translated Blocks* (TB).


## A primer on Translated Blocks

The TCG engine is a JIT compiler, this means it dynamically translates
the target architecture instructions set to the host architecture
instruction set. For those not familiar with the concept please refer
to [this](https://en.wikipedia.org/wiki/Just-in-time_compilation) and
have a look at an introduction to the QEMU TCG engine
[here](https://wiki.qemu.org/Documentation/TCG). The translation is
done in two steps:
- from target ISA to intermediate representation (IR)
- from IR to host ISA

QEMU first tries to look for existing TBs, with
[`tb_find`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cpu-exec.c#L395). If
no one exists for the current location, it generates a new one with
[`tb_gen_code`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/translate-all.c#L1664):

```c
static inline TranslationBlock *tb_find(CPUState *cpu,
                                        TranslationBlock *last_tb,
                                        int tb_exit, uint32_t cf_mask)
{
...
    tb = tb_lookup__cpu_state(cpu, &pc, &cs_base, &flags, cf_mask);
    if (tb == NULL) {
        tb = tb_gen_code(cpu, pc, cs_base, flags, cf_mask);
...
}
```

When a TB is available, QEMU runs it with
[`cpu_loop_exec_tb`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cpu-exec.c#L611)
which in short calls
[`cpu_tb_exec`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cpu-exec.c#L141)
and then
[`tcg_qemu_tb_exec`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/tcg.h#L1160). At
this point the target (VM) code has been translated to host code, QEMU
can run it directly on the host CPU. If we look at the definition of
this last function:

```c
#define tcg_qemu_tb_exec(env, tb_ptr) \
    ((uintptr_t (*)(void *, void *))tcg_ctx->code_gen_prologue)(env, tb_ptr)
```

The translation buffer receiving generated opcodes is *casted* to a
function pointer and called with arguments.

In the TCG dedicated blog post, we will see the TCG strategy in detail
and present various *helpers* for system instructions, memory access
and things which can't be translated from an architecture to the
other.

## Back to events handling

When an hardware interrupt (IRQ) or exception is raised, QEMU *helps*
the vCPU redirects execution to the appropriate handler. These
mechanisms are very specific to the target architecture, conquentyl
hardly translatable. The answer comes from *helpers* which are tiny
wrappers written in C, built with QEMU for a target architecture and
natively callable on the host architecture directly from the
translated blocks. Again, we will cover them in details later.

For instance for the PPC target (VM), the *helpers* backend to inform
QEMU that an exception is being *raised* is located into
[excp_helper.c](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/excp_helper.c#L972):

```c
void raise_exception(CPUPPCState *env, uint32_t exception)
{
    raise_exception_err_ra(env, exception, 0, 0);
}

void raise_exception_err_ra(CPUPPCState *env, uint32_t exception,
                            uint32_t error_code, uintptr_t raddr)
{
    CPUState *cs = env_cpu(env);

    cs->exception_index = exception;
    env->error_code = error_code;
    cpu_loop_exit_restore(cs, raddr);
}
```

Notice the call to `cpu_loop_exit_restore` to get back to the main cpu
loop execution context and enter
[`cpu_handle_exception`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cpu-exec.c#L464):

```c
static inline bool cpu_handle_exception(CPUState *cpu, int *ret)
{
    if (cpu->exception_index >= EXCP_INTERRUPT) {
        /* exit request from the cpu execution loop */
        *ret = cpu->exception_index;
        if (*ret == EXCP_DEBUG) {
            cpu_handle_debug_exception(cpu);
        }
        cpu->exception_index = -1;
        return true;
    } else {
...
        /* deal with exception/interrupt */
        CPUClass *cc = CPU_GET_CLASS(cpu);
        cc->do_interrupt(cpu);
...
    }
}
```

There is once again a specific handling on *debug exceptions*, but in
essence if there is a pending exception in `cpu->exception_index` it
will be managed by `cc->do_interrupt` which is architecture dependent.

The `exception_index` field can hold the real hardware exception but
is also used for meta information (QEMU debug event, halt instruction,
VMEXIT for nested virtualization on x86).

The `CPUClass` type has several function pointers to be initialized by
specific target code. For an i386 target we may find it at
[x86_cpu_common_class_init](https://github.com/qemu/qemu/blob/v4.2.0/target/i386/cpu.c#L7040):

```c
static void x86_cpu_common_class_init(ObjectClass *oc, void *data)
{
    X86CPUClass *xcc = X86_CPU_CLASS(oc);
    CPUClass *cc = CPU_CLASS(oc);
    DeviceClass *dc = DEVICE_CLASS(oc);
...
#ifdef CONFIG_TCG
    cc->do_interrupt = x86_cpu_do_interrupt;
    cc->cpu_exec_interrupt = x86_cpu_exec_interrupt;
#endif
    cc->dump_state = x86_cpu_dump_state;
    cc->get_crash_info = x86_cpu_get_crash_info;
    cc->set_pc = x86_cpu_set_pc;
    cc->synchronize_from_tb = x86_cpu_synchronize_from_tb;
    cc->gdb_read_register = x86_cpu_gdb_read_register;
    cc->gdb_write_register = x86_cpu_gdb_write_register;
    cc->get_arch_id = x86_cpu_get_arch_id;
    cc->get_paging_enabled = x86_cpu_get_paging_enabled;
...
}
```

The underlying `x86_cpu_do_interrupt` is a place holder for various
cases (userland, system emulation or nested virtualization). In basic
system emulation mode it will call
[`do_interrupt_all`](https://github.com/qemu/qemu/blob/v4.2.0/target/i386/seg_helper.c#L1206)
which implements low level x86 specific interrupt handling.
