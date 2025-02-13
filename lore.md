# LoRe: loadtime binary rewrite for software fault isolation

## Abstract 
(Not working)
ARM64 architecture has dominated mobile computing and is gaining traction in personal and cloud computing. The fixed-instruction-length property of ARM64 instruction(or RISC in general) set makes it a natural target for building software fault isolation. In addition, ARMv8.1 introduces PACBTI extension to limit the scope of indirect control flow exploits. We notice that past SFI techniques on 64bit systems divide the 64bit memory space into 4GB aligned compartments and only 16bits are needed for locating a compartment. Therefore, we design a new SFI technique LoRe that rewrites arbitrary BTI-enabled ARM64 binary at load time. LoRe directly operates on closed-source executable file and introduces minimal(hopefully) runtime overhead at the cost of low code size overhead by adding guards before every load/store and indirect branch instructions.

## Introduction

Isolation is a foundational requirement for running unstrusted code, critical for enforcing security, stability, and resource management. Traditional hardware-based isolation mechanisms, such as virtual memory, achieve this by partitioning processes into separate address spaces. However, these approaches incur significant performance penalties due to context-switching overheads, including costly TLB shootdowns and cache misses during domain transitions. Recent CPUs introduce [new hardware isolation mechanisms](https://mars-research.github.io/doc/2024-atc-hw-isolation.pdf) with lower overhead, including but not limited to Intel MPK, ARM PAC, ARM MTE and ARM morello. However, those solutions have the limitation of a small number of isolation domains and IPI cost etc(need to read the paper more carefully)  

Software Fault Isolation (SFI) offers an alternative strategy that avoids such hardware dependencies. By instrumenting low-level machine code, SFI enforces memory and control-flow safety within a single address space, allowing multiple protection domains to coexist efficiently. This technique can be implemented in various ways—via machine-code interpreters, compilers, or machine-code rewriters. Among these, machine-code rewriting stands out as a compelling approach: it minimizes both the trusted computing base (TCB) and runtime overhead by operating directly on compiled binaries, bypassing the need for source code or specialized hardware. Machine code rewriting removes the compiler and rewriter from TCB by checking the rewritten binary with an independent verifier). This work focuses on SFI through machine-code rewriting, analyzing its design principles, enforcement guarantees, and practical trade-offs in modern systems.

A naive machine code rewriter restricts memory-access(read and write) by inserting a dynamic address check before every load and store instruction, at the cost of CPU cycles and code size. Washbe and LFI uses dedicated, reserved registers for memory access, enforcing data-access policy by maintaining two invariants: pointers in dedicated registers are always in range and all memory access instructions only use dedicated registers for addressing. This technique usually reserves one register for holding the base address of the sandbox. LFI incurs zero cycle overhead for basic addressing mode and 1 cycle overhead for other addressing modes.(put code and data in 2 different sandboxes? Modifying code at run time? Exchange heap by writing to the data base register? Stack access is easy to identify but all pointers to the heap on the stack gets invalidated). KFLex use the same scheme as LFI on x86: the guard is implemented with an AND and an ADD(two reserved registers).

In-place Address masking modifies the register state at the cost of one cycle and enables more optimization by static analysis.(address masking can also be used with reserved registers but more instructions is required). In ARM64, in-place address masking can be implemented with a single `movk` instruction(need to read the NACL on Arm paper)(inplace write creates a data dependencies, a 2 cycle stall). However, inline guards may be bypassed by indirect branch. Existing methods for enforcing the control-flow policy includes aligned chunk enforcement and control flow integrity. Both introduce overhead and complexity. The BTI (Branch Target Identification) instruction in ARM64 (introduced in ARMv8.5-A) is a hardware-assisted feature designed to enforce Control-Flow Integrity (CFI) and mitigate code-reuse attacks. BTI ensures that indirect branches (e.g., jumps via function pointers, virtual method calls, or returns) only target valid "branch targets" explicitly marked in the code. This prevents attackers from hijacking control flow by redirecting branches to arbitrary code locations. BTI incurs negligible [overhead](https://newsroom.arm.com/blog/pac-bti) both in code size and runtime performance.(range analysis, minimizing the number of masking instructions needed). 

According to LFI, register hoisting only reduced 1% overhead. Zero-instruction guard contributes most to overhead reduction. 

A sandboxed program lives in a logical 0-based address space. At runtime the actual address is a cancatenation of the logic address and base addressing(sounds like segmenentation). If we have only one sandbox, we can change all 64 bit poiter registers to the 32bit counterpart: ldr x11, [x12] -> ldr w11, [x12] and no masking needed. If all sandboxes live in a logical 0-based address, does it make object sharing easier? Consider a linked list(See KFLEX).

## Limitation of SFI

### Need source code or reliable disassembly(still much engineering effort)

- Need to disable compress jump table(See LFI)

- No vector scatter/gather instructions(See LFI)

### Zero-copy data transfer and stack unwinding

- Need something like capability
- KFLEX: verify kernel interface and object table at cancellation point

### Overhead of instrumentation

- NACL: 5% but designed for only 1 sandbox per address space

- LFI: 7% (1% with no load guard)

### Portability

### Granularity

- Sandbox size(4G too large)
- Ideally, object level granularity

### Interface

#### IDL

#### Sec-comp?

## Idea

- Is it possible to implement the private-shared heap design at machine code level?
    - KFLEX implemented something like this

- A mix of inplace and reserved-register masking: any benefits?






