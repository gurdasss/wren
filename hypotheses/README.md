# Hypotheses

A log of hypothesis-driven experiments on the Wren VM. Each experiment makes one small, isolated change, predicts the outcome **before** running it, and then records what actually happened and why.

## Rules

1. **One experiment = one branch** (`experiment/NN-short-name`). The change lives only on that branch. This log lives on `main`.
2. **Write the prediction before running anything.** A prediction made after seeing the result doesn't count, and if none was made, the write-up says so.
3. **Lowest consequence first.** Climb the ladder below one rung at a time.
4. **Measure, don't assume.** Run the full suite as well as one test to see the blast radius.

## Consequence ladder

| Level | Where the change lives | Example |
|---|---|---|
| 1 | Host / test runner (`test/`) | Remove a config callback |
| 2 | Debug flags (`wren_common.h`) | Dump compiled bytecode |
| 3 | Core library (`wren_core.wren`, `wren_core.c`) | Change a built-in method |
| 4 | Compiler (`wren_compiler.c`) | Change how a construct compiles |
| 5 | Interpreter / object model / GC (`wren_vm.c`, `wren_value.c`) | Change dispatch, value representation, rooting |

## Index

| # | Title | Level | Branch | Prediction held? |
|---|---|---|---|---|
| 01 | [Remove the host's `writeFn`](01-no-writefn.md) | 1 | `experiment/01-no-writefn` | Yes (2/2), plus one unpredicted finding |

## Template

Copy this for each new experiment as `NN-short-name.md`.

```markdown
# NN: <Title>

| | |
|---|---|
| **Date** | YYYY-MM-DD |
| **Branch** | `experiment/NN-short-name` |
| **Subsystem** | <host / compiler / interpreter / core / GC ...> |
| **Consequence level** | N (see ladder in README) |
| **Status** | planned / done |

## Question
What are we trying to find out? One or two sentences.

## Background
What we knew going in, with file:line references to the code involved.

## Hypothesis
Numbered predictions, each with its reasoning. Written BEFORE running.
1. **Prediction:** ... **Because:** ...

## Change
The exact diff on the experiment branch.

## How we tested
The exact commands, including a baseline run before the change.

## Results
| # | Predicted | Actual | Held? |
|---|---|---|---|

## Why
The mechanism behind each result, especially wherever a prediction failed.

## Surprises
Anything we didn't predict.

## Lessons
The systems-thinking takeaways, in short bullets.

## Follow-ups
Experiments this one suggests.
```
