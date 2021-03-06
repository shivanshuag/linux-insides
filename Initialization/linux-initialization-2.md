Kernel initialization. Part 2.
================================================================================

Early interrupt and exception handling
--------------------------------------------------------------------------------

In the previous [part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html) we stopped before setting of early interrupt handlers. We continue in this part and will know more about interrupt and exception handling.

Remember that we stopped before following loop:

```C
 	for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
		set_intr_gate(i, early_idt_handlers[i]);
```

from the [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c) source code file. But before we started to sort out this code, we need to know about interrupts and handlers.

Some theory
--------------------------------------------------------------------------------

Interrupt is an event caused by software or hardware to the CPU. On interrupt, CPU stops the current task and transfer control to the interrupt handler, which handles interruption and transfer control back to the previously stopped task. We can split interrupts on three types:

* Software interrupts - when a software signals CPU that it needs kernel attention. These interrupts are generally used for system calls;
* Hardware interrupts - when a hardware event happens, for example button is pressed on a keyboard;
* Exceptions - interrupts generated by CPU, when the CPU detects error, for example division by zero or accessing a memory page which is not in RAM.

Every interrupt and exception is assigned a unique number which called - `vector number`. `Vector number` can be any number from `0` to `255`. There is common practice to use first `32` vector numbers for exceptions, and vector numbers from `32` to `255` are used for user-defined interrupts. We can see it in the code above - `NUM_EXCEPTION_VECTORS`, which defined as:

```C
#define NUM_EXCEPTION_VECTORS 32
```

CPU uses vector number as an index in the `Interrupt Descriptor Table` (we will see description of it soon). CPU catch interrupts from the [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) or through it's pins. Following table shows `0-31` exceptions:

```
----------------------------------------------------------------------------------------------
|Vector|Mnemonic|Description         |Type |Error Code|Source                                |
----------------------------------------------------------------------------------------------
|0     | #DE    |Divide Error        |Fault|NO        |DIV and IDIV                          |
|---------------------------------------------------------------------------------------------
|1     | #DB    |Reserved            |F/T  |NO        |                                      |
|---------------------------------------------------------------------------------------------
|2     | ---    |NMI                 |INT  |NO        |external NMI                          |
|---------------------------------------------------------------------------------------------
|3     | #BP    |Breakpoint          |Trap |NO        |INT 3                                 |
|---------------------------------------------------------------------------------------------
|4     | #OF    |Overflow            |Trap |NO        |INTO  instruction                     |
|---------------------------------------------------------------------------------------------
|5     | #BR    |Bound Range Exceeded|Fault|NO        |BOUND instruction                     |
|---------------------------------------------------------------------------------------------
|6     | #UD    |Invalid Opcode      |Fault|NO        |UD2 instruction                       |
|---------------------------------------------------------------------------------------------
|7     | #NM    |Device Not Available|Fault|NO        |Floating point or [F]WAIT             |
|---------------------------------------------------------------------------------------------
|8     | #DF    |Double Fault        |Abort|YES       |Ant instrctions which can generate NMI|
|---------------------------------------------------------------------------------------------
|9     | ---    |Reserved            |Fault|NO        |                                      |
|---------------------------------------------------------------------------------------------
|10    | #TS    |Invalid TSS         |Fault|YES       |Task switch or TSS access             |
|---------------------------------------------------------------------------------------------
|11    | #NP    |Segment Not Present |Fault|NO        |Accessing segment register            |
|---------------------------------------------------------------------------------------------
|12    | #SS    |Stack-Segment Fault |Fault|YES       |Stack operations                      |
|---------------------------------------------------------------------------------------------
|13    | #GP    |General Protection  |Fault|YES       |Memory reference                      |
|---------------------------------------------------------------------------------------------
|14    | #PF    |Page fault          |Fault|YES       |Memory reference                      |
|---------------------------------------------------------------------------------------------
|15    | ---    |Reserved            |     |NO        |                                      |
|---------------------------------------------------------------------------------------------
|16    | #MF    |x87 FPU fp error    |Fault|NO        |Floating point or [F]Wait             |
|---------------------------------------------------------------------------------------------
|17    | #AC    |Alignment Check     |Fault|YES       |Data reference                        |
|---------------------------------------------------------------------------------------------
|18    | #MC    |Machine Check       |Abort|NO        |                                      |
|---------------------------------------------------------------------------------------------
|19    | #XM    |SIMD fp exception   |Fault|NO        |SSE[2,3] instructions                 |
|---------------------------------------------------------------------------------------------
|20    | #VE    |Virtualization exc. |Fault|NO        |EPT violations                        |
|---------------------------------------------------------------------------------------------
|21-31 | ---    |Reserved            |INT  |NO        |External interrupts                   |
----------------------------------------------------------------------------------------------
```

