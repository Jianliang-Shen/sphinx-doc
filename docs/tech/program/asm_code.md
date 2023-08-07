# 汇编，🐶都不写

这是一段v8m mainline的汇编代码

```c
__naked void arch_non_preempt_call(uintptr_t fn_addr, uintptr_t frame_addr,
                                   uint32_t stk_base, uint32_t stk_limit)
{
    __asm volatile(
        SYNTAX_UNIFIED
        "   push   {r4-r6, lr}                      \n"
        "   cpsid  i                                \n"
        "   isb                                     \n"
        "   mov    r4, r2                           \n"
        "   mrs    r5, psplim                       \n"
        "   movs   r12, #0                          \n"
        "   cmp    r2, #0                           \n"
        "   itttt  ne                               \n" /* To callee stack */
        "   msrne  psplim, r12                      \n"
        "   movne  r4, sp                           \n"
        "   movne  sp, r2                           \n"
        "   msrne  psplim, r3                       \n"
        "   ldr    r2, =scheduler_lock              \n" /* To lock sched */
        "   movs   r3, #"M2S(SCHEDULER_LOCKED)"     \n"
        "   str    r3, [r2, #0]                     \n"
        "   cpsie  i                                \n"
        "   isb                                     \n"
        "   mov    r6, r1                           \n"
        "   bl     cross_call_entering_c            \n"
        "   cpsid  i                                \n"
        "   isb                                     \n"
        "   mov    r1, r6                           \n"
        "   bl     cross_call_exiting_c             \n"
        "   movs   r12, #0                          \n"
        "   cmp    r4, #0                           \n"
        "   ittt   ne                               \n" /* To caller stack */
        "   msrne  psplim, r12                      \n"
        "   movne  sp, r4                           \n"
        "   msrne  psplim, r5                       \n"
        "   cpsie  i                                \n"
        "   isb                                     \n"
        "   pop    {r4-r6, pc}                      \n"
    );
}
```

```c
__naked
void arch_cross_call_entry(uint32_t a0, uint32_t a1, uint32_t a2, uint32_t a3)
{
    __asm volatile(
        SYNTAX_UNIFIED
        "   push    {r4-r6, lr}             \n"
        "   mov     r4, r12                 \n"
        "   push    {r0-r5}                 \n"
        "   cpsid   i                       \n"
        "   isb                             \n"
        "   bl      cross_call_entering_c   \n" /* r0: new SP, r1: new PSPLIM */
        "   mov     r6, sp                  \n"
        "   mov     r4, r0                  \n"
        "   mrs     r5, psplim              \n"
        "   cmp     r4, #0                  \n"
        "   itttt   ne                      \n"
        "   movsne  r2, #0                  \n"
        "   msrne   psplim, r2              \n"
        "   movne   sp, r0                  \n" /* To SPM stack */
        "   msrne   psplim, r1              \n"
        "   cpsie   i                       \n"
        "   isb                             \n"
        "   push    {r4, r5}                \n"
        "   ldmia   r6!, {r0-r5}            \n" /* Load PSA interface input args
                                                 * and target function
                                                 */
        "   blx     r4                      \n"
        "   cpsid   i                       \n"
        "   isb                             \n"
        "   bl      cross_call_exiting_c    \n"
        "   pop     {r4, r5}                \n"
        "   cmp     r4, #0                  \n"
        "   ittt    ne                      \n"
        "   movsne  r2, #0                  \n"
        "   msrne   psplim, r2              \n"
        "   msrne   psplim, r5              \n" /* To caller stack */
        "   mov     sp, r6                  \n"
        "   cpsie   i                       \n"
        "   isb                             \n"
        "   pop     {r4-r6, pc}             \n"
    );
}
```
