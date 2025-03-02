<!-- # LoRe: loadtime binary rewrite for software fault isolation -->
# Towards Zero-overhead Software Fault Isolation


## Abstract 
(Skip this part)
<!-- ARM64 architecture has dominated mobile computing and is gaining traction in personal and cloud computing. The fixed-instruction-length property of ARM64 instruction(or RISC in general) set makes it a natural target for building software fault isolation. In addition, ARMv8.1 introduces PACBTI extension to limit the scope of indirect control flow exploits. We notice that past SFI techniques on 64bit systems divide the 64bit memory space into 4GB aligned compartments and only 16bits are needed for locating a compartment. Therefore, we design a new SFI technique LoRe that rewrites arbitrary BTI-enabled ARM64 binary at load time. LoRe directly operates on closed-source executable file and introduces minimal(hopefully) runtime overhead at the cost of low code size overhead by adding guards before every load/store and indirect branch instructions. -->

## Introduction

Isolation is a foundational requirement for running unstrusted code(third-party driver, browser plugins etc), critical for enforcing security, stability, and resource management. Traditional hardware-based isolation mechanisms, such as virtual memory, achieve this by partitioning processes into separate address spaces. However, these approaches incur significant performance penalties due to context-switching overheads, including costly TLB shootdowns and cache misses during domain transitions. Recent CPUs introduce [new hardware isolation mechanisms](https://mars-research.github.io/doc/2024-atc-hw-isolation.pdf) with lower overhead, including but not limited to Intel MPK, ARM PAC, ARM MTE and ARM morello. However, those solutions have the limitation of a small number of isolation domains or IPI(Inter-processor interrupt) cost etc(need to read the paper more carefully)  

Software Fault Isolation (SFI) offers an alternative strategy that avoids such hardware dependencies. By instrumenting low-level machine code, SFI enforces memory and control-flow safety within a single address space, allowing multiple protection domains to coexist efficiently. This technique can be implemented in various ways—via [machine-code interpreters, compilers, or machine-code rewriters](https://www.cse.psu.edu/~gxt29/papers/sfi-final.pdf). Among these, machine-code rewriting stands out as a compelling approach: it minimizes both the trusted computing base (TCB) and runtime overhead by operating directly on compiled binaries, bypassing the need for source code(ideally, needs a lot of engineering effort to rewrite closed-source binary) or specialized hardware. Machine code rewriting removes the compiler and rewriter from TCB by checking the rewritten binary with an independent verifier). This work focuses on SFI through machine-code rewriting, analyzing its design principles, enforcement guarantees, and practical trade-offs in modern systems.

### Machine code rewriter
A naive machine code rewriter restricts memory-access(read and write) by inserting a dynamic address check before every load and store instruction, at the cost of CPU cycles and code size. [Wahbe](https://cs155.stanford.edu/papers/sfi.pdf) and [LFI](https://dl.acm.org/doi/pdf/10.1145/3620665.3640408) uses dedicated, reserved registers for memory access, enforcing data-access policy by maintaining two invariants: pointers in dedicated registers are always in range and all memory access instructions only use dedicated registers for addressing. This technique usually reserves at least one register for holding the base address of the sandbox. LFI achieves zero cycle overhead for basic addressing mode and 1 cycle overhead for other addressing modes.(put code and data in 2 different sandboxes? Modifying code at run time? Exchange heap by writing to the data base register? Stack access is easy to identify but all pointers to the heap on the stack gets invalidated). [KFLex(kernel extension)](https://rs3lab.github.io/assets/papers/2024/dwivedi:kflex.pdf) use the same scheme as LFI on x86: the guard is implemented with an AND(with mask to get offset) and an ADD(to the base address of the sandbox)(two reserved registers).

In-place Address masking modifies the register state at the cost of one cycle and enables more optimization by static analysis.(address masking can also be used with reserved registers but more instructions is required). In ARM64, in-place address masking can be implemented with a single `movk` instruction(need to read the [NACL on Arm](https://static.usenix.org/events/sec10/tech/full_papers/Sehr.pdf) paper)(Google's Native Client on Arm uses `bic` with a 8bit mask, but this design only supports one domain per address space)(According to NACL, inplace write creates a data dependency with the later load/store, a 2 cycle stall, really? This may not be true on recent processors). However, inline guards may be bypassed by indirect branch. Existing methods for enforcing the control-flow policy includes aligned chunk enforcement and control flow integrity. Both introduce overhead and complexity. The BTI (Branch Target Identification) instruction in ARM64 (introduced in ARMv8.5-A) is a hardware-assisted feature designed to enforce Control-Flow Integrity (CFI) and mitigate code-reuse attacks. BTI ensures that indirect branches (e.g., jumps via function pointers, virtual method calls, or returns) only target valid "branch targets" explicitly marked in the code. This prevents attackers from hijacking control flow by redirecting branches to arbitrary code locations. BTI incurs negligible [overhead](https://newsroom.arm.com/blog/pac-bti) both in code size and runtime performance.(range analysis, minimizing the number of masking instructions needed). 

### LFI: up to 2^16 sandboxes in the same address space(4GB)
The runtime of LFI works like a mini Unix-like kernel and provides syscall services to sandboxes.

### Original idea

Use `movk` instruction to implement in-place address masking and apply optimizations like register hoisting(eliminate redundant checks when accessing struct fields and array. or hoisting at the basic block level). Use BTI for control flow integrity. At load time rewrite the immediate in the `movk`
instruction to be the base address of the sandbox.

However, according to LFI, register hoisting only reduced 1% overhead. Zero-instruction guard contributes most to overhead reduction. 


### Other ideas 

A sandboxed program lives in a logical 0-based address space. At runtime the actual address is a concatenation of the logic address and base addressing(sounds like segmenentation). If we have only one sandbox per process, we can change all 64 bit pointer registers to the 32bit counterpart: ldr x11, [x12] -> ldr w11, [x12] and no masking needed. If all sandboxes live in a logical 0-based address, does it make object sharing easier? Consider a linked list(See KFLEX).

## Limitation of SFI

### Need source code or reliable disassembly(still much engineering effort)

- Need to disable compressed jump table(See LFI)

- No vector scatter/gather instructions(See LFI)

### Lack of mechanisms for Zero-copy data transfer and stack unwinding(after the sandbox crashes)

- How to safely transfer objects between domains? 
    - Need to run destructors of shared objects if the domain crashes(imagine a ref-counted object). Difficult to do with machine code rewriters.
<!-- - Need something like capability -->
- KFLEX: manipulate kernel objects with a verified interface and maintains object tables at cancellation points

### Overhead of instrumentation

- NACL: 5% but designed for only 1 sandbox per address space(spec 2000)

- LFI: 7% (1% with no load guard)

### Portability
- Ideally a SFI scheme should support different architectures. The zero-overhead addressing masking of LFI uses the special addressing mode of ARM64.
- ARM cores that does not have a MMU are 32bit microcontrollers.
<!-- - For convenience, just consider ARM64(Or RISC) for now. -->

### Granularity

- Sandbox size(4G too large)
- Ideally, object level granularity

### Interface, how does the sandbox interact with the trusted runtime

- IDL(Interface description language)

- Sec-comp?

### Back to segmenentation
- All machine code rewriters are trying to build a simple mechanism similar to old x86 memory segmenentation.
- There exists new hardware like [CHERI(capability hardware)](https://www.cl.cam.ac.uk/research/security/ctsrd/cheri/) and [HFI(hardware-assisted software isolation)](https://cseweb.ucsd.edu/~tullsen/hfi.pdf)that can be used to implement software sandboxes.  
- However, limitations of SFI due to lack of high-level semantic information persist.

## Idea

- Is it possible to implement the private-shared heap design at machine code level?
    - KFLEX implemented something like this. Public kernel objects can be only be access through an interface and private heap objects are guarded bySFI.

- A mix of inplace and reserved-register masking: any benefits?
    - Inplace: no need to reserve registers(According to LFI, arm in a register-rich architecture so reservation of a few registers have mininal impact on performance). If the code memory and data memory are in two different sandboxes, the code memory can be writable and we can at run time dynamically overwrite the base address of some `movk` instruction to switch the pointer to another sandbox(fine granularity). 
    - Reserved register: changing the base address in the reserved-register redirects all pointers to another sandbox(coarse granularity). 

- KFLEX limitations:
    - TCB gets larger over time. [eBPF verifier may be buggy](https://sigops.org/s/conferences/hotos/2023/papers/jia.pdf)
    - SFI overhead is high(2 reserved registers and 2 added instructions for guarding)

- Is it possible to combine safe Rust with SFI?
    - SFI is language-independent.
    <!-- We need a foundational proof of the type system: check [RefineRust](https://plv.mpi-sws.org/refinedrust)(Try #25860 with refinedRust, unfortunately it does not work)  -->
    - [TRust](https://www.usenix.org/system/files/usenixsecurity23-bang.pdf) uses SFI(with source code) or Intel MPK(without source code, usually FFI) to isolate unsafe Rust code from safe Rust(12% overhead, high). Threat model: safe Rust code is trusted but unsafe code(as a dependency) is not.
    - Unfortunately, SFI cannot mitigate unsoundness of safe Rust.
    
- Rust(ownership) for Isolation
    <!-- - Preserve type info in the assembly? Not workable. The problem remains the same: the implementation of the type system may be buggy.  -->
    - The soundness of the Rust type system has been formulated by RustBelt(followup: RefinedRust)
    - The unique ownership system of safe Rust can be used to establish protection domains(see [RedLeaf](https://www.usenix.org/system/files/osdi20-narayanan_vikram.pdf)) at almost zero cost 
    - Rust for isolation: TCB too huge: Rustc and LLVM. However, the benefit is 
        - (almost)zero-cost isolation, 
        - hardware-independent, 
        - object level granularity, flexible. 
        - Problems with assembly rewriter: lack of semantic information, too many runtime checks. 
    - The major problem: unsoundness bug in Rustc and LLVM: [unsoundness in safe Rust](https://github.com/rust-lang/rust/issues/25860#issuecomment-1955285462); [Open issues about unsoundness](https://github.com/rust-lang/rust/issues?q=state%3Aopen%20label%3A%22I-unsound%22)
    - [Rudra](https://taesoo.kim/pubs/2021/bae:rudra.pdf) also discusses this. One hole in the soundness can compromise the memory safety of the whole system: basically Rust degenerates to C or worse, undefined behaviors. 
    - There exists research operating systems that implement isolation based on Rust safety: RedLeaf and TockOS. TockOS targets microcontrollers that does not have MMU. 
- Rust + Verification towards Zero-overhead Isolation
    - Ideally we will have: 
        - A verified Rust compiler like CompCert for C. Then Rustc and LLVM are out of TCB. Most importantly, we need to prove the absence of unsoundness in safe Rust so that all reasonings based on the ownership system hold. 
        - On top of that we can build a verified runtime service and std for software sandboxes(memory allocator, IO etc). All interfaces are safe but to use some interface with higher order invariants the application developer must provide proofs(for example, assertions) to satisfy the preconditions of the interface  
        - The sandboxed code is restricted to safe Rust so that the ownership system can provide memory safety guarantees(and information flow control? Enforcing security polices via types). Also, in safe Rust, no inline assembly allowed so there will be no privileged instructions in the binary.
        - Cross domain call proxying for fault isolation(See Redleaf)
        - Target application: safety-critical systems or real-time systems. No MMU. Direct mapping so TLB not needed. 

    - Unfortunately, Rust does not have a language specification. Memory model and operational semantics are not defined.
    - No third-party frontend compiler 
    - Currently most Rust program verifier drives Rustc. Verus, especially, also relies of the correctness of the borrow checker of Rustc, 
    - For the case of kernel extensions, the interface is suitable for formal verification. Can we ask extension developers to only write safe Rust and help provide proofs for the postconditions required by the interface? Is it possbile to ease extension developer burden by providing them a better Verification Programming Interface(like Vstd in Verus, Vstd provides axioms and external specs to make writing verification easier). 
    - Would it be easier to build a verified Rustc if we only support a limited subset of safe Rust? Or a checker for unsoundness. LLVM can stay in the TCB but at least we need a more trustworthy Rustc frontend. To check list:
        - Miri 
        - Ferrocene
        - RefinedRust
        - MiniRust
        - Check model checker Kani's approach for verifying Rust code 