To react on interrupt CPU uses special structure - Interrupt Descriptor Table or IDT. IDT is an array of 8-byte descriptors like Global Descriptor Table, but IDT entries are called `gates`. CPU multiplies vector number on 8 to find index of the IDT entry. But in 64-bit mode IDT is an array of 16-byte descriptors and CPU multiplies vector number on 16 to find index of the entry in the IDT. We remember from the previous part that CPU uses special `GDTR` register to locate Global Descriptor Table, so CPU uses special register `IDTR` for Interrupt Descriptor Table and `lidt` instruuction for loading base address of the table into this register.

64-bit mode IDT entry has following structure:

```
127                                                                             96
 --------------------------------------------------------------------------------
|                                                                               |
|                                Reserved                                       |
|                                                                               |
 --------------------------------------------------------------------------------
95                                                                              64
 --------------------------------------------------------------------------------
|                                                                               |
|                               Offset 63..32                                   |
|                                                                               |
 --------------------------------------------------------------------------------
63                               48 47      46  44   42    39             34    32
 --------------------------------------------------------------------------------
|                                  |       |  D  |   |     |      |   |   |     |
|       Offset 31..16              |   P   |  P  | 0 |Type |0 0 0 | 0 | 0 | IST |
|                                  |       |  L  |   |     |      |   |   |     |
 --------------------------------------------------------------------------------
31                                   15 16                                      0
 --------------------------------------------------------------------------------
|                                      |                                        |
|          Segment Selector            |                 Offset 15..0           |
|                                      |                                        |
 --------------------------------------------------------------------------------
```

Where:

* Offset - is offset to entry point of an interrupt handler;
* DPL -    Descriptor Privilege Level;
* P -      Segment Present flag;
* Segment selector - a code segment selector in GDT or LDT
* IST -    provides ability to switch to a new stack for interrupts handling.

And the last `Type` field describes type of the `IDT` entry. There are three different kinds of handlers for interrupts:

* Task descriptor
* Interrupt descriptor
* Trap descriptor

Interrupt and trap descriptors contain a far pointer to the entry point of the interrupt handler. Only one difference between these types is how CPU handles `IF` flag. If interrupt handler was accessed through interrupt gate, CPU clear the `IF` flag to prevent other interrupts while current interrupt handler executes. After that current interrupt handler executes, CPU sets the `IF` flag again with `iret` instruction. 

Other bits reserved and must be 0.

Now let's look how CPU handles interrupts:

* CPU save flags register, `CS`, and instruction pointer on the stack.
* If interrupt causes an error code (like `#PF` for example), CPU saves an error on the stack after instruction pointer;
* After interrupt handler executed, `iret` instruction used to return from it.

Now let's back to code.

Fill and load IDT
--------------------------------------------------------------------------------

We stopped at the following point:

```C
for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
	set_intr_gate(i, early_idt_handlers[i]);
```

Here we call `set_intr_gate` in the loop, which takes two parameters:

* Number of an interrupt;
* Address of the idt handler.

