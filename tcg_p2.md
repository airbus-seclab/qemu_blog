# A deep dive into QEMU: The Tiny Code Generator (TCG), part 2

This blog post is the second [part](tcg_p1.md) dedicated to the QEMU
TCG engine. It covers TCG IR to host code translation and TBs
execution.

## Generating host code

In the context of the TCG, the terminology changes. The target isn't
the VM anymore but the host from the TCG point of view. Indeed, the
TCG generates assembly code from IR to be run on the host
architecture. So it's natural that its *target* is the host
architecture. It's referred as *tcg-target*.

As such, the *tcg-target* specific code is located at
`tcg/ARCH/tcg-target.inc.c`. For our Intel x86 host, it would be
[`tcg/i386/tcg-target.inc.c`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/i386/tcg-target.inc.c)

The *tcg-target* code is generated thanks to
[`tcg_gen_code`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/tcg.c#L4013):

```c
int tcg_gen_code(TCGContext *s, TranslationBlock *tb)
{
    int i, num_insns;
    TCGOp *op;
...
#ifdef TCG_TARGET_NEED_LDST_LABELS
    QSIMPLEQ_INIT(&s->ldst_labels);
#endif

    tcg_reg_alloc_start(s);

    QTAILQ_FOREACH(op, &s->ops, link) {
        TCGOpcode opc = op->opc;

        switch (opc) {
        case INDEX_op_mov_i32:
        case INDEX_op_mov_i64:
            tcg_reg_alloc_mov(s, op);
...
        case INDEX_op_call:
            tcg_reg_alloc_call(s, op);
            break;
        default:
            tcg_reg_alloc_op(s, op);
            break;
        }
    }
#ifdef TCG_TARGET_NEED_LDST_LABELS
    i = tcg_out_ldst_finalize(s);
    if (i < 0) {
        return i;
    }
#endif
...
}
```

Beside several optimizations and debug code not shown here, the
translation process generates corresponding host opcodes and registers
allocation from IR instructions. We will cover the `ldst` labels in a
blogpost dedicated to TCG memory accesses implementation
(ie. `qemu_ld`, `qemu_st`).

Remember the previous article TCG IR code extract:

```asm
0xfff00100:  movi_i32    r1,$0x10000
             movi_i32    tmp0,$0x409c
             or_i32      r1,r1,tmp0

0xfff00108:  movi_i32    r0,$0x0

0xfff0010c:  movi_i32    tmp1,$0x4
             add_i32     tmp0,r1,tmp1
             qemu_st_i32 r0,tmp0,beul,3

0xfff00110:  movi_i32    nip,$0xfff00114
             mov_i32     tmp0,r0
             call        store_msr,$0,tmp0

             movi_i32    nip,$0xfff00114
             exit_tb     $0x0
             set_label   $L0
             exit_tb     $0x7f5a0caf8043
```

It is translated into the following x86 assembly code:

```asm
0x7f5a0caf810b:  movl     $0x1409c, 4(%rbp)

0x7f5a0caf8112:  xorl     %ebx, %ebx

0x7f5a0caf8114:  movl     %ebx, (%rbp)
0x7f5a0caf8117:  movl     $0x140a0, %r12d
0x7f5a0caf811d:  movl     %r12d, %edi
0x7f5a0caf8129:  addq     0x398(%rbp), %rdi
...
0x7f5a0caf8159:  movq     %rbp, %rdi
0x7f5a0caf815c:  movl     %ebx, %esi
0x7f5a0caf815e:  callq    *0x34(%rip)

0x7f5a0caf8164:  movl     $0xfff00114, 0x16c(%rbp)
0x7f5a0caf8182:  movl     %ebx, %edx
0x7f5a0caf8184:  movl     $0xa3, %ecx
0x7f5a0caf8189:  leaq     -0x41(%rip), %r8
0x7f5a0caf8190:  pushq    %r8
0x7f5a0caf8192:  jmpq     *8(%rip)
0x7f5a0caf8198:  .quad  0x000055d62e46eba0
0x7f5a0caf81a0:  .quad  0x000055d62e3895a0
```

The x86 instruction set allows optimizations toward IR RISC-like
opcodes (ie. load immediate 32 bits value in a single
instruction). The immediate value memory store operation `qemu_st_i32`
has been directly translated at address `0x7f5a0caf8129`.

However, the `call store_msr` is implemented at address
`0x7f5a0caf815e` thanks to `RIP relative` addressing. The call site
locations are mixed within the generated code (`.quad xxxxx`). The
target function address is `0x000055d62e46eba0` which when you
disassemble the QEMU engine binary gives us:

```asm
(gdb) x/i 0x000055d62e46eba0
   0x000055d62e46eba0 <helper_store_msr>: push %r12
```

This is a QEMU *TCG helper* which we'll have a closer look at ... right now !


## Executing translated blocks

Now that we have host assembly code, we can directly run it on our
physical CPU. How does QEMU do that ?

The execution flow of a virtual CPU has been depicted in the
[execution loop blog post](exec.md). The
[`cpu_exec`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cpu-exec.c#L661)
function calls
[`cpu_loop_exec_tb`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cpu-exec.c#L611)
which then calls
[`cpu_tb_exec`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cpu-exec.c#L141)
and finally
[`tcg_qemu_tb_exec`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/tcg.h#L1160)
which is nothing more than a *function pointer cast* to the TB
buffer. Literally, QEMU *calls* the generated code.

However, some parts of the generated code redirect to *special
handlers* for IR operations that couldn't be translated into host
assembly code. For instance the PowerPC write to MSR (`mtmsr`) has no
equivalent Intel x86 instruction. It is a system specific instruction,
not a general purpose one.

The `mtmstr` opcode has been translated into IR `call store_msr` which
results in Intel x86 `callq *0x34(%rip)` redirecting to the QEMU
engine function `helper_store_msr`.


## TCG Helpers

The
[`helper_store_msr`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/excp_helper.c#L1013)
function is implemented inside the PowerPC emulation part of
QEMU. That's the trade-off between emulation and virtualization. The
QEMU TCG is a *JIT-compiler* for general purpose instructions found in
any architecture. But once we reach system specific instructions, they
remain *emulated* and emulation requires a lot more *host*
instructions to be executed for a single *guest* instruction.

```c
void helper_store_msr(CPUPPCState *env, target_ulong val)
{
    uint32_t excp = hreg_store_msr(env, val, 0);

    if (excp != 0) {
        CPUState *cs = env_cpu(env);
        cpu_interrupt_exittb(cs);
        raise_exception(env, excp);
    }
}
```

The
[`hreg_store_msr`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/helper_regs.h#L114)
implements the behavior of a write to the PowerPC `MSR` register. If
the emulation detects a `CPU exception` condition, QEMU will be able
to generate the event at the virtual CPU level thanks to
[`raise_exception`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/excp_helper.c#L990)
which redirects execution back to the [main cpu loop events
handling](exec.md#back-to-events-handling).

Whilst `helper_store_msr` is the last step of the emulation process,
how does QEMU knows that it should translate the PowerPC `mtmsr`
instruction into a call to the `helper_store_msr` TCG helper ?

We already explained how guest instructions are [first
translated](tcg_p1.md#translating-instructions) into
IR. The `opcodes` table has also an entry for our PowerPC system
instruction `mtmsr`:

```c
static opcode_t opcodes[] = {
...
GEN_HANDLER(mtmsr, 0x1F, 0x12, 0x04, 0x001EF801, PPC_MISC),
...
};

#define GEN_HANDLER(name, opc1, opc2, opc3, inval, type)                      \
GEN_OPCODE(name, opc1, opc2, opc3, inval, type, PPC_NONE)

#define GEN_OPCODE(name, op1, op2, op3, invl, _typ, _typ2)                    \
{                                                                             \
    .opc1 = op1,                                                              \
    .opc2 = op2,                                                              \
    .opc3 = op3,                                                              \
    .opc4 = 0xff,                                                             \
    .handler = {                                                              \
        .inval1  = invl,                                                      \
        .type = _typ,                                                         \
        .type2 = _typ2,                                                       \
        .handler = &gen_##name,                                               \
    },                                                                        \
    .oname = stringify(name),                                                 \
}
```

This tells the QEMU translation engine that it should call
[`gen_mtmsr`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/translate.c#L4390)
whenever it encounters the `mtmsr` instruction:

```c
static void gen_mtmsr(DisasContext *ctx)
{
    CHK_SV;
...
    gen_update_nip(ctx, ctx->base.pc_next);
    tcg_gen_mov_tl(msr, cpu_gpr[rS(ctx->opcode)]);
    gen_helper_store_msr(cpu_env, msr);
...
}
```

If you remember our [previous
articles](tcg_p1.md#example-powerpc-basic-block-translation) example
PowerPC basic block, you will find the IR opcodes related to
`gen_update_nip`, `tcg_gen_mov_tl` and `gen_helper_store_msr`:

```asm
0xfff00110:  movi_i32    nip,$0xfff00114
             mov_i32     tmp0,r0
             call        store_msr,$0,tmp0
```

There is no definition of `gen_helper_store_msr` in the QEMU code,
because it is *generated* during compilation. The programmer provides
the final implementation into `helper_store_msr` and QEMU generates
the glue to reach that function.

The starting point for the PowerPC is
[target/ppc/helper.h](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/helper.h#L8):

```c
DEF_HELPER_2(store_msr, void, env, tl)
```

And subsequent macro definitions in
[helper-head.h](https://github.com/qemu/qemu/blob/v4.2.0/include/exec/helper-head.h#L141)
and
[helper-gen.h](https://github.com/qemu/qemu/blob/v4.2.0/include/exec/helper-gen.h):

```c
#define HELPER(name) glue(helper_, name)

#define DEF_HELPER_2(name, ret, t1, t2) \
    DEF_HELPER_FLAGS_2(name, 0, ret, t1, t2)

#define DEF_HELPER_FLAGS_2(name, flags, ret, t1, t2)                    \
static inline void glue(gen_helper_, name)(dh_retvar_decl(ret)          \
    dh_arg_decl(t1, 1), dh_arg_decl(t2, 2))                             \
{                                                                       \
  TCGTemp *args[2] = { dh_arg(t1, 1), dh_arg(t2, 2) };                  \
  tcg_gen_callN(HELPER(name), dh_retvar(ret), 2, args);                 \
}
```

Without exposing too many details, the several levels of macro
expansion transform `gen_helper_store_msr()` into
`tcg_gen_callN(helper_store_msr)`. And
[tcg_gen_callN](https://github.com/qemu/qemu/blob/v4.2.0/tcg/tcg.c#L1688)
is simply responsible for the generation of a *function call* to our
helper function `helper_store_msr`.


## Last words

To be able to simulate *system* instructions, QEMU introduced the
*helper* concept which allows programmers to implement their emulation
inside the QEMU engine.

We will see in the [next blog post](tcg_p3.md) that the QEMU helpers
are largely used to implement guest memory accesses with the support
of virtual TLBs. They are also used for dealing with exceptions. The
interested reader can have a look at
[target/ppc/excp_helper.c](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/excp_helper.c)
and
[target/ppc/mem_helper.c](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/mem_helper.c).
