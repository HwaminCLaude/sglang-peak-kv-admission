[한국어](README.md) · English

# Why long LLM answers freeze mid-stream for up to about a minute

**SGLang 0.5.18 · the cause of request retraction under KV-cache pressure, a <!--n:patch.lines_added-->102<!--/n-->-line patch, and what it costs**

[![cpu-tests](https://github.com/HwaminJung00/sglang-peak-kv-admission/actions/workflows/cpu-tests.yml/badge.svg)](https://github.com/HwaminJung00/sglang-peak-kv-admission/actions/workflows/cpu-tests.yml)

Qwen3-4B · one RTX 4090 · <!--n:trace.n_requests-->120<!--/n--> synthetic requests with answer lengths fixed in advance

When SGLang 0.5.18 runs out of KV-cache memory, it takes back the memory of a request that is still generating and sends that request to the back of the queue. This is a *retraction*, and the retracted answer stops until its turn comes again. The cause lies in the default scheduler. It reserves far less KV memory for the running requests than they will use. It may reserve only about 400 tokens for a request with 10,000 tokens still to generate, so it admits too many new requests and soon runs out of memory. At 2 requests per second, well above what the GPU can sustain, <!--n:q2.default.stall5_range-->19–20<!--/n--> of <!--n:trace.n_requests-->120<!--/n--> answers froze mid-stream for <!--n:q2.default.stall_range_s-->30.8–64.8<!--/n--> s each in all three runs.

To confirm that a stall is a retraction, I ran unpatched SGLang at 7 loads from 0.5 to 8 req/s. The retractions in the server logs and the pauses of more than 5 s measured at the client matched one to one (<!--n:diag.stall_match-->89/89<!--/n-->).

For requests of fixed length, SGLang already has an exact calculation that counts every remaining output token, but it runs only when the prompt-sharing cache (the radix cache) is off. I wrote a <!--n:patch.lines_added-->102<!--/n-->-line patch that adds the same calculation as an extra admission check when the cache is on. With the patch's option (`--enable-peak-kv-reservation`, off by default) turned on, <!--n:q2.mine.stall5-->0<!--/n--> requests stall at 2 req/s (three runs), and the existing option that turns the cache off, `--disable-radix-cache`, also gives <!--n:q2.noradix.stall5-->0<!--/n--> (one run). In exchange, the patch raises the p99 time to first token (TTFT) from <!--n:q2.default.ttft_p99_s-->64.2<!--/n--> s to <!--n:q2.mine.ttft_p99_s-->92.3<!--/n--> s. All of this was measured with every request generating until it reached a fixed length, so the server knew each answer's length in advance.

This was an individual assignment in a 5-week LLM inference-engine course, and the patch follows a hint given by the course. After submission, an AI tool helped with verification and documentation ([who did what](#who-did-what)).

**Mid-answer stalls (red) turn into waiting before the first token (grey)**

![Request timelines at 2 req/s: default vs. patch](docs/figures/hero_timeline.png)

*Each row is one request, and blue is the time it spends generating. In the figure, q2 means 2 req/s, "default (flag off)" is the default with the patch's option off, and α=1.0 means the patch reserves all of the remaining length. Each panel shows one run out of three (r2), so the range of the stalls differs slightly from the text. The "09-07 sweep" in the bottom line of the figure is the 7-load experiment above.*

## Results at a glance

| 2 req/s, <!--n:trace.n_requests-->120<!--/n--> requests | Requests stalled mid-answer | p99 of each request's worst token gap | TTFT p99 | Goodput |
|---|---|---|---|---|
| Default: patched build, option off (median of 3) | <!--n:q2.default.stall5-->19<!--/n--> | <!--n:q2.default.maxitl_p99_s-->63.1<!--/n--> s | <!--n:q2.default.ttft_p99_s-->64.2<!--/n--> s | <!--n:q2.default.goodput-->0.403<!--/n--> |
| **Patch**: option on (median of 3) | <!--n:q2.mine.stall5-->0<!--/n--> | <!--n:q2.mine.maxitl_p99_s-->0.11<!--/n--> s | <!--n:q2.mine.ttft_p99_s-->92.3<!--/n--> s | <!--n:q2.mine.goodput-->0.440<!--/n--> |
| `--disable-radix-cache` (existing option, one run) | <!--n:q2.noradix.stall5-->0<!--/n--> | <!--n:q2.noradix.maxitl_p99_s-->0.12<!--/n--> s | <!--n:q2.noradix.ttft_p99_s-->89.9<!--/n--> s | <!--n:q2.noradix.goodput-->0.435<!--/n--> |
| `--max-running-requests 40` (existing option, one run) | <!--n:q2.tuned_n40.stall5-->0<!--/n--> | <!--n:q2.tuned_n40.maxitl_p99_s-->0.09<!--/n--> s | <!--n:q2.tuned_n40.ttft_p99_s-->93.5<!--/n--> s | <!--n:q2.tuned_n40.goodput-->0.432<!--/n--> |

The worst token gap is the longest pause in one request's stream. Goodput is the number of requests whose mean time per output token was <!--n:trace.tpot_slo_ms-->60<!--/n--> ms or less, divided by the time from the first request sent to the last answer finished (req/s); it ignores TTFT. Half of the goodput difference comes from the single longest answer finishing earlier ([worked out in the technical summary](docs/TECHNICAL_SUMMARY.md#how-to-read-these-numbers)). With unpatched SGLang, <!--n:q2.upstream.stall5-->19<!--/n--> requests stalled, like the default (one run).

## Cause and evidence

With the default settings, SGLang reserves only `min(remaining tokens, 4096) × new_token_ratio` per running request ([code](third_party/sglang_v0_5_18/schedule_policy.py#L654)), and that ratio decays from 0.7 to 0.098. The exact calculation is [here](third_party/sglang_v0_5_18/schedule_policy.py#L1065), and it runs only when the radix cache is off ([code](third_party/sglang_v0_5_18/schedule_policy.py#L1213)).

1. **One retraction, one stall.** In the <!--n:diag.stall_match-->89/89<!--/n--> match above, no stall was left unmatched (one run per load).
2. **Retractions begin near the computed limit.** The highest arrival rate the KV pool can sustain without retraction, computed from its size and the answer lengths, is <!--n:diag.little_lambda_max-->0.56–0.67<!--/n--> req/s. In the sweep there were no retractions at 0.5 req/s, and the first came at <!--n:diag.first_retraction_qps-->0.6<!--/n--> req/s.
3. **The stall is time spent waiting in the queue, not recomputation.** In the <!--n:diag.reprefill_confirmed-->38<!--/n--> cases where the log shows when the retracted request ran again, the time from the retraction to that moment was within <!--n:diag.reprefill_delay_vs_stall_max_s-->0.90<!--/n--> s of the stall.
4. **At 2 req/s, admitting more conservatively removes the retractions.** None of the three methods in the table had any, while a comparison run that turned off chunked prefill, which should be unrelated, still had them ([numbers in the technical summary](docs/TECHNICAL_SUMMARY.md#diagnosis-why-long-answers-stall)).

## What this result does not show

- **The pause moved; it did not go away.** Requests waiting more than 10 s for the first token rose from <!--n:q2.default.wait10-->43<!--/n--> to <!--n:q2.mine.wait10-->60<!--/n--> (medians of 3). This is not usable as-is for interactive services.
- **Nothing shows that the patch beats the existing options.** The goodput differences in the table are small, and the options were run once. The patch differs in keeping the radix cache on, but these requests share a prompt prefix of only <!--n:trace.shared_prefix_tokens-->36<!--/n--> tokens, so this workload cannot show what that is worth.
- **Answer lengths are known in advance.** For requests of unknown length the maximum (`max_tokens`) is only an upper bound, so reserving all of it is far too conservative. That kind of traffic was not measured.
- **One setup.** One GPU, one model, one SGLang version and one request set, and `--schedule-conservativeness` was not measured.

## Who did what

- **Provided by the course.** The request set, the measurement harness, the protocol, and the hint to reserve peak KV conservatively when admitting a request ([course repository](https://github.com/mlleo/inference-engine-study)).
- **My work.** The measurements at each load, the one-to-one match between retractions and stalls, finding that the exact calculation is off with the default settings, the patch, the experiment automation and <!--n:budget.runs-->53<!--/n--> runs, the selection rule fixed before the main runs, and the analysis. All GPU runs were made before submission.
- **With an AI tool after submission ([Claude Code](https://claude.com/claude-code)).** Recomputing and verifying from the same data, the equivalence tests, the metrics module, the figures, and drafts of the corrected documents, this page included. I reviewed the published version.

## Check it yourself, and read more

Without a GPU, one minute is enough to check the patch's decision logic and the numbers in the documents. It does not repeat the GPU measurements.

```bash
python3 tests/run_cpu_tests.py               # unit tests of the patch's decision logic
python3 tests/test_upstream_equivalence.py   # verdicts against the unmodified SGLang function
python3 tools/check_numbers.py               # every marked number in the docs against the data
```

- [Patch](patch/README.md): +<!--n:patch.lines_added-->102<!--/n-->/−<!--n:patch.lines_removed-->0<!--/n--> lines. It only adds one more condition on top of SGLang's own checks, so it can hold a request back but never admits one that SGLang would have held back ([commit on a fork](https://github.com/HwaminJung00/sglang/commit/5157a47e8649c6b3cc43705114fa8ec41d8d733d))
- [Technical summary](docs/TECHNICAL_SUMMARY.md), 10 minutes: results by load, comparison runs, method, hard questions, reproduction
- [Full report](docs/REPORT.md), 40 minutes: the complete analysis and appendices
- [Errata](docs/ERRATA.md): 34 errors and omissions in the submitted report. The 2 req/s measurements are unchanged, and 7 high-impact items change how those measurements should be read
- [CPU tests](tests/README.md) · [Data](data/README.md) · [Experiment log](docs/EXPERIMENTS.md) · [Plan written before the main runs](docs/preregistration/plan_2026-09-13.md) (Korean)

Code is under Apache-2.0, and documents and data are under CC BY 4.0 ([LICENSE](LICENSE), [LICENSE-docs](LICENSE-docs), [NOTICE](NOTICE)). This is a personal repository, not affiliated with the SGLang project.