and inserts an interrupt gate in the nth `IDT` entry. First of all let's look on the `early_idt_handlers`. It is an array which contains address of the first 32 interrupt handlers:

```C
extern const char early_idt_handlers[NUM_EXCEPTION_VECTORS][2+2+5];
```

We're filling only first 32 IDT entries because all of the early setup runs with interrupts disabled, so there is no need to set up early exception handlers for vectors greater than 32. `early_idt_handlers` contains generic idt handlers and we can find it in the [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S), we will look it soon.

Now let's look on `set_intr_gate` implementation:

```C
#define set_intr_gate(n, addr)                                           \
         do {                                                            \
                 BUG_ON((unsigned)n > 0xFF);                             \
                 _set_gate(n, GATE_INTERRUPT, (void *)addr, 0, 0,        \
                           __KERNEL_CS);                                 \
                 _trace_set_gate(n, GATE_INTERRUPT, (void *)trace_##addr,\
                                 0, 0, __KERNEL_CS);                     \
         } while (0)
```

First of all it checks with that passed interrupt number is not greater than `255` with `BUG_ON` macro. We need to do this check because we can have only 256 interrupts. After this it calls `_set_gate` which writes address of an interrupt gate to the `IDT`:


```C
static inline void _set_gate(int gate, unsigned type, void *addr,
	                         unsigned dpl, unsigned ist, unsigned seg)
{
         gate_desc s;
         pack_gate(&s, type, (unsigned long)addr, dpl, ist, seg);
         write_idt_entry(idt_table, gate, &s);
         write_trace_idt_entry(gate, &s);
}
```

At the start of `_set_gate` function we can see call of the `pack_gate` function which fills `gate_desc` structure with the given values:

```C
static inline void pack_gate(gate_desc *gate, unsigned type, unsigned long func,
                             unsigned dpl, unsigned ist, unsigned seg)
{
        gate->offset_low        = PTR_LOW(func);
        gate->segment           = __KERNEL_CS;
        gate->ist               = ist;
        gate->p                 = 1;
        gate->dpl               = dpl;
        gate->zero0             = 0;
        gate->zero1             = 0;
        gate->type              = type;
        gate->offset_middle     = PTR_MIDDLE(func);
        gate->offset_high       = PTR_HIGH(func);
}
```

As mentioned above we fill gate descriptor in this function. We fill three parts of the address of the interrupt handler with the address which we got in the main loop (address of the interrupt handler entry point). We are using three following macro to split address on three parts:

```C
#define PTR_LOW(x) ((unsigned long long)(x) & 0xFFFF)
#define PTR_MIDDLE(x) (((unsigned long long)(x) >> 16) & 0xFFFF)
#define PTR_HIGH(x) ((unsigned long long)(x) >> 32)
```

With the first `PTR_LOW` macro we get the first 2 bytes of the address, with the second `PTR_MIDDLE` we get the second 2 bytes of the address and with the third `PTR_HIGH` macro we get the last 4 bytes of the address. Next we setup the segment selector for interrupt handler, it will be our kernel code segment - `__KERNEL_CS`. In the next step we fill `Interrupt Stack Table` and `Descriptor Privilege Level` (highest privilege level) with zeros. And we set `GAT_INTERRUPT` type in the end. 

Now we have filled IDT entry and we can call `native_write_idt_entry` function which just copies filled `IDT` entry to the `IDT`:

```C
static inline void native_write_idt_entry(gate_desc *idt, int entry, const gate_desc *gate)
{
        memcpy(&idt[entry], gate, sizeof(*gate));
}
```

After that main loop will finished, we will have filled `idt_table` array of `gate_desc` structures and we can load `IDT` with:

```C
load_idt((const struct desc_ptr *)&idt_descr);
```

Where `idt_descr` is:

```C
struct desc_ptr idt_descr = { NR_VECTORS * 16 - 1, (unsigned long) idt_table };
```

and `load_idt` just executes `lidt` instruction:

