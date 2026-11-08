# 05: Fix the test runner's missing `return` after a decode failure

| | |
|---|---|
| **Date** | 2026-11-08 |
| **Branch** | `fix/test-runner-decode-return` (commit `5ef03b75`, branched from `upstream/main`) |
| **Subsystem** | Test runner (`util/test.py`) |
| **Consequence level** | 1: tooling only, no VM code touched |
| **Status** | done. The fix is on the fork; no upstream PR opened yet. |

## Question
Does adding `return` after `self.fail('Error decoding output.')` stop one bad test from crashing the whole run?

## Background
Found in [04](04-dump-bytecode.md). In `util/test.py`, `validate()` does this:
```python
try:
  out = out.decode("utf-8")...   # raises on invalid UTF-8; the next line never runs
  err = err.decode("utf-8")...
except:
  self.fail('Error decoding output.')
                                  # no return: falls through
error_lines = err.split('\n')     # err is still bytes → TypeError → whole run aborts
```
The `except` is meant for **containment**: fail *this* test and move on. Without `return`, the failure escapes and kills the run.

The fix branch is based on **`upstream/main`**, not on the fork's `main`, so it contains only the fix and none of the `hypotheses/` notes. That keeps it ready for a clean upstream PR.

## Hypothesis
*(Proposed by Claude at the end of [04](04-dump-bytecode.md) and adopted by the user before running.)*
1. **Prediction:** with the fix and the bytecode dump turned on, the suite runs **all 866 tests to completion** instead of crashing after 513.
2. **Prediction:** `test/core/string/count.wren` and the other tests that print invalid UTF-8 are reported as `Error decoding output.`, not as a Python crash.

*(Added by Claude before running, as a regression check.)*
3. **Prediction:** with the fix and a normal build (dump off), all 866 tests still pass, so the fix doesn't change normal behavior.

## Change
```diff
--- a/util/test.py
+++ b/util/test.py
     except:
       self.fail('Error decoding output.')
+      return
```

## How we tested
```sh
git fetch upstream
git checkout -b fix/test-runner-decode-return upstream/main
# ...apply the one-line fix...

# Predictions 1 and 2: turn the dump on TEMPORARILY (never committed)
sed -i 's/DUMP_COMPILED_CODE 0/DUMP_COMPILED_CODE 1/' src/vm/wren_common.h
make -C projects/make wren_test && python3 util/test.py

# Prediction 3: restore the flag, normal build
git checkout src/vm/wren_common.h
make -C projects/make wren_test && python3 util/test.py
```

## Results
| # | Predicted | Actual | Held? |
|---|---|---|---|
| 1 | all 866 run, no crash | **5 passed + 861 failed = 866**, 0 Python tracebacks | ✅ |
| 2 | the invalid-UTF-8 tests report `Error decoding output.` | yes, **11** tests, not 4 (see Surprises) | ✅ (the count was off) |
| 3 | normal build: all 866 pass | `All 866 tests passed (3861 expectations).` | ✅ |

The 11 decode failures, all deliberately invalid UTF-8 string tests:
- `core/string/`: `count`, `iterate`, `iterator_value`, `subscript`
- `core/string_byte_sequence/`: `count`
- `core/string_code_point_sequence/`: `count`, `iterate`, `iterator_value`, `iterator_value_incomplete`, `subscript`, `subscript_incomplete`

## Why
- **1 and 2:** with `return`, a decode failure ends `validate()` for that test only. The runner moves on to the next test, so the failure stays where it belongs.
- **3:** in a normal build no test prints invalid UTF-8, so the `except` branch never runs and the new line never executes. That's also why the bug went unnoticed upstream.
- **Why 11 and not 4:** in 04 the crash happened partway through, and afterwards we scanned only `core/string/`. Two sibling directories, `string_byte_sequence` and `string_code_point_sequence`, also test invalid UTF-8. The earlier search was too narrow.

## Surprises
- **5 tests passed even with 2000 lines of dump in stdout.** They're `example/` scripts: the runner deliberately ignores their stdout (`util/test.py:199`, `if is_example: return`). It also skips `example/skynet.wren` for being slow (`util/test.py:403`). So the 5 passes were `hello`, `mandelbrot`, `syntax` and the two `import_module` files.
- The count of affected tests was nearly 3× our estimate.

## Lessons
- **Error handling must stop, or the error escapes.** `except: log()` without `return`/`raise` turns "fail one test" into "crash the whole run." When reading error-handling code, ask what happens on the line *after* the handler.
- **Search wide before stating a count.** The "4 tests" figure came from looking in one directory. Saying "at least 4" would have been accurate.
- **Test the fix *and* the normal path.** Prediction 3 (the regression check) is what shows the fix is safe to ship, not just effective.
- **Keep test scaffolding out of the fix.** The dump flag was needed to *trigger* the bug but was never committed. The fix branch's diff against `upstream/main` is exactly one line.
- **A test runner has its own policies:** example scripts are exempt from output checks, and some scripts are skipped entirely. A "pass" means "passed this runner's checks," which can be less than you'd assume.

## Follow-ups
- **Upstream PR:** open a PR from `gurdasss:fix/test-runner-decode-return` to `wren-lang/wren:main`. This is public and posted under the user's name, so the user decides.
- The fork's `main` doesn't contain the fix (it's only on the fix branch). Merge or cherry-pick it into the fork's `main` if wanted.
- **Level 2:** `WREN_DEBUG_TRACE_INSTRUCTIONS`, to watch bytecode *execute* and compare with the compile-time dump from 04.
