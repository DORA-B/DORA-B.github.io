# Download and install the WasmEdge Env
+ Follow the [tutorials](https://wasmedge.org/docs/start/install/), Build WasmEdge from source: Build on Linux
+ Build in debug mode
  ```shell
  # After pulling our wasmedge docker image
  docker run -it --rm \
      -v <path/to/your/wasmedge/source/folder>:/root/wasmedge \
      wasmedge/wasmedge:latest
  # In docker
  cd /root/wasmedge
  # If you don't use docker then you need to run only the following commands in the cloned repository root
  mkdir -p build && cd build
  cmake -DCMAKE_BUILD_TYPE=Debug -DWASMEDGE_BUILD_TESTS=ON .. && make -j
  make install
  ```
+ Install and Validation
  ![install WasmEdge](/commons/images/3141911-20231107001231285-854428511.png)

# Running with WasmEdge

+ Compile C/C++ file in to wasm file (emcc)
+ run wasmedge using
  ```shell
  wasmedge hello.wasm
  ``` 
+ Setting debug information 
  ```json
  {
  "version": "0.2.0",
  "configurations": [
    {
      "name":"Debug Wasm3",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/usr/local/bin/wasm3",
      "args": ["/root/WasmPerf/test/hellotest.wasm"],
      "stopAtEntry": false,
      "cwd": "${workspaceFolder}",
      "environment": [],
      "externalConsole": false,
      "MIMode": "gdb",
      "setupCommands": [
          {
              "description": "Enable pretty-printing for gdb",
              "text": "-enable-pretty-printing",
              "ignoreFailures": true
          }
      ],
      "miDebuggerPath": "/usr/bin/gdb",
      "linux": {
          "MIMode": "gdb"
      }
  },
  {
    "name":"Debug WasmEdge",
    "type": "cppdbg",
    "request": "launch",
    "program": "${workspaceFolder}/usr/local/bin/wasmedge",
    "args": ["/root/WasmPerf/test/hellotest.wasm"],
    "stopAtEntry": false,
    "cwd": "${workspaceFolder}",
    "environment": [],
    "externalConsole": false,
    "MIMode": "gdb",
    "setupCommands": [
        {
            "description": "Enable pretty-printing for gdb",
            "text": "-enable-pretty-printing",
            "ignoreFailures": true
        }
    ],
    "miDebuggerPath": "/usr/bin/gdb",
    "linux": {
        "MIMode": "gdb"
    } //,
  }
  ]
  }
  ```
+ Setting break points and go on
## Overview of WasmEdge Execution Flow

![WasmEdge Execution Flow](/commons/images/3141911-20231107000340182-1058266724.png)

## Debug Process

+ Entry in `WasmEdge/tools/wasmedge/wasmedge.cpp`
  ```cpp
  int main(int Argc, const char *Argv[]) {
    return WasmEdge_Driver_UniTool(Argc, Argv);
  }
  ```
+ `WasmEdge::Driver::UniTool()` in `/WasmEdge/lib/api/wasmedge.cpp`
  ```cpp
  WASMEDGE_CAPI_EXPORT int WasmEdge_Driver_UniTool(int Argc, const char *Argv[]) {
    return WasmEdge::Driver::UniTool(Argc, Argv, WasmEdge::Driver::ToolType::All);
  }
  ```
+ During `Tool()` function in `runtimeTool.cpp`
  ![get wasm file's absolute path as input](/commons/images/3141911-20231107041921772-1412443045.png)
  + VM instantiation
    ```cpp
    VM::VM(const Configure &Conf)
        : Conf(Conf), Stage(VMStage::Inited),
          LoaderEngine(Conf, &Executor::Executor::Intrinsics),
          ValidatorEngine(Conf), ExecutorEngine(Conf, &Stat),
          Store(std::make_unique<Runtime::StoreManager>()), StoreRef(*Store.get()) {
      unsafeInitVM();
    }
    void VM::unsafeInitVM() {
      // Load the built-in modules and the plug-ins.
      unsafeLoadBuiltInHosts();
      unsafeLoadPlugInHosts();

      // Register all module instances.
      unsafeRegisterBuiltInHosts();
      unsafeRegisterPlugInHosts();
    }
    ```
  + Get Imported modules from WASI,
    ```cpp
    Runtime::Instance::ModuleInstance *
    VM::unsafeGetImportModule(const HostRegistration Type) const {
      if (auto Iter = BuiltInModInsts.find(Type); Iter != BuiltInModInsts.cend()) {
        return Iter->second.get();
      }
      return nullptr;
    }
    ```
  + Load Wasm 
  + Validation Step:
  ```cpp
  Expect<void> VM::unsafeValidate() {
    if (Stage < VMStage::Loaded) {
      // When module is not loaded, not validate.
      spdlog::error(ErrCode::Value::WrongVMWorkflow);
      return Unexpect(ErrCode::Value::WrongVMWorkflow);
    }
    if (auto Res = ValidatorEngine.validate(*Mod.get())) {
      Stage = VMStage::Validated;
      return {};
    } else {
      return Unexpect(Res);
    }
  }
  ```
  + Instantiate step
  ```cpp
  /// Instantiate a WASM Module. See "include/executor/executor.h".
  Expect<std::unique_ptr<Runtime::Instance::ModuleInstance>>
  Executor::instantiateModule(Runtime::StoreManager &StoreMgr,
                              const AST::Module &Mod) {
    if (auto Res = instantiate(StoreMgr, Mod)) {
      return Res;
    } else {
      // If Statistics is enabled, then dump it here.
      // When there is an error happened, the following execution will not
      // execute.
      if (Stat) {
        Stat->dumpToLog(Conf);
      }
      return Unexpect(Res);
    }
  }
  ```
  ```cpp
  // Instantiate module instance. See "include/executor/Executor.h".
  Expect<std::unique_ptr<Runtime::Instance::ModuleInstance>>
  Executor::instantiate(Runtime::StoreManager &StoreMgr, const AST::Module &Mod,
                        std::optional<std::string_view> Name) {
    // Check the module is validated.
    if (unlikely(!Mod.getIsValidated())) {
      spdlog::error(ErrCode::Value::NotValidated);
      spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Module));
      return Unexpect(ErrCode::Value::NotValidated);
    }

    // Create the stack manager.
    Runtime::StackManager StackMgr;

    // Check is module name duplicated when trying to registration.
    if (Name.has_value()) {
      const auto *FindModInst = StoreMgr.findModule(Name.value());
      if (FindModInst != nullptr) {
        spdlog::error(ErrCode::Value::ModuleNameConflict);
        spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Module));
        return Unexpect(ErrCode::Value::ModuleNameConflict);
      }
    }

    // Insert the module instance to store manager and retrieve instance.
    std::unique_ptr<Runtime::Instance::ModuleInstance> ModInst;
    if (Name.has_value()) {
      ModInst = std::make_unique<Runtime::Instance::ModuleInstance>(Name.value());
    } else {
      ModInst = std::make_unique<Runtime::Instance::ModuleInstance>("");
    }

    // Instantiate Function Types in Module Instance. (TypeSec)
    for (auto &FuncType : Mod.getTypeSection().getContent()) {
      // Copy param and return lists to module instance.
      ModInst->addFuncType(FuncType);
    }

    // Instantiate ImportSection and do import matching. (ImportSec)
    const AST::ImportSection &ImportSec = Mod.getImportSection();
    if (auto Res = instantiate(StoreMgr, *ModInst, ImportSec); !Res) {
      spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Sec_Import));
      spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Module));
      StoreMgr.recycleModule(std::move(ModInst));
      return Unexpect(Res);
    }

    // Instantiate Functions in module. (FunctionSec, CodeSec)
    const AST::FunctionSection &FuncSec = Mod.getFunctionSection();
    const AST::CodeSection &CodeSec = Mod.getCodeSection();
    // This function will always success.
    instantiate(*ModInst, FuncSec, CodeSec);

    // Instantiate TableSection (TableSec)
    const AST::TableSection &TabSec = Mod.getTableSection();
    // This function will always success.
    instantiate(*ModInst, TabSec);

    // Instantiate MemorySection (MemorySec)
    const AST::MemorySection &MemSec = Mod.getMemorySection();
    // This function will always success.
    instantiate(*ModInst, MemSec);

    // Add a temp module to Store with only imported globals for initialization.
    std::unique_ptr<Runtime::Instance::ModuleInstance> TmpModInst =
        std::make_unique<Runtime::Instance::ModuleInstance>("");
    for (uint32_t I = 0; I < ModInst->getGlobalImportNum(); ++I) {
      TmpModInst->importGlobal(*(ModInst->getGlobal(I)));
    }
    for (uint32_t I = 0; I < ModInst->getFuncNum(); ++I) {
      TmpModInst->importFunction(*(ModInst->getFunc(I)));
    }

    // Push a new frame {TmpModInst:{globaddrs}, locals:none}
    StackMgr.pushFrame(TmpModInst.get(), AST::InstrView::iterator(), 0, 0);

    // Instantiate GlobalSection (GlobalSec)
    const AST::GlobalSection &GlobSec = Mod.getGlobalSection();
    if (auto Res = instantiate(StackMgr, *ModInst, GlobSec); !Res) {
      spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Sec_Global));
      spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Module));
      StoreMgr.recycleModule(std::move(ModInst));
      return Unexpect(Res);
    }

    // Pop frame with the temp. module.
    StackMgr.popFrame();

    // Instantiate ExportSection (ExportSec)
    const AST::ExportSection &ExportSec = Mod.getExportSection();
    // This function will always success.
    instantiate(*ModInst, ExportSec);

    // Push a new frame {ModInst, locals:none}
    StackMgr.pushFrame(ModInst.get(), AST::InstrView::iterator(), 0, 0);

    // Instantiate ElementSection (ElemSec)
    const AST::ElementSection &ElemSec = Mod.getElementSection();
    if (auto Res = instantiate(StackMgr, *ModInst, ElemSec); !Res) {
      spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Sec_Element));
      spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Module));
      StoreMgr.recycleModule(std::move(ModInst));
      return Unexpect(Res);
    }

    // Instantiate DataSection (DataSec)
    const AST::DataSection &DataSec = Mod.getDataSection();
    if (auto Res = instantiate(StackMgr, *ModInst, DataSec); !Res) {
      spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Sec_Data));
      spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Module));
      StoreMgr.recycleModule(std::move(ModInst));
      return Unexpect(Res);
    }

    // Initialize table instances
    if (auto Res = initTable(StackMgr, ElemSec); !Res) {
      spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Sec_Element));
      spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Module));
      StoreMgr.recycleModule(std::move(ModInst));
      return Unexpect(Res);
    }

    // Initialize memory instances
    if (auto Res = initMemory(StackMgr, DataSec); !Res) {
      spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Sec_Data));
      spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Module));
      StoreMgr.recycleModule(std::move(ModInst));
      return Unexpect(Res);
    }

    // Instantiate StartSection (StartSec)
    const AST::StartSection &StartSec = Mod.getStartSection();
    if (StartSec.getContent()) {
      // Get the module instance from ID.
      ModInst->setStartIdx(*StartSec.getContent());

      // Get function instance.
      const auto *FuncInst = ModInst->getStartFunc();

      // Execute instruction.
      if (auto Res = runFunction(StackMgr, *FuncInst, {}); unlikely(!Res)) {
        spdlog::error(ErrInfo::InfoAST(ASTNodeAttr::Module));
        StoreMgr.recycleModule(std::move(ModInst));
        return Unexpect(Res);
      }
    }

    // Pop Frame.
    StackMgr.popFrame();

    // For the named modules, register it into the store.
    if (Name.has_value()) {
      StoreMgr.registerModule(ModInst.get());
    }

    return ModInst;
  }

  } // namespace Executor
  } // namespace WasmEdge
  ```

## `Instatiate()` Step
+ `findModule()`
+ `getContent()`
+ `getImportSection()`
+ `getTableSection()`
+ `getMemorySection()`
+ `importGlobal()`
+ `importFunction()`
+ `pushFrame()`
  ```cpp
  // Push a new frame {TmpModInst:{globaddrs}, locals:none}
  StackMgr.pushFrame(TmpModInst.get(), AST::InstrView::iterator(), 0, 0);
  ```
+ `getGlobalSection()`
+ `getExportSection()`

+ `addFuncType()` in `Instantiate()` step.
```cpp
  /// Copy the function types in type section to this module instance.
  void addFuncType(const AST::FunctionType &FuncType) {
    std::unique_lock Lock(Mutex);
    FuncTypes.emplace_back(FuncType);
  }
```