```C
asm volatile("lidt %0"::"m" (*dtr));
```

You can note that there are calls of the `_trace_*` functions in the `_set_gate` and other functions. These functions fills `IDT` gates in the same manner that `_set_gate` but with one difference. These functions use `trace_idt_table` Interrupt Descriptor Table instead of `idt_table` for tracepoints (we will cover this theme in the another part).

Okay, now we have filled and loaded Interrupt Descriptor Table, we know how the CPU acts during interrupt. So now time to deal with interrupts handlers.

Early interrupts handlers
--------------------------------------------------------------------------------

As you can read above, we filled `IDT` with the address of the `early_idt_handlers`. We can find it in the [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S):

```assembly
	.globl early_idt_handlers
early_idt_handlers:
	i = 0
	.rept NUM_EXCEPTION_VECTORS
	.if (EXCEPTION_ERRCODE_MASK >> i) & 1
	ASM_NOP2
	.else
	pushq $0
	.endif
	pushq $i
	jmp early_idt_handler
	i = i + 1
	.endr
```

We can see here, interrupt handlers generation for the first 32 exceptions. We check here, if exception has error code then we do nothing, if exception does not return error code, we push zero to the stack. We do it for that would stack was uniform. After that we push exception number on the stack and jump on the `early_idt_handler` which is generic interrupt handler for now. As i wrote above, CPU pushes flag register, `CS` and `RIP` on the stack. So before `early_idt_handler` will be executed, stack will contain following data:

```
|--------------------|
| %rflags            |
| %cs                |
| %rip               |
| rsp --> error code |
|--------------------|
```

Now let's look on the `early_idt_handler` implementation. It locates in the same [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S#L343). First of all we can see check for [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt), we no need to handle it, so just ignore they in the `early_idt_handler`:

```assembly
	cmpl $2,(%rsp)
	je is_nmi
```

where `is_nmi`:

```assembly
is_nmi:
	addq $16,%rsp
	INTERRUPT_RETURN
```

we drop error code and vector number from the stack and call `INTERRUPT_RETURN` which is just `iretq`. As we checked the vector number and it is not `NMI`, we check `early_recursion_flag` to prevent recursion in the `early_idt_handler` and if it's correct we save general registers on the stack:

```assembly
	pushq %rax
	pushq %rcx
	pushq %rdx
	pushq %rsi
	pushq %rdi
	pushq %r8
	pushq %r9
	pushq %r10
	pushq %r11
```

we need to do it to prevent wrong values in it when we return from the interrupt handler. After this we check segment selector in the stack:

```assembly
	cmpl $__KERNEL_CS,96(%rsp)
	jne 11f
```

it must be equal to the kernel code segment and if it is not we jump on label `11` which prints `PANIC` message and makes stack dump.

After code segment was checked, we check the vector number, and if it is `#PF`, we put value from the `cr2` to the `rdi` register and call `early_make_pgtable` (well see it soon):

```assembly
	cmpl $14,72(%rsp)
	jnz 10f
	GET_CR2_INTO(%rdi)
	call early_make_pgtable
	andl %eax,%eax
	jz 20f
```

If vector number is not `#PF`, we restore general purpose registers from the stack:

```assembly
	popq %r11
	popq %r10
	popq %r9
	popq %r8
	popq %rdi
	popq %rsi
	popq %rdx
	popq %rcx
	popq %rax
```

and exit from the handler with `iret`.

It is the end of the first interrupt handler. Note that it is very early interrupt handler, so it handles only Page Fault now. We will see handlers for the other interrupts, but now let's look on the page fault handler.

Page fault handling
--------------------------------------------------------------------------------

In the previous paragraph we saw first early interrupt handler which checks interrupt number for page fault and calls `early_make_pgtable` for building new page tables if it is. We need to have `#PF` handler in this step because there are plans to add ability to load kernel above 4G and make access to `boot_params` structure above the 4G.

