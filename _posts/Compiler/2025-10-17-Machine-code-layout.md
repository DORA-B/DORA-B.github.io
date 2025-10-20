---
title: Machine code layout
date: 2025-10-17
categories: [Compiler]
tags: [arm, compiler]     # TAG names should always be lowercase
published: false
---

> Problem: I am thinking whether it is possible to convert a linked binary file (EXEC) back to a assembly file (PIC friendly)?
> Which means:
> 1. convert jump target addresses back to label, code position without pc.
> 2. convert data fetching address back to label
> ...

# Resources 
- [MachineBlockPlacement.cpp in LLVM](https://llvm.org/doxygen/MachineBlockPlacement_8cpp_source.html)
- [Machine code layout optimizations](https://easyperf.net/blog/2019/03/27/Machine-code-layout-optimizatoins?utm_source=chatgpt.com)
Compilers like to operate on a basic block level, because it is guaranteed that every instruction in the basic block will be executed exactly once. Thus for some problems we can treat all instructions in the basic block as one instruction. This greatly reduces the problem of CFG (control flow graph) analysis and transformations.

# Main idea description
- Basic block placement
  - Compare two layouts, the second one is more friendly for cache: the hot path is closer.
  - Main reason is because we maintain fall through between hot pieces of the code. Not taken branches are fundamentally cheaper that taken. Additionally second case better utilizes L1 I-cache and uop-cache (DSB).
  ![compare_two_path](/commons/images/compiler/compare_two_path.png)
- [Basic block alignment Issues](https://easyperf.net/blog/2018/01/18/Code_alignment_issues)
  - This is purely micro-architectural optimization which is usually applied to loops. Figure below is the best brief explanation of the matter:
  - ![code alignment](/commons/images/compiler/code-alignment.png)
  - shift the hot code (in yellow) down using NOPs (in blue) so that the whole loop will reside in one cache line. On the picture below cache line start from c0 and ends at ff. This transformation usually improves I-cache and DSB utilization.
- Function splitting, which can be used to separate hot from cold code. This usually improves the memory locality of the hot code.
  - The idea of function splitting is to separate hot from cold code. This usually improves the memory locality of the hot code. Example:
  ```c
  void foo(bool cond1, bool cond2) {
    // hot path
    if (cond1)
      // cold code 1
    //hot code
    if (cond2)
      // cold code 2
  }
  ```
  - We might want to cut cold part of the function into it’s own new function and put a call to it instead (Outlining). Something like this:
  ````c
  void foo(bool cond1, bool cond2) {
    // hot path
    if (cond1)
      cold1(); 
    //hot code
    if (cond2)
      cold2(); 
  }

  void cold1() __attribute__((noinline)) { // cold code 1 }
  void cold2() __attribute__((noinline)) { // cold code 2 }
  ```
  - Because we just left the call instruction inside the hot path it’s likely that next hot instruction will reside in the same cache line. (Shrinkage the cold code usage in the cache)This improves utilization of CPU Front-End data structures like I-cache and DSB-cache.
- Function Grouping
  - Usually we want to place hot functions together such that they touch each other in the same cache line.
  - This optimization works best when there are many small hot functions. 
  - There is also very cool tool for doing this automatically: hfsort. It uses linux perf for getting profile information, then it does it’s ordering magic and gives the text file with optimized function order which you can then pass to the linker. Here is the whitepaper if you want to read more about the underlying algorithms.
- Profile guided optimizations (PGO) - Optmize the app just for that single workload
  - Using PGO -- It is usually the best option to use PGO if you can come up with a typical scenario for your application.
  - Given profiling information compiler doesn’t need to question what should be the decision. And often times you will see in compiler implementation the pattern like:
  ```c
  if (profiling information available)
    make decision based on profiling data
  else
    make decision based on heuristics
  ```
# What is more?
- some research works reorder functions using PGO (function reordering), or newer techniques like basic-block sections (each block in its own section, so the linker can stitch them using profile). 
  - Google’s [Propeller](https://research.google/pubs/propeller-a-profile-guided-relinking-optimizer-for-warehouse-scale-applications/?utm_source=chatgpt.com) relinks using basic-block sections to build globally optimal hot traces without disassembling.
  - Post-link optimizers like [BOLT](https://research.facebook.com/publications/bolt-a-practical-binary-optimizer-for-data-centers-and-beyond/?utm_source=chatgpt.com) (Meta/Facebook) profile running code and then physically reorder blocks and functions in the final binary to shrink I-cache/I-TLB misses and improve branch prediction.
- Net effect: the dumped binary’s block order is the result of backend MBP decisions + (optionally) profile-guided function/block reordering at link or post-link time. Taken branches might be flipped to make the likely path fall-through, which causes padding/alignment NOPs around hot blocks.

## Answers to my problems
- “Turn PC-relative addresses into labels and group to basic blocks?”
  - Yes. Disassemblers compute targets and name them; tools like Ghidra/radare2 build the full CFG and show basic blocks with fall-through and branch edges. Exported assembly will have labels for block entries. (For reassembleable output, use DDisasm/GTIRB.)
- “How is fall-through vs branch chosen?”
  - The backend’s block placement and branch selection (plus PGO) try to make the hot successor the fall-through and the cold edge the explicit branch; then layout & alignment reduce taken-branch penalties.
- “How is the final physical order decided?”
  - Backend MBP first; then (optionally) Propeller/BOLT or linker’s PGO can reorder functions/blocks using basic-block sections or post-link rewriting, guided by real profiles.
