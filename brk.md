# A deep dive into QEMU: Breakpoints handling

In this post we will learn how breakpoints are checked and raised
during translation, and processed inside the vCPU main execution
loop. We assume an i386 target as most readers are familiar with this
architecture.

## From target to IR breakpoints

Remember QEMU translates target instructions with
[`tb_gen_code`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/translate-all.c#L1664)
which calls
[`gen_intermediate_code`](https://github.com/qemu/qemu/blob/v4.2.0/target/i386/translate.c#L8615). Obviously
this function is target specific because it translates from *target
assembly* to IR:

```c
void gen_intermediate_code(CPUState *cpu, TranslationBlock *tb, int max_insns)
{
    DisasContext dc;
    translator_loop(&i386_tr_ops, &dc.base, cpu, tb, max_insns);
}

static const TranslatorOps i386_tr_ops = {
    .init_disas_context = i386_tr_init_disas_context,
    .tb_start           = i386_tr_tb_start,
    .insn_start         = i386_tr_insn_start,
    .breakpoint_check   = i386_tr_breakpoint_check,
    .translate_insn     = i386_tr_translate_insn,
    .tb_stop            = i386_tr_tb_stop,
    .disas_log          = i386_tr_disas_log,
};
```

However the
[`translator_loop`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/translator.c#L34)
function is generic but called with target specific [translator
operators](https://github.com/qemu/qemu/blob/v4.2.0/include/exec/translator.h#L80)
(ie. `i386_tr_ops`).

During the translation process, QEMU checks for breakpoints hit:

```c
void translator_loop(const TranslatorOps *ops, DisasContextBase *db,
                     CPUState *cpu, TranslationBlock *tb, int max_insns)
{
    while (true) {
...
    /* Pass breakpoint hits to target for further processing */
    QTAILQ_FOREACH(bp, &cpu->breakpoints, entry) {
       if (bp->pc == db->pc_next) {
          if (ops->breakpoint_check(db, cpu, bp)) {
              break;
          }
        }
    }
...
}
```

The check makes use of target specific operator
`ops->breakpoint_check` which is set to
[`i386_tr_breakpoint_check`](https://github.com/qemu/qemu/blob/v4.2.0/target/i386/translate.c#L8534):

```c
static bool i386_tr_breakpoint_check(DisasContextBase *dcbase, CPUState *cpu,
                                     const CPUBreakpoint *bp)
{
    DisasContext *dc = container_of(dcbase, DisasContext, base);
    /* If RF is set, suppress an internally generated breakpoint.  */
    int flags = dc->base.tb->flags & HF_RF_MASK ? BP_GDB : BP_ANY;
    if (bp->flags & flags) {
        gen_debug(dc, dc->base.pc_next - dc->cs_base);
        dc->base.is_jmp = DISAS_NORETURN;
        dc->base.pc_next += 1;
        return true;
    } else {
        return false;
    }
}
```

## QEMU breakpoints

There exists different [breakpoint
types](https://github.com/qemu/qemu/blob/v4.2.0/include/hw/core/cpu.h#L1032)
inside QEMU. Some are installed from inside the VM, others might be
installed through the GDB server stub when debugging a VM. In such a
situation they are of type `BP_GDB` and are never ignored even if
`EFLAGS.RF` is set.

When there is a breakpoint hit, the
[`gen_debug`](https://github.com/qemu/qemu/blob/v4.2.0/target/i386/translate.c#L2528)
function is called which in turn calls `gen_helper_debug`, a *macro
generated* debug helper
[`helper_debug`](https://github.com/qemu/qemu/blob/v4.2.0/target/i386/misc_helper.c#L610):

```c
static void gen_debug(DisasContext *s, target_ulong cur_eip)
{
    gen_update_cc_op(s);
    gen_jmp_im(cur_eip);
    gen_helper_debug(cpu_env);
    s->base.is_jmp = DISAS_NORETURN;
}

void helper_debug(CPUX86State *env)
{
    CPUState *cs = CPU(x86_env_get_cpu(env));

    cs->exception_index = EXCP_DEBUG;
    cpu_loop_exit(cs);
}
```

The expected behavior should look familiar to you now. The
`exception_index` is set to `EXCP_DEBUG` which has a special meaning
and we go back to the main cpu execution loop, where
[`cpu_handle_exception`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cpu-exec.c#L481)
is called and does:

```c
static inline bool cpu_handle_exception(CPUState *cpu, int *ret)
{
...
        *ret = cpu->exception_index;
        if (*ret == EXCP_DEBUG) {
            cpu_handle_debug_exception(cpu);
        }
...
}
```

Here we are with our debug event handler
[`cpu_handler_debug_exception`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cpu-exec.c#L450):

```c
static inline void cpu_handle_debug_exception(CPUState *cpu)
{
    CPUClass *cc = CPU_GET_CLASS(cpu);
    CPUWatchpoint *wp;

    if (!cpu->watchpoint_hit) {
        QTAILQ_FOREACH(wp, &cpu->watchpoints, entry) {
            wp->flags &= ~BP_WATCHPOINT_HIT;
        }
    }

    cc->debug_excp_handler(cpu);
}
```

There is a special treatment for *watchpoints* that will be explained
later. But remember the `cc` pointer initialized in
[x86_cpu_common_class_init](https://github.com/qemu/qemu/blob/v4.2.0/target/i386/cpu.c#L7089). Its
`debug_excp_handler` field points to
[`breakpoint_handler`](https://github.com/qemu/qemu/blob/v4.2.0/target/i386/bpt_helper.c#L209):

```c
void breakpoint_handler(CPUState *cs)
{
    X86CPU *cpu = X86_CPU(cs);
    CPUX86State *env = &cpu->env;
    CPUBreakpoint *bp;

    if (cs->watchpoint_hit) {
        if (cs->watchpoint_hit->flags & BP_CPU) {
            cs->watchpoint_hit = NULL;
            if (check_hw_breakpoints(env, false)) {
                raise_exception(env, EXCP01_DB);
            } else {
                cpu_loop_exit_noexc(cs);
            }
        }
    } else {
        QTAILQ_FOREACH(bp, &cs->breakpoints, entry) {
            if (bp->pc == env->eip) {
                if (bp->flags & BP_CPU) {
                    check_hw_breakpoints(env, true);
                    raise_exception(env, EXCP01_DB);
                }
                break;
            }
        }
    }
}
```

This function is quite interesting. It decides if QEMU should inject
or not, the debug exception inside the target. Let's say for instance,
that the generated breakpoint is due to GDB from a host client. In
this case, no `raise_exception` happens and we return from
`breakpoint_handler`. But return where ? Out of
`cpu_handle_debug_exception`, then `cpu_handle_exception`, then
[`cpu_exec`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cpu-exec.c#L711)
and even out of
[`tcg_cpu_exec`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L1461)
to land back in
[`qemu_tcg_rr_cpu_thread_fn`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L1580):

```c
static void *qemu_tcg_rr_cpu_thread_fn(void *arg)
{

...
    r = tcg_cpu_exec(cpu);

    if (r == EXCP_DEBUG) {
        cpu_handle_guest_debug(cpu);
        break;
    }
...
```

Where QEMU deals with `EXCP_DEBUG` and calls
[`cpu_handle_guest_debug`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L1141) which has nothing to do anymore with low level target breakpoint handling:

```c
static void cpu_handle_guest_debug(CPUState *cpu)
{
    gdb_set_stop_cpu(cpu);
    qemu_system_debug_request();
    cpu->stopped = true;
}
```

At this stage, this is pure QEMU internals about event requests and VM
state changes. We will have a blog post on this too. What you should
keep in mind is that the QEMU
[`main_loop_should_exit`](https://github.com/qemu/qemu/tree/v4.2.0/vl.c#L1743)
function will check the debug request and all associated handlers will
be notified.


## QEMU watchpoints

The logic is pretty much the same for *execution watchpoints*. However
watchpoints can also be installed for read/write memory operations. To
that extent, the QEMU memory access path should check for possible
watchpoint hit.

This is happening in QEMU [virtual
TLB](https://github.com/qemu/qemu/tree/v4.2.0/accel/tcg/cputlb.c)
management code for the TCG execution mode. The implied function is
[`cpu_check_watchpoint`](https://github.com/qemu/qemu/tree/v4.2.0/exec.c#L2685).

As you are getting used to, we will cover this in a QEMU low level
memory management dedicated blog post :)
