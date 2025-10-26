---
title: Tools Related to Lifting - GTIRB
date: 2025-10-21
categories: [Compiler]
tags: [arm, compiler]     # TAG names should always be lowercase
published: false
---

> Problem: I need to find out the usages of gtirb and do the correct mapping

Resources:
- [GrammaTech toolchain dev](https://www.grammatech.com/open-source-software/gtirb/)
- [GTIRB introduction](https://grammatech.github.io/gtirb/)

# GTIRB DATA structures
- The core data structure illustration
- ![gtirb datastructure](/commons/images/compiler/Gtirb_data_struct.png)

## modules
- modules are for FAT elf, like one binary for multiple arches, so in this case, there are more than one modules
- other cases, there are libraries, like .so, in the modules
- in my problem range, I only need to use the module related to the arm binary

## sections (ByteIntervals)
- there is no debug and comment sections in the final gtirb module's sections
- And sections and module.byte_interval can be seen as one to one mapping.
```shell
ByteInterval at 0x65964 section .gnu.hash
ByteInterval at 0x66368 section .text
ByteInterval at 0x66816 section .ARM.exidx
ByteInterval at 0x66196 section .gnu.version_r
ByteInterval at 0x65896 section .note.gnu.build-id
ByteInterval at 0x66276 section .init
ByteInterval at 0x135168 section .got
ByteInterval at 0x134932 section .fini_array
ByteInterval at None section .ARM.attributes
ByteInterval at 0x66108 section .dynstr
ByteInterval at 0x65932 section .note.ABI-tag
ByteInterval at 0x66228 section .rel.dyn
ByteInterval at 0x135204 section .data
ByteInterval at 0x135212 section .bss
ByteInterval at 0x66288 section .plt
ByteInterval at 0x66824 section .eh_frame
ByteInterval at 0x66808 section .rodata
ByteInterval at 0x65876 section .interp
ByteInterval at 0x66182 section .gnu.version
ByteInterval at 0x134936 section .dynamic
ByteInterval at 0x66236 section .rel.plt
ByteInterval at 0x66012 section .dynsym
ByteInterval at 0x134928 section .init_array
ByteInterval at 0x66800 section .fini
```
## Blocks (data-block, code-block and proxy-block)


# Auxiliary library
## gtirb-layout and gtirb-pprinter
- [gtirb-layout](https://github.com/GrammaTech/gtirb-pprinter/tree/master/src)
  - A small utility/library that chooses a concrete byte layout for a GTIRB module: orders sections, byte-intervals, blocks; assigns/adjusts addresses; and ensures reachability constraints (e.g., literal pool distances). You run it before pretty-printing so addresses/offsets are consistent.
- [gtirb-pprinter](https://github.com/GrammaTech/gtirb-pprinter)
  - A pretty-printer that turns a laid-out GTIRB module into gas-syntax assembly, optionally driving an assembler/linker to produce a working binary. It uses the layout to emit correct labels, directives, relocation expressions (GOT/PLT/PCREL/TLS, etc.), and instruction text.
- Pipeline
  - 1. **Lift** a binary → **GTIRB** (e.g., with `ddisasm`).
  - 2. **Run `gtirb-layout`** to finalize addresses & ordering (sections, intervals, blocks).
  - 3. **Run `gtirb-pprinter`** to generate assembly (and, if requested, call the toolchain to re-assemble/link). 
## How symbols and PC-relative references are represented in GTIRB

GTIRB puts symbolic info **on the bytes**, not the instruction object:
**Symbolic expressions** live in `ByteInterval.symbolic_expressions[off]`.  
Two main forms:    
- * `SymAddrConst(S, +A)` → computes **S + A**.  
- * `SymAddrAddr(S1, S2, scale, +A)` → computes **(S1 − S2)/scale + A** (used for PC-relative encodings like `S − P − 8` on ARM). 
  
* **Attributes** mark intent like **PCREL**, **PLT**, **GOT**, **SECREL**, **TLS**; these drive how the pretty-printer formats the expression (`S - .`, `%plt(S)`, `%got(S)`, etc.). [GrammaTech](https://grammatech.github.io/gtirb/md__symbolic_expression.html)
    
* **Symbols** (`module.symbols`) refer to **CodeBlock**/**DataBlock**/**ProxyBlock**. External/undefined things (e.g., `printf`) are usually **ProxyBlock**s; GOT/PLT entries point at these. 

## What `gtirb-layout` actually decides

Think of `gtirb-layout` as producing a **feasible address assignment**:

* **Section/interval order**: decides the order of sections (e.g., `.text`, `.rodata`, `.got`, …) and the order of **ByteIntervals** inside them, then sets `ByteInterval.address` accordingly.
    
* **Block packing**: preserves block order within intervals (unless told otherwise), placing **CodeBlocks** and **DataBlocks** at specific `offset`s. This directly affects **P** in PC-relative formulas (`S − P + A`).
    
* **Distance constraints**: in ARM/Thumb, ensures **literal pools** reside within branch/load reach of their consumers. If not feasible, it moves/duplicates pools or inserts new pool sites per interval policy (so later the `LDR (PC,#imm)` can reach them).
    
* **Alignment & padding**: meets section alignment, block alignment, and any toolchain-expected padding so that later assembly can be reproduced faithfully.
    

Net effect: after `gtirb-layout`, every **site** (the byte location holding a symbolic expression) has a stable **VA**, so `S` and `P` in all expressions are well-defined.

## What `gtirb-pprinter` reads and how it prints PC-relative uses

The pretty-printer walks the laid-out module and emits:

1. **Labels for symbols**  
    For each `Symbol` with a concrete referent, it prints a label at that byte address (function names, local labels like `.Ldatause`, data labels, pool anchors, etc.). External symbols become `.extern`/undefined references depending on the target. 

2. **Instructions**  
    It uses instruction bytes and decoder meta (architecture specified in GTIRB) to print **mnemonic + operands**.
    
3. **Data directives**  
    For bytes not part of instructions (e.g., literal pools, GOT/PLT, constants), it prints `.long/.quad/.byte` with **symbolic expressions** instead of raw immediates whenever there’s a `symbolic_expression` at that site.
    
4. **Symbolic expressions → assembly syntax**  
    It translates GTIRB’s expressions/attributes into assembler syntax:
    
    * **PCREL**: prints a PC-relative form (e.g., `S - .` or platform-specific relocation suffix) when the expression is relative to the **current code location**. 

    * **`SymAddrAddr(S, P, 1, -8)`** (typical ARM literal pool value): becomes **`S - P - 8`** at the pool site (often rendered by a label math like `.word S - Luse - 8`).
        
    * **GOT/PLT**: attributes like **GOT**, **PLT**, **GOTPC**, **GOTOFF** map to target-specific notations, e.g., `%got(S)`, `%plt(S)`, or appropriate reloc suffixes on ELF. 

### Code examples in pprinter to print the final asm according to gtirb

- main entry in `pretty_printer.cpp`
```c
// ctx can be seen as GTIR
ContextForgetter ctx;
//....
// Configure the pretty-printer
gtirb_pprint::PrettyPrinter pp;
// ...
// Update DynMode (-shared or -pie or none) for the module
pp.updateDynMode(M, SharedOption);
// Apply any needed fixups
applyFixups(ctx, M, pp);
//...
// Write ASM to the standard output if no other action was taken.
if ((vm.count("asm") == 0) && (vm.count("binary") == 0) &&
    (vm.count("version-script") == 0)) {
  pp.print(std::cout, ctx, M);
}
```

- start print the whole module
```cpp
std::ostream& PrettyPrinterBase::print(std::ostream& os) {
  // Collect all ambiguous symbols in the module and give them
  // unique names
  computeAmbiguousSymbols();

  printHeader(os);

  // print every section
  for (const auto& section : module.sections()) {
    printSection(os, section);
  }

  printIntegralSymbols(os);

  // print footer
  printFooter(os);
  return os;
}
// section level printer
void PrettyPrinterBase::printSection(std::ostream& os,
                                     const gtirb::Section& section) {
  if (shouldSkip(policy, section)) {
    return;
  }
  programCounter = gtirb::Addr{0};

  printSectionHeader(os, section);

  for (const auto& Block : section.blocks()) {
    if (auto* CB = gtirb::dyn_cast<gtirb::CodeBlock>(&Block)) {
      printBlock(os, *CB);
    } else if (auto* DB = gtirb::dyn_cast<gtirb::DataBlock>(&Block)) {
      printBlock(os, *DB);
    } else {
      assert(!"non block in block iterator!");
    }
  }

  printSectionFooter(os, section);
}
// Block level
void PrettyPrinterBase::printBlock(std::ostream& os,
                                   const gtirb::CodeBlock& block) {
  setDecodeMode(os, block); // .arm
  printBlockImpl(os, block);
}
// jump debug blocks and symbols during printing
void PrettyPrinterBase::printBlockImpl(std::ostream& os, BlockType& block) {
  if (shouldSkip(policy, block)) {
    return;
  }

  // Print symbols associated with block.
  gtirb::Addr addr = *block.getAddress();
  uint64_t offset;

  if (addr < programCounter) {
    // If the program counter is beyond the address already, then overlap is
    // occuring, so we need to print a symbol definition after the fact (rather
    // than place a label in the middle).

    offset = programCounter - addr;
    printOverlapWarning(os, addr);
    for (const auto& sym : module.findSymbols(block)) {
      if (!sym.getAtEnd() && !shouldSkip(policy, sym)) {
        printSymbolDefinitionRelativeToPC(os, sym, programCounter);
      }
    }
  } else {
    // Normal symbol; print labels before block.

    offset = 0;

    if (auto Align = getAlignment(block)) {
      printAlignment(os, *Align);
    }

    for (const auto& sym : module.findSymbols(block)) {
      if (!sym.getAtEnd() && !shouldSkip(policy, sym)) {
        printSymbolDefinition(os, sym);
      }
    }
  }

  // If this occurs in an array section, and the block points to something we
  // should skip: Skip contents, but do not skip label, so things can refer to
  // the array as a whole.
  if (policy.arraySections.count(
          block.getByteInterval()->getSection()->getName())) {
    if (auto SymExpr =
            block.getByteInterval()->getSymbolicExpression(block.getOffset())) {
      if (std::holds_alternative<gtirb::SymAddrConst>(*SymExpr)) {
        if (shouldSkip(policy, *std::get<gtirb::SymAddrConst>(*SymExpr).Sym)) {
          return;
        }
      } else {
        assert(!"Unexpected sym expr type in array section!");
      }
    }
  }

  // Print actual block contents.
  printBlockContents(os, block, offset);

  // Update the program counter.
  programCounter = std::max(programCounter, addr + block.getSize());

  // Print any symbols that should go at the end of this block.
  for (const auto& sym : module.findSymbols(block)) {
    if (sym.getAtEnd() && !shouldSkip(policy, sym)) {
      printSymbolDefinition(os, sym);
    }
  }
  // Print function ends if applicable
  if (FunctionLastBlocks.count(block.getUUID()) > 0) {
    const gtirb::Symbol* FunctionSymbol =
        getContainerFunctionSymbol(block.getUUID());
    // A function could have no name associated to it.
    if (FunctionSymbol) {
      printFunctionEnd(os, *FunctionSymbol);
      if (auto Aliases = FunctionAliases.find(FunctionSymbol);
          Aliases != FunctionAliases.end()) {
        for (const auto* Alias : Aliases->second) {
          printFunctionEnd(os, *Alias);
        }
      }
    }
  }
}
```
- In terms of printing the literal pools in the instructions related to label-replaced instruciton, we can find the implementation here
```cpp

void ArmPrettyPrinter::printBlockContents(std::ostream& Os,
                                          const gtirb::CodeBlock& X,
                                          uint64_t Offset) {
  if (Offset > X.getSize()) {
    return;
  }

  gtirb::Addr Addr = *X.getAddress();
  Os << '\n';

  const std::vector<size_t>& CsModes =
      X.getDecodeMode() != gtirb::DecodeMode::Thumb ? ArmCsModes : ThumbCsModes;

  std::unique_ptr<cs_insn, std::function<void(cs_insn*)>> InsnPtr;
  size_t InsnCount = 0;

  // NOTE: If the ARM CPU profile is not known, we may have to switch modes
  // to successfully decode all instructions.
  // Thumb2 MRS and MSR instructions support a larger set of `<spec_reg>` on
  // M-profile devices, so they do not decode without CS_MODE_MCLASS.
  // The Thumb 'blx label' instruction does not decode with CS_MODE_MCLASS,
  // because it is not a supported instruction on M-profile devices.
  //
  // This loop is to try out multiple CS modes to see if decoding succeeds.
  // Currently, this is done only when the arch type info is not available.
  bool Success = false;
  for (size_t CsMode : CsModes) {
    cs_insn* Insn = nullptr;

    cs_option(this->csHandle, CS_OPT_MODE, CsMode);
    InsnCount = cs_disasm(this->csHandle, X.rawBytes<uint8_t>() + Offset,
                          X.getSize() - Offset,
                          static_cast<uint64_t>(Addr) + Offset, 0, &Insn);

    // Exception-safe cleanup of instructions
    std::unique_ptr<cs_insn, std::function<void(cs_insn*)>> TmpInsnPtr(
        Insn, [InsnCount](cs_insn* Instr) { cs_free(Instr, InsnCount); });

    size_t TotalSize = 0;
    for (size_t J = 0; J < InsnCount; J++) {
      TotalSize += Insn[J].size;
    }
    // If the sum of the instruction sizes equals to the block size, that
    // indicates the decoding succeeded.
    Success = (TotalSize == X.getSize() - Offset);
    // Keep the decoding attempt.
    if (Success) {
      // Assign the ownership of TmpInsnPtr to InsnPtr. This passes along the
      // deleter as well.
      // https://en.cppreference.com/w/cpp/memory/unique_ptr/operator%3D
      InsnPtr = std::move(TmpInsnPtr);
      break;
    }
  }

  if (!Success) {
    LOG_ERROR << "Failed to decode block at " << std::hex
              << static_cast<uint64_t>(Addr) + Offset << std::dec;
    std::exit(EXIT_FAILURE);
  }

  gtirb::Offset BlockOffset(X.getUUID(), Offset);
  for (size_t I = 0; I < InsnCount; I++) {
    fixupInstruction((&(*InsnPtr))[I]);
    printInstruction(Os, X, (&(*InsnPtr))[I], BlockOffset);
    BlockOffset.Displacement += (&(*InsnPtr))[I].size;
  }

  // print any CFI directives located at the end of the block
  // e.g. '.cfi_endproc' is usually attached to the end of the block
  printCFIDirectives(Os, BlockOffset);
}

void ArmPrettyPrinter::printInstruction(std::ostream& os,
                                        const gtirb::CodeBlock& block,
                                        const cs_insn& inst,
                                        const gtirb::Offset& offset) {
  gtirb::Addr ea(inst.address);
  std::stringstream InstructLine;
  printComments(InstructLine, offset, inst.size);
  printCFIDirectives(InstructLine, offset);
  printEA(InstructLine, ea);
  std::string opcode = ascii_str_tolower(inst.mnemonic);
  if (auto index = opcode.rfind(".w"); index != std::string::npos)
    opcode = opcode.substr(0, index);

  auto isItInstr = [](const std::string& i) {
    static const std::vector<std::string> it_instrs{
        "it",    "itt",   "ite",   "ittt",  "itte",  "itet",  "itee", "itttt",
        "ittte", "ittet", "ittee", "itett", "itete", "iteet", "iteee"};
    return (std::find(std::begin(it_instrs), std::end(it_instrs), i) !=
            std::end(it_instrs));
  };

  InstructLine << "  " << opcode;
  if (isItInstr(opcode)) {
    std::string cc = armCc2String(inst.detail->arm.cc);
    InstructLine << " " << cc;
  }
  InstructLine << ' ';
  // Make sure the initial m_accum_comment is empty.
  m_accum_comment.clear();
  printOperandList(InstructLine, block, inst);

  if (inst.detail->arm.cps_flag != ARM_CPSFLAG_NONE &&
      inst.detail->arm.cps_flag != ARM_CPSFLAG_INVALID) {
    if (inst.detail->arm.cps_flag & ARM_CPSFLAG_I)
      InstructLine << "i";
    if (inst.detail->arm.cps_flag & ARM_CPSFLAG_F)
      InstructLine << "f";
    if (inst.detail->arm.cps_flag & ARM_CPSFLAG_A)
      InstructLine << "a";
  }

  if (!m_accum_comment.empty()) {
    printCommentableLine(InstructLine, os, ea);
    InstructLine.str(std::string()); // Clear
    os << '\n';
    InstructLine << syntax.comment() << " ";
    printEA(InstructLine, ea);
    InstructLine << ": " << m_accum_comment;
    m_accum_comment.clear();
  }
  printCommentableLine(InstructLine, os, ea);
  os << '\n';
}

void ArmPrettyPrinter::printOperandList(std::ostream& os,
                                        const gtirb::CodeBlock& block,
                                        const cs_insn& inst) {
  cs_arm& detail = inst.detail->arm;

  int opCount = detail.op_count;

  static const std::set<arm_insn> LdmStm = {
      ARM_INS_LDM,     ARM_INS_LDMDA,   ARM_INS_LDMDB,  ARM_INS_LDMIB,
      ARM_INS_FLDMDBX, ARM_INS_FLDMIAX, ARM_INS_VLDMDB, ARM_INS_VLDMIA,
      ARM_INS_STM,     ARM_INS_STMDA,   ARM_INS_STMDB,  ARM_INS_STMIB,
      ARM_INS_FSTMDBX, ARM_INS_FSTMIAX, ARM_INS_VSTMDB, ARM_INS_VSTMIA};

  static const std::set<arm_insn> PushPop = {ARM_INS_POP, ARM_INS_PUSH,
                                             ARM_INS_VPOP, ARM_INS_VPUSH};

  static const std::set<arm_insn> VldVst = {
      ARM_INS_VLD1, ARM_INS_VLD2, ARM_INS_VLD3, ARM_INS_VLD4,
      ARM_INS_VST1, ARM_INS_VST2, ARM_INS_VST3, ARM_INS_VST4};

  // VLDn/VSTn
  if (VldVst.find(static_cast<arm_insn>(inst.id)) != VldVst.end()) {
    os << "{ ";
    for (int i = 0; i < opCount; i++) {
      const cs_arm_op& op = detail.operands[i];
      // Print out closing parenthesis once a memory operand is encountered.
      if (op.type == ARM_OP_MEM) {
        os << " }, [";
        if (op.mem.base != ARM_REG_INVALID) {
          os << getRegisterName(op.mem.base);
        }
        // The disp is for alignment for VLDn and VSTn instructions.
        if (op.mem.disp != 0) {
          os << " :" << op.mem.disp;
        }
        os << "]";
        if (detail.writeback) {
          os << "!";
        }
      } else {
        if (i != 0) {
          os << ", ";
        }
        printOperand(os, block, inst, i);
      }
    }
    return;
  }

  int RegBitVectorIndex = -1;

  if (LdmStm.find(static_cast<arm_insn>(inst.id)) != LdmStm.end())
    RegBitVectorIndex = 1;
  if (PushPop.find(static_cast<arm_insn>(inst.id)) != PushPop.end())
    RegBitVectorIndex = 0;

  for (int i = 0; i < opCount; i++) {
    if (i != 0) {
      os << ", ";
    }
    if (i == RegBitVectorIndex)
      os << "{ ";
    printOperand(os, block, inst, i);
    if (LdmStm.find(static_cast<arm_insn>(inst.id)) != LdmStm.end() && i == 0 &&
        detail.writeback) {
      os << "!";
    }
  }
  if (RegBitVectorIndex != -1)
    os << " }";
}
void ArmPrettyPrinter::printOperand(std::ostream& os,
                                    const gtirb::CodeBlock& block,
                                    const cs_insn& inst, uint64_t index) {
  gtirb::Addr ea(inst.address);
  const cs_arm_op& op = inst.detail->arm.operands[index];

  const gtirb::SymbolicExpression* symbolic = nullptr;
  switch (op.type) {
  case ARM_OP_REG:
  case ARM_OP_SYSREG: {
    if (op.subtracted)
      os << "-";
    printOpRegdirect(os, inst, index);
    return;
  }
  case ARM_OP_IMM:
  case ARM_OP_PIMM:
  case ARM_OP_CIMM: {
    symbolic = block.getByteInterval()->getSymbolicExpression(
        ea - *block.getByteInterval()->getAddress());
    printOpImmediate(os, symbolic, inst, index);
    return;
  }
  case ARM_OP_FP: {
    os << "#" << std::scientific << std::setprecision(18) << op.fp;
    return;
  }
  case ARM_OP_MEM: {
    symbolic = block.getByteInterval()->getSymbolicExpression(
        ea - *block.getByteInterval()->getAddress());
    printOpIndirect(os, symbolic, inst, index);
    return;
  }
  case ARM_OP_SETEND: {
    switch (op.setend) {
    case ARM_SETEND_BE:
      os << "BE";
      break;
    case ARM_SETEND_LE:
      os << "LE";
      break;
    default:
      std::cerr << "invalid SETEND operand\n";
      exit(1);
    }
    return;
  }
  default:
    std::cerr << "invalid operand\n";
    exit(1);
  }
}
```
- print immediate and indirect op
```cpp
void ArmPrettyPrinter::printOpImmediate(
    std::ostream& os, const gtirb::SymbolicExpression* symbolic,
    const cs_insn& inst, uint64_t index) {
  const cs_arm_op& op = inst.detail->arm.operands[index];
  if (const auto* SAA = std::get_if<gtirb::SymAddrAddr>(symbolic)) {
    printSymbolicExpression(os, SAA, false);
  } else if (const gtirb::SymAddrConst* s =
                 this->getSymbolicImmediate(symbolic)) {
    // The operand is symbolic.
    if (inst.id == ARM_INS_MOV)
      os << "#:lower16:";
    if (inst.id == ARM_INS_MOVT)
      os << "#:upper16:";
    this->printSymbolicExpression(os, s, true);
  } else {
    if (op.type == ARM_OP_IMM)
      os << '#';
    else if (op.type == ARM_OP_CIMM)
      os << "cr";
    // The operand is just a number.
    os << op.imm;
  }
}
void ArmPrettyPrinter::printOpIndirect(
    std::ostream& os, const gtirb::SymbolicExpression* symbolic,
    const cs_insn& inst, uint64_t index) {
  const cs_arm& detail = inst.detail->arm;
  const cs_arm_op& op = detail.operands[index];
  assert(op.type == ARM_OP_MEM &&
         "printOpIndirect called without a memory operand");

  // pc-relative indirect operands like [pc, #100] with a symbolic expression
  // must replace the operand with the expression.
  // NOTE: For pc with register offset (including jump-table instructions)
  //       print the PC-relative operand as it is.
  if (op.mem.base == ARM_REG_PC && op.mem.index == ARM_REG_INVALID) {
    if (const auto* s = std::get_if<gtirb::SymAddrConst>(symbolic)) {
      printSymbolicExpression(os, s, false);
      return;
    }
  }

  bool first = true;
  os << '[';

  if (op.mem.base != ARM_REG_INVALID) {
    first = false;
    os << getRegisterName(op.mem.base);
  }

  if (op.mem.index != ARM_REG_INVALID) {
    if (!first)
      os << ", ";
    first = false;
    if (op.mem.scale == -1)
      os << "-";
    os << getRegisterName(op.mem.index);
  }

  if (op.shift.value != 0 && op.shift.type != ARM_SFT_INVALID) {
    os << ", ";
    switch (op.shift.type) {
    case ARM_SFT_ASR_REG:
    case ARM_SFT_ASR:
      os << "ASR";
      break;
    case ARM_SFT_LSL_REG:
    case ARM_SFT_LSL:
      os << "LSL";
      break;
    case ARM_SFT_LSR_REG:
    case ARM_SFT_LSR:
      os << "LSR";
      break;
    case ARM_SFT_ROR_REG:
    case ARM_SFT_ROR:
      os << "ROR";
      break;
    case ARM_SFT_RRX_REG:
    case ARM_SFT_RRX:
      os << "RRX";
      break;
    case ARM_SFT_INVALID:
      std::cerr << "Invalid ARM shift operation.\n";
      exit(1);
    }
    os << " " << op.shift.value;
  }

  if (const auto* s = std::get_if<gtirb::SymAddrConst>(symbolic)) {
    os << ", #";
    printSymbolicExpression(os, s, false);
  } else {
    if (op.mem.disp != 0)
      os << ", #" << op.mem.disp;
  }
  os << ']';
  if (detail.writeback) {
    // If this operand has 'writeback', and it's the last operand,
    // it can be assumed to be pre-indexed.
    // NOTE: Is there a way of finding the info about pre/post-index in
    // capstone??
    if ((uint64_t)(detail.op_count - 1) == index) {
      os << '!';
    }
  }
}
```

