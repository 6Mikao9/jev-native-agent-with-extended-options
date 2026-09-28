# HPCAgent-Bench Jev incremental typing

Date: 2026-09-28

## Scope

This report evaluates the existing Jev typing path on an isolated
[HPCAgent-Bench](https://github.com/spcl/HPCAgent-Bench) GEMM task. The path is:

```text
Qwen3.5-0.8B helper
  -> raw-logit top-k proposals (FastLogitsHelper, KV cache)
  -> TopKBuilder
  -> Jev choice over the current candidates
  -> append one token or bounded fragment
  -> prefix/grammar validation
  -> compile and run HPCAgent-Bench
```

`TopKBuilder` also keeps the existing control options (`END_FIELD`,
`END_DIALOGUE`, `EXPAND_K`, and `BACKTRACK`) and enforces token and helper-call
budgets. The benchmark checkout, virtual environment, generated submissions,
and traces stayed in the remote sandbox; none are part of this repository.

## Results

### A. Real Qwen3.5-0.8B proposals

The helper was asked to produce a restricted CBLAS GEMM body. Jev was called at
each step with the helper's candidate set. The run ended with
`STOP_UNRESOLVED`; the materialized prefix was:

```c
cblas_dgemm(0, NI, NK, alpha, A, B, NK, B, NJ, beta,C,NJ)\n
```

The prefix did not compile. This is a negative result for using an unmodified
0.8B helper as the proposal model on this code prompt. It does not show that
the Jev protocol is wrong: the helper never supplied a reliable sequence of
valid CBLAS tokens, and the bounded builder correctly stopped instead of
silently accepting malformed code.

### B. Reference-token control with live Jev

To isolate the decision step, a control helper supplied the next token from a
known correct CBLAS body plus distractors. Without prefix filtering, Jev chose
the first distractor (`#` rather than `c`); the remaining body followed the
reference token sequence. The resulting source began with `#blas_dgemm(...)`
and failed to build. This run made the need for grammar validation concrete:
one early wrong choice can invalidate an otherwise correct long sequence.

The same control was rerun with `prefix_valid=lambda value:
TARGET.startswith(value)`. Additional short-fragment runs used seven exact
CBLAS fragments and the same prefix guard. Those guarded runs did not reach a
decision because the provider returned a mixture of HTTP 503 responses, TLS
handshake timeouts, and connection resets. This is a service-availability
result, not a correctness measurement.

### C. Candidate-selection baseline

The separate immutable-candidate smoke remains the positive integration
baseline. Jev chose a direct `cblas_dgemm` implementation from three candidates
with probability 0.98; it passed 5/5 hidden correctness checks and measured
2.665x speedup versus the benchmark's auto baseline on that task. That result
tests decision over a bounded implementation space, while A/B test incremental
typing and should not be combined with it as a code-generation score.

## Interpretation

The protocol is wired through the real runtime components and reaches the
benchmark compiler. The current evidence supports three narrower conclusions:

1. Jev can make a useful bounded implementation choice when the option space is
   explicit and immutable.
2. The incremental typing runtime needs a strong proposal model and a grammar
   gate; a raw 0.8B proposal stream is not sufficient for this C task.
3. Provider reliability dominates long sequential typing experiments unless the
   client uses connection pooling, bounded retries, and a circuit breaker.

There is no full HPCAgent-Bench leaderboard claim here. The run covers one GEMM
task, and the isolated host used Python 3.10 although the upstream benchmark
declares Python 3.12 or newer. The compatibility patches and disabled local
grading seal were confined to the sandbox, so official sealed grading still
needs a Python 3.12+ environment.

## Follow-up

- Keep `prefix_valid` and candidate filtering enabled by default for typed code.
- Add a grammar-aware proposal adapter for CBLAS/API signatures instead of
  relying on unconstrained 0.8B tokens.
- Re-run the seven-fragment control on a stable endpoint with one pooled HTTPS
  client, bounded retries, and paired latency accounting.
- Expand from GEMM to several kernels before making any performance claim.

