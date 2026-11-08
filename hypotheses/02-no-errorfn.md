# 02: Remove the host's `errorFn`

| | |
|---|---|
| **Date** | 2026-11-08 |
| **Branch** | `experiment/02-no-errorfn` (commit `c37a832`) |
| **Subsystem** | Host ↔ VM boundary (`WrenConfiguration` callbacks) |
| **Consequence level** | 1: host only, no VM code touched |
| **Status** | done |

## Question
Mirror of [01](01-no-writefn.md): if the host gives the VM no way to report errors, but printing still works, what does each judge say?

## Background
- The test runner sets `config.errorFn = reportError;` (`test/main.c:22`). The default is `NULL` (`src/vm/wren_vm.c:52`).
- Both the compiler and the runtime check for `NULL` before reporting, and if it's `NULL` they just `return`: `src/vm/wren_compiler.c:427` and `src/vm/wren_debug.c:8`.
- The exit code doesn't come from `errorFn`. `test/main.c` maps the result of `wrenInterpret` to an exit code: `WREN_RESULT_COMPILE_ERROR` → `65` (`WREN_EX_DATAERR`).
- `util/test.py` decides whether an error test passed by reading the **error text on stderr** (it looks for `[... line N] Error`) and matching it against `// expect error` comments. It doesn't rely on the exit code alone.

## Hypothesis
*(User's own predictions, recorded before running.)*
1. **Single error test** (`test/language/unexpected_character.wren`): **Prediction:** no error message is shown, but the process still returns the compile-error result. The test runner reports FAIL. **Because:** with `errorFn` NULL there's nowhere to send the message, but the result code is separate from the message.
2. **Full suite:** **Prediction:** the 469 tests that failed in 01 now PASS, and the 397 that passed in 01 now FAIL. **Because:** `writeFn` is back and those 469 tests check printed output, while the 397 check errors, which can no longer be reported.
3. **Will any test fail in both 01 and 02?** No prediction ("hard to say with certainty").

## Change
```diff
--- a/test/main.c
+++ b/test/main.c
@@ -19,7 +19,7 @@ static WrenVM* initVM(bool isAPITest)
   config.loadModuleFn = readModule;
   config.writeFn = vm_write;
-  config.errorFn = reportError;
+  // config.errorFn = reportError;  // EXPERIMENT 02: remove the host's error callback
```

## How we tested
```sh
git checkout -b experiment/02-no-errorfn
./bin/wren_test test/language/unexpected_character.wren; echo $?   # baseline (clean build)
python3 util/test.py language/unexpected_character
# ...apply change, rebuild, rerun the same commands...
python3 util/test.py                                              # full suite
# Then rebuilt experiment 01 and diffed the two lists of failing tests.
```

## Results
| # | Predicted | Actual | Held? |
|---|---|---|---|
| — | *(baseline)* | `Error at '^': Expected expression.`, exit `65`, PASS | — |
| 1 | no message, compile-error exit code, FAIL | no message, exit **`65`**, FAIL: `Missing expected error on line 3.` | ✅ |
| 2 | 469 pass / 397 fail (an exact swap) | **472 pass / 394 fail** | ⚠️ direction right, swap not exact |
| 3 | *(none)* | **14 tests fail in both**, and **17 pass in both** | n/a |

Small correction on terms: `COMPILE_ERROR` isn't a C macro. It's `WREN_RESULT_COMPILE_ERROR`, a value of the `WrenInterpretResult` **enum** in `wren.h`. The test runner's host then turns it into the exit code `65`.

## Why
**Why the swap wasn't exact (results 2 and 3):** the prediction assumed every test belongs to exactly one group, either "checks output" or "checks errors." In reality there are **four** groups:

| Checks output? | Checks errors? | 01 (no `writeFn`) | 02 (no `errorFn`) | Count |
|---|---|---|---|---|
| ✅ | ❌ | FAIL | PASS | |
| ❌ | ✅ | PASS | FAIL | |
| ✅ | ✅ | FAIL | FAIL | **14** |
| ❌ | ❌ | PASS | PASS | **17** |

- **Both (fail in both).** For example, `test/core/list/swap.wren` prints results (`// expect: [3, 1, 2, 0, 4]`) and then expects a runtime error (`// expect runtime error: ...`). Losing either channel breaks it. Most of the 14 are fiber and error-handling tests, plus API tests.
- **Neither (pass in both).** These tests have nothing to match: `empty_file.wren`, comment-only files, the `example/` scripts (no `// expect:` lines), and the `random` range tests that print nothing on success. A test with no expectations can't miss one.

The arithmetic: 469 − 14 = 455 output-only tests. 394 − 14 = 380 error-only tests. 455 + 380 + 14 + 17 = 866 ✅

## Surprises
- 14 + 17 = 31 tests (about 3.6%) didn't follow the "it's one or the other" model.
- The `example/` scripts and `random` tests "pass" even when the VM can't print *or* report errors. They test very little on their own.

## Lessons
- **Hidden assumption: "categories don't overlap."** The prediction was right in direction and wrong in detail because it assumed each test was in exactly one group. When predicting a swap, ask: *could something be in both groups, or in neither?*
- **Exit code ≠ error message.** They travel by different routes: the return value of `wrenInterpret` vs the `errorFn` callback. This confirms the idea from 01.
- **A test with no expectations is a weak test.** It passes no matter what the VM does. That's worth knowing before trusting a green test run.
- **Process improvement:** this time the predictions were written down *before* running. That's what let us see exactly where the mental model was off.

## Follow-ups
- Remove `loadModuleFn`. Predict which import tests fail and whether `import "random"` still works (built-in modules, `src/vm/wren_vm.c:750`).
- Level 2: turn on `WREN_DEBUG_DUMP_COMPILED_CODE` to see the bytecode for `System.print("hi")`.
