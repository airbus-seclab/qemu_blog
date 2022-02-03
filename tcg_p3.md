# A deep dive into QEMU: TCG memory accesses

This blog post details how the QEMU TCG engine manages guest memory
accesses.

As as JIT-compiler enabling guest code to be translated into host
code, a lot of questions arise:
- How does a virtual PowerPC userland address can be translated and
accessed into the host ?
- How the host cpu instructions are restricted to only access the
available QEMU virtual machine memory ?
- Is there an address translation cache ?
- What about the performances ?

Don't hesitate to refresh your QEMU memory model knowledge by reading
[this blog post](regions.md).


## Environment

We assume the reader to have previous knowledge of memory management
in modern architectures and operating systems. A lot of ressources are
publicly available:
- [Memory Management Unit](https://wiki.osdev.org/Memory_Management_Unit)
- [Memory Paging](https://wiki.osdev.org/Paging)
- [TLBs](https://wiki.osdev.org/TLB)

As usual in the blog series, we will assume the guest being a PowerPC
32 bits machine and the host an Intel x86_64 one.

We won't detail anything related to address translation in the host,
because QEMU is a simple user process of your hosting operating
system. The guest physical memory (RAM) is just a buffer allocated
into that QEMU process. The QEMU TCG job is thus to redirect any guest
memory access to address X (whether virtual or physical) to that
memory buffer.

Obviously, the guest vCPU type and running mode will have an impact on
how addresses are translated. This is where the so called
*QEMU-softmmu* enters the game. In system-mode emulation, as opposed
to user-mode, and for some architectures the QEMU engine supports a
*Software Memory Managment Unit* (soft-MMU). This component is able to
translate guest virtual addresses into guest physical ones. QEMU also
supports *Virtual Translation Lookaside Buffers* (vTLBs) to speed-up
further accesses to previously translated addresses.

The guest physical address is always translated into an offset to the
QEMU maintained representation of the guest RAM, which is usually a
memory mapped area in the host (a buffer obtained via `malloc()` or
`mmap()`).


## Analysing a QEMU TCG memory store operation

What happens in QEMU when it has to translate the following PowerPC
instruction:

```asm
0xfff0017c:  90010004  stw      r0, 4(r1)
```

### Translating PowerPC `stw` into IR

Like any other instructions, the
[`opcodes`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/translate.c#L7381)
table has a specific entry for `stw`:

```c
static opcode_t opcodes[] = {
...
GEN_STS(stw, st32, 0x04, PPC_INTEGER)
...
};

#define GEN_ST(name, stop, opc, type)                        \
  GEN_HANDLER(name, opc, 0xFF, 0xFF, 0x00000000, type),

#define GEN_HANDLER(name, opc1, opc2, opc3, inval, type)     \
  GEN_OPCODE(name, opc1, opc2, opc3, inval, type, PPC_NONE)
```

We purposely omitted the expansion of `GEN_STS` to `GEN_HANDLER`
because it breaks down into all signed, unsigned and extended variant
of the store operation which is of no interest to us.

The `GEN_OPCODE` macro will expand to the declaration, if you remember
our [TCG article](tcg_p2.md#tcg-helpers), of the
[`gen_stw`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/translate.c#L2717)
handler. We can't find its direct definition in the QEMU source code,
as for TCG *helpers*, it is partly generated at compilation time:

```c
#define GEN_ST(name, stop, opc, type)                                         \
static void glue(gen_, name)(DisasContext *ctx)                               \
{                                                                             \
    TCGv EA;                                                                  \
    gen_set_access_type(ctx, ACCESS_INT);                                     \
    EA = tcg_temp_new();                                                      \
    gen_addr_imm_index(ctx, EA, 0);                                           \
    gen_qemu_##stop(ctx, cpu_gpr[rS(ctx->opcode)], EA);                       \
    tcg_temp_free(EA);                                                        \
}
```

Beside some effective address computation, the interesting line of
code is the expansion of `gen_qemu_##stop` into
[`gen_qemu_st32`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/translate.c#L2489)
which again has no direct definition but is generated thanks to
several macro expansions:

```c
#define GEN_QEMU_STORE_TL(stop, op)                                     \
static void glue(gen_qemu_, stop)(DisasContext *ctx,                    \
                                  TCGv val,                             \
                                  TCGv addr)                            \
{                                                                       \
    tcg_gen_qemu_st_tl(val, addr, ctx->mem_idx, op);                    \
}

GEN_QEMU_STORE_TL(st32, DEF_MEMOP(MO_UL))

/* from tcg/tcg-op.h */
#if TARGET_LONG_BITS == 32
...
#define tcg_gen_qemu_st_tl tcg_gen_qemu_st_i32
...
```

For a 32 bits PowerPC guest, the initial `stw` guest instruction gets
translated into QEMU TCG *frontend-op*
[`tcg_gen_qemu_st_i32`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/tcg-op.c#L2845):

```c
void tcg_gen_qemu_st_i32(TCGv_i32 val, TCGv addr, TCGArg idx, TCGMemOp memop)
{
...
    gen_ldst_i32(INDEX_op_qemu_st_i32, val, addr, memop, idx);
...
}
```

From that point, we have an IR qemu_st_i32 opcode which is emitted.



### Translating IR `qemu_st_i32` into Intel x86_64 for execution

We won't explain host code generation again, read the [dedicated blog
post](tcg_p2.md). Once the execution loop reaches
[`tcg_gen_code`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/tcg.c#L4013)
and more specifically
[`tcg_reg_alloc_op`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/tcg.c#L3575),
QEMU generates the TCG *backend-op* for `qemu_st_i32`.

```c
static void tcg_reg_alloc_op(TCGContext *s, const TCGOp *op)
{
...
     tcg_out_op(s, op->opc, new_args, const_args);
...

}
```

In our situation, the *tcg-target* is an Intel x86 machine, so we will
find suitable `tcg_out_op` definition in
[`tcg/i386/tcg-target.inc.c`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/i386/tcg-target.inc.c#L2255)

```c
static inline void tcg_out_op(TCGContext *s, TCGOpcode opc,
                              const TCGArg *args, const int *const_args)
{
...
    case INDEX_op_qemu_st_i32:
        tcg_out_qemu_st(s, args, 0);
...
}
```

Here we are. The
[`tcg_out_qemu_st`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/i386/tcg-target.inc.c#L2219)
is a very interesting function to study. It holds the internals of
QEMU guest memory addressing.


## Resolving host address

The blog series does not intend to be a complete guide to the TCG
internals. Keep in mind, that at this level, functions are developped
with some conventions related to TCG arguments (ie. `TCGArg *args`).

```c
static void tcg_out_qemu_st(TCGContext *s, const TCGArg *args, bool is64)
{
    TCGReg datalo, datahi, addrlo;
    TCGReg addrhi __attribute__((unused));
    TCGMemOpIdx oi;
    MemOp opc;
#if defined(CONFIG_SOFTMMU)
    int mem_index;
    tcg_insn_unit *label_ptr[2];
#endif

    datalo = *args++;
    datahi = (TCG_TARGET_REG_BITS == 32 && is64 ? *args++ : 0);
    addrlo = *args++;
    addrhi = (TARGET_LONG_BITS > TCG_TARGET_REG_BITS ? *args++ : 0);
    oi = *args++;
    opc = get_memop(oi);

#if defined(CONFIG_SOFTMMU)
    mem_index = get_mmuidx(oi);

    tcg_out_tlb_load(s, addrlo, addrhi, mem_index, opc,
                     label_ptr, offsetof(CPUTLBEntry, addr_write));

    /* TLB Hit.  */
    tcg_out_qemu_st_direct(s, datalo, datahi, TCG_REG_L1, -1, 0, 0, opc);

    /* Record the current context of a store into ldst label */
    add_qemu_ldst_label(s, false, is64, oi, datalo, datahi, addrlo, addrhi,
                        s->code_ptr, label_ptr);
#else
    tcg_out_qemu_st_direct(s, datalo, datahi, addrlo, x86_guest_base_index,
                           x86_guest_base_offset, x86_guest_base_seg, opc);
#endif
}
```

Thanks to the *soft-mmu* and support of virtual TLBs, QEMU offers a
slow path and a fast path when accessing guest memory. The slow path
can be seen as a *TLB-miss* and implies a subsequent call to the
PowerPC software MMU implemented inside QEMU to translate a guest
virtual address into a guest physical address.

If there is a *TLB-hit*, QEMU already holds the guest physical address
in its vCPU maintained TLBs and is able to directly generate the final
memory access into guest RAM with an Intel x86 instruction. Have a
look at
[`tcg_out_qemu_st_direct`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/i386/tcg-target.inc.c#L2138).

The mechanic behind that is tied to the following 3 lines:

```c
/* try to find a filled TLB entry */
tcg_out_tlb_load(s, addrlo, addrhi, mem_index, opc,
                 label_ptr, offsetof(CPUTLBEntry, addr_write));

/* TLB Hit. So generate a physical guest memory access */
tcg_out_qemu_st_direct(s, datalo, datahi, TCG_REG_L1, -1, 0, 0, opc);

/* TLB Miss. Filled during tlb_load and redirect to soft-MMU */
add_qemu_ldst_label(s, false, is64, oi, datalo, datahi, addrlo, addrhi,
                    s->code_ptr, label_ptr);
```

### QEMU vCPU TLBs

First,
[`tcg_out_tlb_load`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/i386/tcg-target.inc.c#L1699)
will generate *host instructions* to check for a TLB entry. The QEMU
TLBs are generic to the architecture and defined at
[cpu-defs.h](https://github.com/qemu/qemu/blob/v4.2.0/include/exec/cpu-defs.h#L109):

```c
typedef struct CPUTLBEntry {
    /* bit TARGET_LONG_BITS to TARGET_PAGE_BITS : virtual address
       bit TARGET_PAGE_BITS-1..4  : Nonzero for accesses that should not
                                    go directly to ram.
       bit 3                      : indicates that the entry is invalid
       bit 2..0                   : zero
    */
    union {
        struct {
            target_ulong addr_read;
            target_ulong addr_write;
            target_ulong addr_code;
            /* Addend to virtual address to get host address.  IO accesses
               use the corresponding iotlb value.  */
            uintptr_t addend;
        };
        /* padding to get a power of two size */
        uint8_t dummy[1 << CPU_TLB_ENTRY_BITS];
    };
} CPUTLBEntry;
```

As translated blocks are generated once and executed several times
(potentially), it is convenient to generate host code that will
dynamically check for QEMU maintained vCPU TLBs. During the life of a
translated block, a given TLB entry might be invalidated then filled
again.

The
[`tcg_out_tlb_load`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/i386/tcg-target.inc.c#L1699)
code is quite annoying to read, full of *tcg-target* opcodes
generators. A resulting Intel x86_64 translated block containing the
`tlb_load` algorithm looks like the following:

```asm
tcg_out_tlb_load:
0x7ffff41888e9 <code_gen_buffer+22716>:	mov    %esp,%edi
0x7ffff41888eb <code_gen_buffer+22718>:	shr    $0x7,%edi
0x7ffff41888ee <code_gen_buffer+22721>:	and    0x338(%rbp),%edi
0x7ffff41888f4 <code_gen_buffer+22727>:	add    0x388(%rbp),%rdi
0x7ffff41888fb <code_gen_buffer+22734>:	lea    0x3(%r12),%esi
0x7ffff4188900 <code_gen_buffer+22739>:	and    $0xfffff000,%esi
0x7ffff4188906 <code_gen_buffer+22745>:	cmp    0x4(%rdi),%esi
0x7ffff4188909 <code_gen_buffer+22748>:	mov    %r12d,%esi
0x7ffff418890c <code_gen_buffer+22751>:	jne    0x7ffff418897f  ---> back to LDST labels
0x7ffff4188912 <code_gen_buffer+22757>:	add    0x10(%rdi),%rsi
```

QEMU tries to read its `CPUTLBEntries` for the given guest virtual
address. For a store operation, the `addr_write` field is used for
comparaison at `0x7ffff4188906` in the extract.  The `RDI` register
points to the `CPUTLBEntry` and `ESI` holds the guest address.

If the comparaison fails, it's a *TLB-miss* and we jump to a *LDST
label* that we will explain later. Else, the `RSI` register is
adjusted to the final host address thanks to the `CPUTLBEntry.addend`
and the memory access can be done:

```asm
tcg_out_qemu_st_direct:
0x7ffff4188925 <code_gen_buffer+22776>:	movbe  %ebx,(%rsi)
```

The TLB verification implementation can also be found in the QEMU
`cpu_ld/st_xxx` API functions. As defined in the
[documentation](https://github.com/qemu/qemu/blob/v4.2.0/docs/devel/loads-stores.rst),
they operate on guest virtual addresses and **may cause guest CPU
exception**. Thus they do check TLBs and might redirect to the
software MMU. They are implemented through macros in
[cpu_ldst_template.h](https://github.com/qemu/qemu/blob/v4.2.0/include/exec/cpu_ldst_template.h):

```c
/* generic store macro */

static inline void
glue(glue(glue(cpu_st, SUFFIX), MEMSUFFIX), _ra)(CPUArchState *env,
                                                 target_ulong ptr,
                                                 RES_TYPE v, uintptr_t retaddr)
{
...
    addr = ptr;
    mmu_idx = CPU_MMU_INDEX;
    entry = tlb_entry(env, mmu_idx, addr);
    if (unlikely(tlb_addr_write(entry) !=
                 (addr & (TARGET_PAGE_MASK | (DATA_SIZE - 1))))) {
        oi = make_memop_idx(SHIFT, mmu_idx);
        glue(glue(helper_ret_st, SUFFIX), MMUSUFFIX)(env, addr, v, oi,
                                                     retaddr);
    } else {
        uintptr_t hostaddr = addr + entry->addend;
        glue(glue(st, SUFFIX), _p)((uint8_t *)hostaddr, v);
    }
...
}
```

Looks familiar to you, no :) ?


### QEMU LDST labels

The [`LDST
labels`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/tcg-ldst.inc.c#L23)
stand for *load/store labels*. It is the mechanism used by QEMU to
redirect a *TLB-miss* to a call to the software MMU via a TCG helper.

The
[`TCGContext`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/tcg.h#L583)
object holds the output buffer that receive generated *host assembly*
opcodes. Any time an instruction is added, the `code_ptr` pointer to
that buffer is obviously incremented.

During
[`tcg_out_tlb_load`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/i386/tcg-target.inc.c#L1699),
when QEMU generates the comparaison and jump instructions for a
*TLB-miss*, it also records the location into the output buffer that
will hold the `JNE` offset to redirect execution to.

```c
static inline void tcg_out_tlb_load(TCGContext *s, TCGReg addrlo, TCGReg addrhi,
                                    int mem_index, MemOp opc,
                                    tcg_insn_unit **label_ptr, int which)
{
...
    /* jne slow_path */
    tcg_out_opc(s, OPC_JCC_long + JCC_JNE, 0, 0, 0);
    label_ptr[0] = s->code_ptr;
    s->code_ptr += 4;
...
}
```

Additionally, when we get back to
[`tcg_out_qemu_st`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/i386/tcg-target.inc.c#L2247),
the call to
[`add_qemu_ldst_label`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/i386/tcg-target.inc.c#L1784)
creates a new LDST label and records context details to later prepare
a call to the softmmu *slow path* TCG helper:

```c
static void add_qemu_ldst_label(TCGContext *s, bool is_ld, bool is_64,
                                TCGMemOpIdx oi,
                                TCGReg datalo, TCGReg datahi,
                                TCGReg addrlo, TCGReg addrhi,
                                tcg_insn_unit *raddr,
                                tcg_insn_unit **label_ptr)
{
    TCGLabelQemuLdst *label = new_ldst_label(s);

    label->is_ld = is_ld;
    label->oi = oi;
    label->type = is_64 ? TCG_TYPE_I64 : TCG_TYPE_I32;
    label->datalo_reg = datalo;
    label->datahi_reg = datahi;
    label->addrlo_reg = addrlo;
    label->addrhi_reg = addrhi;
    label->raddr = raddr;
    label->label_ptr[0] = label_ptr[0];
    if (TARGET_LONG_BITS > TCG_TARGET_REG_BITS) {
        label->label_ptr[1] = label_ptr[1];
    }
}
```

The call to the *slow path* helper is inserted by
[`tcg_gen_code`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/tcg.c#L4013)
during translated block epilogue generation with a call to
[`tcg_out_ldst_finalize`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/tcg-ldst.inc.c#L44). QEMU
checks if there exists *LDST labels* and generates the corresponding
TCG helper calls. For our store operation,
[`tcg_out_ldst_finalize`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/tcg-ldst.inc.c#L44)
will emit a
[`tcg_out_qemu_st_slow_path`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/i386/tcg-target.inc.c#L1895):


```c
int tcg_gen_code(TCGContext *s, TranslationBlock *tb)
{
...
#ifdef TCG_TARGET_NEED_LDST_LABELS
    i = tcg_out_ldst_finalize(s);
    if (i < 0) {
        return i;
    }
#endif
...
}

static int tcg_out_ldst_finalize(TCGContext *s)
{
...
    /* qemu_ld/st slow paths */
    QSIMPLEQ_FOREACH(lb, &s->ldst_labels, next) {
        if (lb->is_ld
            ? !tcg_out_qemu_ld_slow_path(s, lb)
            : !tcg_out_qemu_st_slow_path(s, lb)) {
            return -2;
        }
...
}

static bool tcg_out_qemu_st_slow_path(TCGContext *s, TCGLabelQemuLdst *l)
{
...
    /* "Tail call" to the helper, with the return address back inline.  */
    tcg_out_push(s, retaddr);
    tcg_out_jmp(s, qemu_st_helpers[opc & (MO_BSWAP | MO_SIZE)]);
    return true;
}
```

The preparation of the *slow path* helper call implies:
- resolving the label offset, previously recorded during `JNE` generation
- arguments/registers setup
- call to the appropriate
[`qemu_st_helper`](https://github.com/qemu/qemu/blob/v4.2.0/tcg/i386/tcg-target.inc.c#L1668)

Like others memory access functions, there exists a lot of variants
for store and load operations: *signed, unsigned, byte, word, long,
big or little endian*. Each one has a specific TCG helper which is a
wrapper to
[`store_helper`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cputlb.c#L1663):


```c
/* helper signature: helper_ret_st_mmu(CPUState *env, target_ulong addr,
 *                                     uintxx_t val, int mmu_idx, uintptr_t ra)
 */
static void * const qemu_st_helpers[16] = {
    [MO_UB]   = helper_ret_stb_mmu,
    [MO_LEUW] = helper_le_stw_mmu,
    [MO_LEUL] = helper_le_stl_mmu,
    [MO_LEQ]  = helper_le_stq_mmu,
    [MO_BEUW] = helper_be_stw_mmu,
    [MO_BEUL] = helper_be_stl_mmu,
    [MO_BEQ]  = helper_be_stq_mmu,
};

void helper_be_stl_mmu(CPUArchState *env, target_ulong addr, uint32_t val,
                       TCGMemOpIdx oi, uintptr_t retaddr)
{
    store_helper(env, addr, val, oi, retaddr, MO_BEUL);
}

static inline void QEMU_ALWAYS_INLINE
store_helper(CPUArchState *env, target_ulong addr, uint64_t val,
             TCGMemOpIdx oi, uintptr_t retaddr, MemOp op)
{
...
        if (!tlb_hit_page(tlb_addr2, page2)) {
            if (!victim_tlb_hit(env, mmu_idx, index2, tlb_off, page2)) {
                tlb_fill(env_cpu(env), page2, size2, MMU_DATA_STORE,
                         mmu_idx, retaddr);
                index2 = tlb_index(env, mmu_idx, page2);
                entry2 = tlb_entry(env, mmu_idx, page2);
            }
            tlb_addr2 = tlb_addr_write(entry2);
        }
...
    haddr = (void *)((uintptr_t)addr + entry->addend);
    store_memop(haddr, val, op);
}
```

The helper checks TLBs, if there is a miss it calls
[`tlb_fill`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cputlb.c#L900). In
any case, once the address is resolved it does the final memory access
in the host memory with
[`store_memop`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cputlb.c#L1633).

```c
static void tlb_fill(CPUState *cpu, target_ulong addr, int size,
                     MMUAccessType access_type, int mmu_idx, uintptr_t retaddr)
{
    CPUClass *cc = CPU_GET_CLASS(cpu);
    bool ok;

    ok = cc->tlb_fill(cpu, addr, size, access_type, mmu_idx, false, retaddr);
    assert(ok);
}
```

As you may guess, the process of *filling a TLB* is done by the
software MMU whose implementation depends on the emulated
architecture. During the [PowerPC CPU
initialisation](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/translate_init.inc.c#L10662)
,
[`cc->tlb_fill`](https://github.com/qemu/qemu/blob/v4.2.0/accel/tcg/cputlb.c#L900)
is set to
[`ppc_cpu_tlb_fill`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/mmu_helper.c#L3041).

```c
bool ppc_cpu_tlb_fill(CPUState *cs, vaddr addr, int size,
                      MMUAccessType access_type, int mmu_idx,
                      bool probe, uintptr_t retaddr)
{
    PowerPCCPU *cpu = POWERPC_CPU(cs);
    PowerPCCPUClass *pcc = POWERPC_CPU_GET_CLASS(cs);
    CPUPPCState *env = &cpu->env;
    int ret;

    if (pcc->handle_mmu_fault) {
        ret = pcc->handle_mmu_fault(cpu, addr, access_type, mmu_idx);
    } else {
        ret = cpu_ppc_handle_mmu_fault(env, addr, access_type, mmu_idx);
    }
    if (unlikely(ret != 0)) {
        if (probe) {
            return false;
        }
        raise_exception_err_ra(env, cs->exception_index, env->error_code,
                               retaddr);
    }
    return true;
}
```

And
[`ppc->handle_mmu_fault`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/translate_init.inc.c#L5319)
might eventually be set to
[`ppc_hash32_handle_mmu_fault`](https://github.com/qemu/qemu/blob/v4.2.0/target/ppc/mmu-hash32.c#L415),
depending on your PowerPC CPU family. This is where the software MMU
implementation lies for the PowerPC.
