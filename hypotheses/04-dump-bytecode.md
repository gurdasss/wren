# 04: Dump the compiled bytecode

| | |
|---|---|
| **Date** | 2026-11-08 |
| **Branch** | `experiment/04-dump-bytecode` (commit `99295202`) |
| **Subsystem** | Compiler (`src/vm/wren_compiler.c`, `src/vm/wren_debug.c`) |
| **Consequence level** | 2: debug flag in VM code, no language behavior changed |
| **Status** | done |

## Question
What bytecode does `System.print("hi")` compile to, and what else does the compiler produce?

## Background
- The compiler gives each function its own `Compiler` struct. When a function finishes, `endCompiler` (`src/vm/wren_compiler.c:1661`) emits `END`, and when the flag is on, it calls `wrenDumpCode` (line 1702).
- `wrenDumpCode` (`src/vm/wren_debug.c:361`) uses **`printf` directly**. It does *not* go through the host's `writeFn`.
- At startup, every VM compiles the core library (`wren_core.wren`, see `src/vm/wren_core.c:1299`) before running any user script.

## Hypothesis
*(User's own predictions, recorded before running.)*
1. **Instructions for the line:** no prediction ("I don't know the bytecode vocabulary"). For reference, Claude's earlier guess: `LOAD_MODULE_VAR System`, `CONSTANT "hi"`, `CALL_1 print(_)`, `POP`.
2. **Amount of output:** **Prediction:** it depends on the number of method calls made, and on how much information is dumped per instruction.
3. **Blast radius:** **Prediction:** about **397** tests fail. **Because:** the dump goes to stdout, and the test runner expects exact stdout without any dump information.

## Change
```diff
--- a/src/vm/wren_common.h
+++ b/src/vm/wren_common.h
 // Set this to true to print out the compiled bytecode of each function.
-#define WREN_DEBUG_DUMP_COMPILED_CODE 0
+#define WREN_DEBUG_DUMP_COMPILED_CODE 1  // EXPERIMENT 04
```

## How we tested
```sh
git checkout -b experiment/04-dump-bytecode
# ...apply change, rebuild...
echo 'System.print("hi")' > hi.wren
./bin/wren_test hi.wren > hi_dump.txt     # 2023 lines
python3 util/test.py                      # full suite
```

## Results

### 1. The bytecode for our line
```
./hi: (script)
   1: 0000  LOAD_MODULE_VAR     22 'System'     push the System class (module variable #22)
      0003  CONSTANT             0 'hi'         push constant #0, the string "hi"
      0006  CALL_1              79 'print(_)'   call method symbol #79 with 1 argument
      0009  POP                                 discard print's return value
   2: 0010  END_MODULE                          the module's top level has finished
      0011  RETURN                              return from the script function
      0012  END                                 end-of-bytecode marker (never executed)

hi
```
How to read a line: `1:` is the source line, `0000` is the byte offset, then the opcode and its argument. `LOAD_MODULE_VAR` takes 3 bytes (1 opcode + a 2-byte argument), which is why the offsets go 0, 3, 6, 9. `POP` has no argument, so it takes 1 byte.

### 2. The total output
- **2023 lines** for a one-line script.
- **94 core-library functions** were dumped first (`<core>: all(_)` and so on), then our **1** script function, and *then* `hi` was printed.
- The core's own `System.print(obj)` compiles to bytecode that calls `writeObject_(_)` and then `writeString_(_)`, matching `wren_core.wren:445`.

### 3. Full suite
The **test runner crashed** with a Python `TypeError` after 513 tests, with **0 passed and 513 failed**. The remaining ~350 tests never ran.

| # | Predicted | Actual | Held? |
|---|---|---|---|
| 1 | *(Claude: 4 instructions)* | the same 4, **plus** `END_MODULE`, `RETURN`, `END` | partly: missed the epilogue |
| 2 | depends on the number of **method calls** | depends on the number of **functions compiled**: 94 core + 1 script, before anything runs | ❌ |
| 3 | ~397 fail | **every test that ran failed (513)**, then the runner crashed | ❌ |

## Why
**2: compiling ≠ calling.** The dump runs in `endCompiler`, once per function *compiled*, not per call. Our script made one call, yet 95 functions were dumped, because the VM compiles the whole core library at startup, whether or not you use it. The order (dump first, `hi` last) shows the compile-time → run-time sequence directly.

**3a: why *every* test failed, including error-only tests.** The dump uses `printf`, not `writeFn`, so the host can't intercept or filter it. Every VM compiles the core at startup, so *every* test prints about 2000 lines of dump. The runner checks stdout in **both** directions:
- `Missing expected output ...` (what we saw in [01](01-no-writefn.md))
- `Got output "<core>: all(_)" when none was expected.` (new). Error-only tests like `unexpected_character.wren` expect *empty* stdout, so they fail too.

The prediction of 397 counted only the tests that check printed output. The runner also enforces "no extra output" on everyone else.

**3b: why the runner crashed.** `test/core/string/count.wren` contains a deliberately invalid UTF-8 string literal: `"\xefok\xf7"`. Normally that string is never printed (the test only prints `.count`, which is `4`). But the dump prints every **constant** raw, so invalid UTF-8 bytes reached stdout. `util/test.py:181` then tries to decode them:
```python
try:
  out = out.decode("utf-8")...
  err = err.decode("utf-8")...
except:
  self.fail('Error decoding output.')
                                  # ← missing `return`
error_lines = err.split('\n')     # err is still bytes → TypeError, whole run aborts
```
The `except` records a failure but **doesn't return**, so execution continues with undecoded bytes and crashes. This is a **real bug in the test runner**, hidden until now because no test normally prints invalid UTF-8.

## Surprises
- 2000+ lines of output for a single line of code.
- Debug output goes around the host entirely (`printf`), unlike everything in experiments 01–03.
- Turning on a "harmless" debug flag uncovered a latent bug in a different program (the Python test runner).

## Lessons
- **The compiler works per function, not per call.** Compile cost is paid once for everything defined. Run-time cost is paid per call.
- **Every VM start compiles the core library.** That's a fixed startup cost paid by every host, every time.
- **Bytecode is small and readable.** Each instruction is one byte, plus 0–2 bytes of argument. Names like `System` and `print(_)` become **numbers** (module variable #22, method symbol #79) at compile time, so the run-time work is array lookups.
- **Tests check in both directions:** "everything expected appeared" *and* "nothing unexpected appeared." The second check was invisible in 01–03.
- **Not every output path goes through the host.** The host-boundary model from 01–03 covers the *product's* output, but debug tooling writes straight to the process's stdout.
- **Error handling that records a failure but carries on is a bug pattern.** Look for `except: log(...)` with no `return` or `raise`.
- **A low-consequence change can still expose high-consequence bugs elsewhere.** Consequence level measures how risky the *change* is, not how much it might reveal.

## Follow-ups
- **Real fix:** add `return` after `self.fail('Error decoding output.')` in `util/test.py`. This is a genuine bug that could be contributed upstream.
- **Level 2:** `WREN_DEBUG_TRACE_INSTRUCTIONS` shows each instruction as it *executes*. Compare the run-time trace with this compile-time dump.
- **Level 2:** dump a script with `import`, a `class`, and a `{ }` block function, and see how many functions it produces and what `IMPORT_MODULE` looks like.