You can find implementation of the `early_make_pgtable` in the [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c) and takes one parameter - address from the `cr2` register, which caused Page Fault. Let's look on it:

```C
int __init early_make_pgtable(unsigned long address)
{
	unsigned long physaddr = address - __PAGE_OFFSET;
	unsigned long i;
	pgdval_t pgd, *pgd_p;
	pudval_t pud, *pud_p;
	pmdval_t pmd, *pmd_p;
	...
	...
	...
}
```

It starts from the definition of some variables which have `*val_t` types. All of these types are just:

```C
typedef unsigned long   pgdval_t;
```

Also we will operate with the `*_t` (not val) types, for example `pgd_t` and etc... All of these types defined in the [arch/x86/include/asm/pgtable_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/pgtable_types.h) and represent structures like this:

```C
typedef struct { pgdval_t pgd; } pgd_t;
```

For example,

```C
extern pgd_t early_level4_pgt[PTRS_PER_PGD];
```

Here `early_level4_pgt` presents early top-level page table directory which consists of an array of `pgd_t` types and `pgd` points to low-level page entries.

After we made the check that we have no invalid address, we're getting the address of the Page Global Directory entry which contains `#PF` address and put it's value to the `pgd` variable:

```C
pgd_p = &early_level4_pgt[pgd_index(address)].pgd;
pgd = *pgd_p;
```

In the next step we check `pgd`, if it contains correct page global directory entry we put physical address of the page global directory entry and put it to the `pud_p` with:

```C
pud_p = (pudval_t *)((pgd & PTE_PFN_MASK) + __START_KERNEL_map - phys_base);
```

where `PTE_PFN_MASK` is a macro:

```C
#define PTE_PFN_MASK            ((pteval_t)PHYSICAL_PAGE_MASK)
```

which expands to:

```C
(~(PAGE_SIZE-1)) & ((1 << 46) - 1)
```

or

```
0b1111111111111111111111111111111111111111111111
```

which is 46 bits to mask page frame.

If `pgd` does not contain correct address we check that `next_early_pgt` is not greater than `EARLY_DYNAMIC_PAGE_TABLES` which is `64` and present a fixed number of buffers to set up new page tables on demand. If `next_early_pgt` is greater than `EARLY_DYNAMIC_PAGE_TABLES` we reset page tables and start again. If `next_early_pgt` is less than `EARLY_DYNAMIC_PAGE_TABLES`, we create new page upper directory pointer which points to the current dynamic page table and writes it's physical address with the `_KERPG_TABLE` access rights to the page global directory:

```C
if (next_early_pgt >= EARLY_DYNAMIC_PAGE_TABLES) {
	reset_early_page_tables();
    goto again;
}
	
pud_p = (pudval_t *)early_dynamic_pgts[next_early_pgt++];
for (i = 0; i < PTRS_PER_PUD; i++)
	pud_p[i] = 0;
*pgd_p = (pgdval_t)pud_p - __START_KERNEL_map + phys_base + _KERNPG_TABLE;
```

After this we fix up address of the page upper directory with:

```C
pud_p += pud_index(address);
pud = *pud_p;
```

In the next step we do the same actions as we did before, but with the page middle directory. In the end we fix address of the page middle directory which contains maps kernel text+data virtual addresses:

```C
pmd = (physaddr & PMD_MASK) + early_pmd_flags;
pmd_p[pmd_index(address)] = pmd;
```

After page fault handler finished it's work and as result our `early_level4_pgt` contains entries which point to the valid addresses.

Conclusion
--------------------------------------------------------------------------------

This is the end of the second part about linux kernel insides. If you have questions or suggestions, ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](anotherworldofworld@gmail.com) or just create [issue](https://github.com/0xAX/linux-internals/issues/new). In the next part we will see all steps before kernel entry point - `start_kernel` function.

**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [GNU assembly .rept](https://sourceware.org/binutils/docs-2.23/as/Rept.html)
* [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html)
