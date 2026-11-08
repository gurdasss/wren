# 03: Remove the host's `loadModuleFn`

| | |
|---|---|
| **Date** | 2026-11-08 |
| **Branch** | `experiment/03-no-loadmodulefn` (commit `1c1a8050`) |
| **Subsystem** | Host ↔ VM boundary: module loading |
| **Consequence level** | 1: host only, no VM code touched |
| **Status** | done |

## Question
If the host doesn't give the VM a way to load modules, which imports still work?

## Background
- The test runner sets `config.loadModuleFn = readModule;` (`test/main.c:20`). `readModule` (`test/test.c:375`) adds `.wren` to the module name and reads that file from disk.
- The VM's import path (`src/vm/wren_vm.c:745`):
  1. If `loadModuleFn != NULL`, ask the host for the source.
  2. If there's still no source, check the built-ins (`"meta"`, `"random"`), lines 750–760.
  3. If there's still none, it's a runtime error: `Could not load module '...'`.
- `resolveModuleFn` is a separate callback that turns `"./module"` into a full path. It's left in place, so error messages still show resolved paths.
- **The main script is not loaded via `loadModuleFn`.** The runner reads it itself (`runFile`) and passes the text to `wrenInterpret`. So every test file still *starts*.

## Hypothesis
*(User's own prediction, recorded before running.)*
1. **Prediction:** imports of host-level modules (`.wren` files on disk) fail, and **only** built-in modules (`random`, `meta`) still import.
   **Because:** built-ins are compiled into the VM (`src/vm/wren_vm.c:750`), while file modules need the host's loader to read them from disk.
2. **Blast radius (number of failing tests):** no prediction recorded.

## Change
```diff
--- a/test/main.c
+++ b/test/main.c
   config.resolveModuleFn = resolveModule;
-  config.loadModuleFn = readModule;
+  // config.loadModuleFn = readModule;  // EXPERIMENT 03: remove the host's module loader
   config.writeFn = vm_write;
```

## How we tested
```sh
# Baseline on a clean build: full suite = all 866 pass
python3 util/test.py
git checkout -b experiment/03-no-loadmodulefn
# ...apply change, rebuild...
./bin/wren_test test/language/module/simple_import/simple_import.wren; echo $?   # file import
./bin/wren_test test/random/new.wren; echo $?                                    # built-in import
./bin/wren_test test/language/module/unknown_module.wren; echo $?               # expects load failure
python3 util/test.py                                                             # full suite
```

## Results
| Probe | Baseline | After | |
|---|---|---|---|
| `simple_import` (file import) | prints `ran module`, `from module`, exit 0 | `Could not load module './test/language/module/simple_import/module'.`, exit **70** | ✅ fails as predicted |
| `random/new` (built-in) | prints `200`, exit 0 | prints `200`, exit 0 | ✅ works as predicted |
| `unknown_module` (expects failure) | runtime error, exit 70, PASS | runtime error, exit 70, **PASS** | see Surprises |

**Full suite: 847 passed, 19 failed** (of 866).
- All 19 failures are tests that import a `.wren` file: 16 in `test/language/module/`, plus `test/api/get_variable.wren`, `test/core/fiber/yield_from_import.wren` and `example/import_module/lovecraft.wren`.
- None of the 25 tests that import `random` or `meta` failed.

| # | Prediction | Held? |
|---|---|---|
| 1 | file imports fail, only built-ins work | ✅ yes |
| 2 | *(none)* | n/a |

## Why
`readModule` was the only code that touched the disk. Without it, step 1 of the import path is skipped. Built-ins are found in step 2, and everything else falls through to step 3's error.

**Exit code 70** is the test runner's code for a *runtime* error (`WREN_RESULT_RUNTIME_ERROR`). A compile error gives 65 (see [02](02-no-errorfn.md)). The failure is a runtime error because **`import` runs at runtime**: the compiler only turns `import "x"` into an instruction, and the actual load happens when the interpreter reaches that instruction.

## Surprises
Four tests contain a file import, yet still **passed**:

| Test | Why it survived |
|---|---|
| `language/module/unknown_module.wren` | It *expects* `Could not load module`. Our breakage produced the same error the test wanted. |
| `language/module/name_collision.wren` | It expects a **compile** error (`var Collides` then `import ... for Collides`). The compiler rejects the file before any import runs. |
| `language/module/missing_for.wren` | Same: a syntax error is caught at compile time, so the import never runs. |
| `example/syntax.wren` | The import is inside a method named `imports {}` that's never called, so the import instruction is never executed. |

All four point to the same fact: **an import only matters if it executes.** Having `import` in a file isn't enough to make the test depend on the loader.

## Lessons
- **The hypothesis held cleanly.** The VM's import path really is "host first, then built-ins, then error." The two kinds of module are independent.
- **Compile time vs. run time.** `import` looks like a declaration at the top of a file, but it's an action that runs. Code that never runs can't fail, even when it's broken. The same reason explains why the `random` tests in [02](02-no-errorfn.md) could hide failures.
- **A broken system can still produce the "expected" failure.** `unknown_module` passed because the test expected an import error and the break also produced an import error. A test can pass for the wrong reason.
- **Small blast radius = a well-isolated component.** Only 19 of 866 tests (2.2%) touched the loader, compared with 469 for `writeFn`. Module loading is used by few tests, while printing is used by almost all of them.
- **Tooling gotcha:** in `cmd | head; echo $?`, `$?` is the exit code of `head`, not of `cmd`. The first probe run reported `exit=0` incorrectly. Measure exit codes without a pipe, or use `${PIPESTATUS[0]}`.

## Follow-ups
- Level 2: `WREN_DEBUG_DUMP_COMPILED_CODE` to see the bytecode for `System.print("hi")`, including how `import` compiles into an instruction.
- Level 1 variant: keep `loadModuleFn` but make it return the source of a file for `"random"`. Does the host's version *override* the built-in one? (Step 1 runs before step 2.)
