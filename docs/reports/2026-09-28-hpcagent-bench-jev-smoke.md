# HPCAgent-Bench Jev candidate-selection smoke

Date: 2026-09-28

## What was tested

The isolated run used the public [HPCAgent-Bench](https://github.com/spcl/HPCAgent-Bench)
harness with one small task:

- kernel: gemm
- language: restricted C
- precision: FP64
- residency: host
- preset: S
- execution: native, one proposal round, 20 timing repetitions
- correctness: the benchmark's public and hidden tests

Jev did not generate arbitrary C in this experiment. The adapter materialized three
immutable implementations and asked Jev to choose one:

1. direct_cblas: direct cblas_dgemm with the task's alpha and beta, avoiding temporary buffers.
2. reference: the generated HPCAgent-Bench reference implementation.
3. naive_loop: a scalar triple loop preserving the GEMM ABI and semantics.

This isolates the decision-model question from unrestricted code generation. HPCAgent-Bench
still compiled the selected source, ran correctness checks, and measured it.

## Result

The run completed successfully:

| Metric | Value |
|---|---:|
| Jev choice | direct_cblas |
| Jev probabilities | 0.98 direct / 0.01 reference / 0.01 naive |
| Jev confidence | 0.97 |
| Jev API latency | 873.8 ms |
| Jev input/output tokens | 4,067 / 44 |
| Hidden correctness | 5 / 5 |
| Maximum relative error | 4.26e-15 |
| Native kernel time | 3.679 ms |
| Auto baseline time | 9.804 ms |
| Speedup vs auto | 2.665x |
| End-to-end trajectory time | 13.49 s |

A separate earlier reference-agent smoke also passed 5/5 hidden tests at 2.495x
versus auto. These timings were separate runs, so the difference is only a
directional signal; it is not a paired performance comparison.

## Interpretation

This is a positive integration result: Jev selected the faster candidate among a
small, explicitly described option space, and the selected implementation passed all
hidden checks. It demonstrates a useful HPCAgent-Bench adapter for a decision model.

It is not a full HPCAgent-Bench score and it does not establish that Jev can replace a
code-generating agent. The candidate library currently covers only gemm/C, and one
timed task is too small for a general performance claim.

## Isolation and compatibility notes

All benchmark source, virtual-environment files, generated submissions, and Jev decision
traces remain under the remote project sandbox:

/home/liuyuntao/jev-agent-prototype/.bench-sandbox/

The remote host has Python 3.10 while the benchmark declares Python 3.12 or newer, and
GCC 11 accepts -std=c2x rather than -std=c23; those compatibility edits were made
only inside the isolated checkout. The smoke used HPCAGENT_BENCH_GRADING_SEAL=false
because this Python 3.10 environment lacks os.unshare; official container-sealed
grading still needs a Python 3.12+ environment. API credentials were supplied through a
temporary file and were not written to the repository or benchmark artifacts.

## Next experiment

For a meaningful comparison, extend the adapter with several small kernels and run
paired repeats for reference, Jev selection, and a stronger candidate set. Keep the
unrestricted code-generation track separate so the results are not conflated.

