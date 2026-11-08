# 01: Remove the host's `writeFn`

| | |
|---|---|
| **Date** | 2026-11-08 |
| **Branch** | `experiment/01-no-writefn` (commit `3130068`) |
| **Subsystem** | Host ↔ VM boundary (`WrenConfiguration` callbacks) |
| **Consequence level** | 1: host only, no VM code touched |
| **Status** | done |

## Question
If the host doesn't give the VM a way to print, what does the VM do, and what do the different "judges" (the VM's exit code and the test runner) say about it?

## Background
- The VM has no I/O of its own. The host passes in function pointers through `WrenConfiguration`. The test runner sets them in `test/main.c:21` (`config.writeFn = vm_write;`).
- By default `writeFn` is `NULL` (`src/vm/wren_vm.c:51`).
- `System.print` is written in Wren (`src/vm/wren_core.wren:445`) and bottoms out in the C primitive `system_writeString` (`src/vm/wren_core.c:1211`), which does:
  ```c
  if (vm->config.writeFn != NULL)
    vm->config.writeFn(vm, AS_CSTRING(args[1]));
  ```
- Errors go through a *separate* callback, `errorFn`, which this experiment leaves alone.
- `util/test.py` judges a test by comparing stdout against `// expect:` comments in the `.wren` file.

## Hypothesis
1. **Prediction:** the VM treats "nothing was printed" as success, so the exit code is still `0`.
   **Because:** the code checks for `NULL` and skips the call. That isn't an error path.
2. **Prediction:** `util/test.py` reports the test as **FAIL**.
   **Because:** the runner checks *output*, not exit codes. No output means the `// expect:` lines aren't matched.
   *(Note: the user's answer addressed the VM's verdict. The runner's verdict was pointed out by Claude before running, so this one isn't a fully independent prediction.)*
3. **Full-suite blast radius:** no prediction was written down before running. ⚠️ A process gap, see Lessons.

## Change
```diff
--- a/test/main.c
+++ b/test/main.c
@@ -18,7 +18,7 @@ static WrenVM* initVM(bool isAPITest)
   config.resolveModuleFn = resolveModule;
   config.loadModuleFn = readModule;
-  config.writeFn = vm_write;
+  // config.writeFn = vm_write;  // EXPERIMENT 01: remove the host's print callback
   config.errorFn = reportError;
```

## How we tested
```sh
git checkout -b experiment/01-no-writefn
make -C projects/make wren_test                      # baseline build (before change)
./bin/wren_test test/core/bool/to_string.wren; echo $?
python3 util/test.py core/bool/to_string
# ...apply change, rebuild, run the same two commands again...
python3 util/test.py                                 # full suite
```
Test used (`test/core/bool/to_string.wren`):
```dart
System.print(true.toString)  // expect: true
System.print(false.toString) // expect: false
```

## Results
| # | Predicted | Actual | Held? |
|---|---|---|---|
| — | *(baseline)* | prints `true`/`false`, exit `0`, test PASS | — |
| 1 | exit code `0` | nothing printed, exit `0` | ✅ |
| 2 | test runner FAIL | FAIL: `Missing expected output "true" on line 1.` (and line 2) | ✅ |
| 3 | *(none recorded)* | full suite: **397 passed, 469 failed** (of 866) | n/a |

## Why
- **1 and 2:** the VM and the test runner check different things. The VM asks "did the code run without an error?" The runner asks "did it produce the right output?" Skipping a `NULL` callback satisfies the first and fails the second.
- **3:** the 397 survivors don't depend on printed output. For example, `test/language/unexpected_character.wren`:
  ```dart
  System.print("no")
  // Something that's not a valid character:
  ^ // expect error
  ```
  This test only checks that a compile error is reported. Errors go through `errorFn`, which still works. `System.print("no")` never runs at all, because the compiler rejects the whole file first.

## Surprises
- Removing the *only* way to print broke just 54% of the suite. The rest of the suite tests error behavior.

## Lessons
- **"Success" depends on who's checking.** Each layer (VM, test runner, human) has its own definition of "correct."
- **Output and errors are separate channels** (`writeFn` vs `errorFn`). Breaking one leaves the other working.
- **Blast radius is a measurement, not a guess.** One test only showed that printing broke. The full suite showed how the system is wired.
- **Process:** write *every* prediction down before running, including the blast radius. The surprise in #3 would have taught more if we'd recorded a guess first.

## Follow-ups
- **02:** remove `errorFn` instead (keep `writeFn`). How many of the 397 survivors fail? Do any of the 469 come back?
- Later (level 2): turn on `WREN_DEBUG_DUMP_COMPILED_CODE` to see the bytecode for `System.print("hi")`.
