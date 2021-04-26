# A deep dive into QEMU: The Tiny Code Generator (TCG), part 1

This blog post details some internals of the QEMU TCG engine, the
machinery responsible for executing target instructions on the host.

You should have already read [Execution loop](exec.md) and
[Breakpoints handling](brk.md) blog posts to have some pointers.


## Be kind rewind

The vCPU thread executes instructions through
[`tcg_cpu_exec`](https://github.com/qemu/qemu/tree/v4.2.0/cpus.c#L1461)
which finds and/or generates translated blocks.

As previously explained,
[`tb_gen_code`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/translate-all.c#L1734)
will generate *intermediate representation* (IR) code thanks to
[`gen_intermediate_code`](https://github.com/qemu/qemu/blob/v4.2.0/target/i386/translate.c#L8615)
and then host architecture assembly instructions with
[`tcg_gen_code`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/tcg.c#L4013):

```c
TranslationBlock *tb_gen_code(CPUState *cpu,
                              target_ulong pc, target_ulong cs_base,
                              uint32_t flags, int cflags)
{
    tb = tb_alloc(pc);
...
    /* generate IR code */
    gen_intermediate_code(cpu, tb, max_insns);
...
    /* generate machine code */
    gen_code_size = tcg_gen_code(tcg_ctx, tb);
...
}
```

The QEMU TCG has a notion of
[frontend](https://wiki.qemu.org/Documentation/TCG/frontend-ops) and
[backend](https://wiki.qemu.org/Documentation/TCG/backend-ops)
operations. The *frontend-ops* are the generated intermediate code
which is **what** QEMU will execute. The *backend-ops* are the
operations implemented on the host CPU, which is **where** the code is
executed.


## Generating Intermediate Representation (IR)

The QEMU git tree has a
[README](https://github.com/qemu/qemu/blob/v4.2.0/tcg/README)
introduction about the TCG which details the IR language. It's an
interesting read for sure.

### Overview

The
[`gen_intermediate_code`](https://github.com/qemu/qemu/blob/v4.2.0/target/i386/translate.c#L8615)
function is a VM architecture dependent wrapper to the
[`translator_loop`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/translator.c#L34)
generic function. Let's imagine we were emulating PowerPC code on an
Intel x86 host.

The
[`gen_intermediate_code`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/translate.c#L7963)
would look like:

```c
static const TranslatorOps ppc_tr_ops = {
    .init_disas_context = ppc_tr_init_disas_context,
    .tb_start           = ppc_tr_tb_start,
    .insn_start         = ppc_tr_insn_start,
    .breakpoint_check   = ppc_tr_breakpoint_check,
    .translate_insn     = ppc_tr_translate_insn,
    .tb_stop            = ppc_tr_tb_stop,
    .disas_log          = ppc_tr_disas_log,
};

void gen_intermediate_code(CPUState *cs, TranslationBlock *tb, int max_insns)
{
    DisasContext ctx;

    translator_loop(&ppc_tr_ops, &ctx.base, cs, tb, max_insns);
}
```

While the
[`translator_loop`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/translator.c#L34)
remains the same for any architecture, it relies on target specific
[translator
operators](https://github.com/qemu/qemu/blob/v4.2.0/include/exec/translator.h#L80). In
our case, the PowerPC ones (`ppc_tr_ops`).

```c
void translator_loop(const TranslatorOps *ops, DisasContextBase *db,
                     CPUState *cpu, TranslationBlock *tb, int max_insns)
{
    ops->init_disas_context(db, cpu);
...
    gen_tb_start(db->tb);
    ops->tb_start(db, cpu);

    while (true) {
        ops->translate_insn(db, cpu);
    }
...
    ops->tb_stop(db, cpu);
    gen_tb_end(db->tb, db->num_insns - bp_insn);
}
```

Each TB has a prologue (`tb_start`), and an epilogue (`tb_end`) with a
generic and target specific part (if needed). Usually epilogues are
placeholders for block chaining, which is an optimization feature of
the TCG that enables TBs to be called successively after execution,
without the need to get back to the QEMU code and look for the next TB
to execute.

### Disassembly context

The architecture dependent
[`DisasContext`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/translate.c#L155)
is created alongside a generic
[`DisasContextBase`](https://github.com/qemu/qemu/blob/v4.2.0/include/exec/translator.h#L56).

For the PPC target, the
[`ppc_tr_init_disas_context`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/translate.c#L7735)
handler will record current cpu state information. This means that TBs
are highly contextual and might not be reused in any situation where
we execute at a previously translated location.

```c
static void ppc_tr_init_disas_context(DisasContextBase *dcbase, CPUState *cs)
{
    DisasContext *ctx = container_of(dcbase, DisasContext, base);
    CPUPPCState *env = cs->env_ptr;
    int bound;

    ctx->exception = POWERPC_EXCP_NONE;
    ctx->spr_cb = env->spr_cb;
    ctx->pr = msr_pr;
    ctx->mem_idx = env->dmmu_idx;
    ctx->dr = msr_dr;
...
    ctx->insns_flags = env->insns_flags;
    ctx->insns_flags2 = env->insns_flags2;
    ctx->access_type = -1;
    ctx->need_access_type = !(env->mmu_model & POWERPC_MMU_64B);
    ctx->le_mode = !!(env->hflags & (1 << MSR_LE));
    ctx->default_tcg_memop_mask = ctx->le_mode ? MO_LE : MO_BE;
    ctx->flags = env->flags;
...
```

### Translated Block prologue/epilogue

The generic TB prologue is generated from
[`gen_tb_start`](https://github.com/qemu/qemu/blob/v4.2.0/include/exec/gen-icount.h#L35)

```c
static inline void gen_tb_start(TranslationBlock *tb)
{
    TCGv_i32 count, imm;

    tcg_ctx->exitreq_label = gen_new_label();
    if (tb_cflags(tb) & CF_USE_ICOUNT) {
        count = tcg_temp_local_new_i32();
    } else {
        count = tcg_temp_new_i32();
    }

    tcg_gen_ld_i32(count, cpu_env,
                   -ENV_OFFSET + offsetof(CPUState, icount_decr.u32));

    if (tb_cflags(tb) & CF_USE_ICOUNT) {
        imm = tcg_temp_new_i32();
        /* We emit a movi with a dummy immediate argument. Keep the insn index
         * of the movi so that we later (when we know the actual insn count)
         * can update the immediate argument with the actual insn count.  */
        tcg_gen_movi_i32(imm, 0xdeadbeef);
        icount_start_insn = tcg_last_op();

        tcg_gen_sub_i32(count, count, imm);
        tcg_temp_free_i32(imm);
    }

    tcg_gen_brcondi_i32(TCG_COND_LT, count, 0, tcg_ctx->exitreq_label);

    if (tb_cflags(tb) & CF_USE_ICOUNT) {
        tcg_gen_st16_i32(count, cpu_env,
                         -ENV_OFFSET + offsetof(CPUState, icount_decr.u16.low));
    }

    tcg_temp_free_i32(count);
}
```

It injects instructions to check for instruction count and an exit
condition.

The epilogue is generated from
[`gen_tb_end`](https://github.com/qemu/qemu/blob/v4.2.0/include/exec/gen-icount.h#L74). It
injects instructions to exit from the TB (`tcg_gen_exit_tb`).

```c
static inline void gen_tb_end(TranslationBlock *tb, int num_insns)
{
    if (tb_cflags(tb) & CF_USE_ICOUNT) {
        /* Update the num_insn immediate parameter now that we know
         * the actual insn count.  */
        tcg_set_insn_param(icount_start_insn, 1, num_insns);
    }

    gen_set_label(tcg_ctx->exitreq_label);
    tcg_gen_exit_tb(tb, TB_EXIT_REQUESTED);
}
```

The `TB_EXIT_REQUESTED` is a special value that tells QEMU to use the
`exitreq_label` created during the prologue. This label will
eventually be updated with the address of the next TB to be executed.


### Translating instructions

The operator used to translate target instructions to IR is
`translate_insn`
([`ppc_tr_translate_insn`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/translate.c#L7845)
for PowerPC). This function makes use of the target CPU
[`opcodes`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/translate.c#L6883)
handlers table which implements IR generation for every target native
instructions.

```c
static void ppc_tr_translate_insn(DisasContextBase *dcbase, CPUState *cs)
{
    opc_handler_t **table, *handler;

    table = cpu->opcodes;
    handler = table[opc1(ctx->opcode)];
...
    (*(handler->handler))(ctx);
}

struct PowerPCCPU {
...
    /* Those resources are used only during code translation */
    /* opcode handlers */
    opc_handler_t *opcodes[PPC_CPU_OPCODES_LEN];
...
}

static opcode_t opcodes[] = {
GEN_HANDLER(cmp, 0x1F, 0x00, 0x00, 0x00400000, PPC_INTEGER),
GEN_HANDLER(cmpi, 0x0B, 0xFF, 0xFF, 0x00400000, PPC_INTEGER),
GEN_HANDLER(cmpl, 0x1F, 0x00, 0x01, 0x00400001, PPC_INTEGER),
...
GEN_LDS(lbz, ld8u, 0x02, PPC_INTEGER)
GEN_LDS(lha, ld16s, 0x0A, PPC_INTEGER)
...
};
```

For the PowerPC `cmp` instruction, the associated handler will be
[`gen_cmp`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/translate.c#L666)
which will emit the following IR instructions:

```c
static void gen_cmp(DisasContext *ctx)
{
    if ((ctx->opcode & 0x00200000) && (ctx->insns_flags & PPC_64B)) {
        gen_op_cmp(cpu_gpr[rA(ctx->opcode)], cpu_gpr[rB(ctx->opcode)],
                   1, crfD(ctx->opcode));
    } else {
        gen_op_cmp32(cpu_gpr[rA(ctx->opcode)], cpu_gpr[rB(ctx->opcode)],
                     1, crfD(ctx->opcode));
    }
}

static inline void gen_op_cmp(TCGv arg0, TCGv arg1, int s, int crf)
{
    TCGv t0 = tcg_temp_new();
    TCGv t1 = tcg_temp_new();
    TCGv_i32 t = tcg_temp_new_i32();

    tcg_gen_movi_tl(t0, CRF_EQ);
    tcg_gen_movi_tl(t1, CRF_LT);
    tcg_gen_movcond_tl((s ? TCG_COND_LT : TCG_COND_LTU),
                       t0, arg0, arg1, t1, t0);
    tcg_gen_movi_tl(t1, CRF_GT);
    tcg_gen_movcond_tl((s ? TCG_COND_GT : TCG_COND_GTU),
                       t0, arg0, arg1, t1, t0);

    tcg_gen_trunc_tl_i32(t, t0);
    tcg_gen_trunc_tl_i32(cpu_crf[crf], cpu_so);
    tcg_gen_or_i32(cpu_crf[crf], cpu_crf[crf], t);

    tcg_temp_free(t0);
    tcg_temp_free(t1);
    tcg_temp_free_i32(t);
}
```

#### Example PowerPC basic block translation

The PowerPC code here after shows 3 types of operations:
- simple instructions: arithmetic, immediate operands (`lis, ori, xor`)
- a memory write (`stw`)
- a system register write (`mtmsr`)

```asm
0xfff00100:  lis    r1,1

0xfff00104:  ori    r1,r1,0x409c

0xfff00108:  xor    r0,r0,r0

0xfff0010c:  stw    r0,4(r1)

0xfff00110:  mtmsr  r0
```

The following code gives the TCG IR equivalent:

```asm
0xfff00100:  movi_i32    r1,$0x10000

0xfff00104:  movi_i32    tmp0,$0x409c
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

We notice that the memory write operation as well as the `MSR` access
are translated into *strange* IR opcodes:
- qemu_st_i32
- call store_msr

We will detail them in a blog post dedicated to TCG helpers.

Also notice the operands of the `exit_tb` opcode. The first one is
`$0x0` and might eventually be fixed with the **absolute host
address** of the next TB to be executed.

The last one is also a **host address** (`$0x7f5a0caf8043`) where code
is directly executed by the physical CPU. In this case, it goes back
to the QEMU translator.

At this point however, no *target translated* code is still
executed. We only end up with IR code which needs a final translation
step to the **host** architecture.
