# DeepSeek V4 Flash on two GB10 nodes — tuning and benchmark notes

Measurement notes from serving `DeepSeek-V4-Flash-0731` across 2x GB10 (SM121)
over vLLM with DSpark speculative decoding, driving a real coding-agent workload
rather than a synthetic benchmark.

Most of what follows is **negative results**. Nine of the eleven knobs we tried
returned nothing or made things worse, and two of our own explanations turned out
to be wrong under scrutiny. Those are recorded with the evidence that killed
them, because they were the expensive part.

---

## Hardware and stack

```text
2x ASUS Ascent GX10 (NVIDIA GB10, SM121), 121 GB unified memory per node
interconnect  ConnectX-7, RoCE, ~109 Gbit/s per path, 185.13 Gbit/s dual-path
NCCL          16 GiB AllGather 21.40 GB/s busbw, 0 wrong
topology      TP=2 across both nodes, PP=1, EP off
```

### Exact pins

Everything below is pinned; the numbers in this document are only meaningful
against these versions.

**Model**

```text
repo              deepseek-ai/DeepSeek-V4-Flash-0731
revision          7872f01b1d1fe23eabc4c98b48bffcef5a386062
shards            48
weight bytes      166,886,535,336
config.json       sha256 6c8f3d2d3b48707541b88f32f22ef3f0f8a6b57d8523281e2b8d3cdb0ae9a023
tokenizer.json    sha256 8f9f37ca37fdc4f5fd36d5cf4d3b0e8392edb4e894fd10cc0d70b4957c8633cf
architecture      DeepseekV4ForCausalLM     43 layers, hidden 4096
quantization      deepseek_v4_fp8
KV cache dtype    nvfp4_ds_mla, block size 256
```

**Runtime**

```text
image             ghcr.io/anemll/dspark-vllm-gx10:0.1.1
                  @sha256:a83948492cf13df455170fb42885f5ef4db54fefe0feff0f841ecbff464ac9d8
vLLM              v0.25.2.dev0+g752a3a504.d20260714
recipe commit     f752cd04ab30f2cf42077dd8811a5e1e682d63e7
model recipe rev  9e165c30e2704aec5d9d593cce3eebd58bbef1cb
speculative       dspark, num_spec_tokens 5
MoE backend       flashinfer_b12x
```

**Host**

```text
OS                Ubuntu 24.04.4 LTS
kernel            6.17.0-1029-nvidia
NVIDIA driver     580.173.02
CUDA              13.0 (nvcc 13.0.88)
Docker            29.2.1
container toolkit 1.20.0-1
NCCL              2.30.7
ConnectX-7 fw     28.45.4028
platform fw       GX10DGX.0105.2026.0505.1153
GPU clocks        default (no -lgc lock); SM 2411 MHz, ceiling 3003 MHz
```

**TP=2 across nodes is not a choice.** The checkpoint is 166.9 GB against 121 GB
of unified memory per node, so it cannot fit on one. Everything below assumes
that constraint.

The workload is a coding agent (OpenAI `codex` CLI) doing real repository tasks —
implementation, review, survey, planning — not fixed-length generation. Contexts
run 27,604-61,335 tokens. A dispatch takes ~21 turns.

---

## Performance report

The settled configuration is **12 seats served, 16 concurrent agent dispatches**.
All figures below are from that cell unless stated.

### Headline

```text
dispatches/hour            35.78          16 of 16 completed
aggregate                 120.67 tok/s    decode-window 129.06
mean in-flight              9.28          peak 12, queued 1.94
per-stream                 13.90 tok/s
wall                       26.8 min       for 16 dispatches, 265 turns
prefix cache hit           92.6%
KV pool usage              <10%           zero preemptions
node peak temperature      96.7 C         no trip (only trip point is 104 C)
```

### Per turn

```text
prompt tokens             34,178
  of which fresh           2,522          92.6% served from prefix cache
generated                    733
wall                        6.08 s
```

### Latency and prefill

Idle, single request:

```text
TTFT floor (short prompt)          0.29 s
fresh prefill rate               1,739 tok/s    slope method, 12k -> 48k tokens of
                                                cache-defeating random content
48k-token fully cold prompt       27.2 s end to end
```

At the operating point (16 agent dispatches, 12 seats):

```text
TTFT mean                  24.6 s         710 requests
inter-token latency        266 ms         see caveat below
end-to-end per request     73.3 s
```

**The loaded TTFT is queueing, not prefill.** A typical agent turn presents
~2,155 fresh tokens behind the 94% prefix cache — about **1.2 s** of prefill at
the measured rate. The other ~23 s of the loaded mean is waiting for a seat.
Size the two separately or the number misleads.

**Two measurement traps worth naming:**

- Streaming `time_starttransfer` reads **0.01-0.03 s** on this server — vLLM
  emits the first SSE chunk (the role delta) *before* prefill completes. Anyone
  benchmarking TTFT off the first stream byte is measuring the HTTP layer, not
  the model. Use the first *content* token, or a non-streaming
  `max_tokens=1` request (our 0.29 s).
- The 266 ms "inter-token latency" is really **decode-step cadence**. With MTP
  accepting ~3.9 tokens per forward, tokens arrive in bursts of ~4 every
  266-358 ms rather than one every 70-80 ms. Smooth-reading clients hide this;
  latency histograms do not.

### Speculative decoding (DSpark MTP, k=5)

```text
acceptance                57.5%           126,869 drafts
accepted per forward       3.88 of 5
```

Per-position survival, at the 27k-61k contexts this workload reaches:

```text
position     0       1       2       3       4
survival   86.0%   69.9%   55.4%   43.0%   33.2%
```

Acceptance was stable at 58-61% across every seat count tested, including under
heavy multi-agent churn, and did not degrade at 12 seats.

### Concurrency curve

Pure decode, fixed 800-token outputs, no agent in the loop, at default clocks:

| N | aggregate tok/s | per-stream |
|---:|---:|---:|
| 1 | 40.1 | 40.10 |
| 2 | 59.1 | 29.55 |
| 4 | 86.5 | 21.64 |
| 8 | 122.2 | 15.28 |

vLLM's own `Avg generation throughput` logger, bucketed by `Running: N` across
2,654 samples of real load:

| N | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 15 | 16 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| tok/s | 47.6 | 63.5 | 77.1 | 81.8 | 99.0 | 111.3 | 102.2 | 119.4 | 143.2 | 143.3 |

### Agent-workload cells

Same nine-task pool, sandbox bypassed, one client host. `G` is generated tokens
per dispatch and is **not** controlled — see the metric-identity section for why
`disp/hr` cannot be compared across rows with different `G`.

| seats | slots | wall min | disp/hr | agg | G | in-flight | per-stream | peak | result |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---|
| 6 | 1 | 3.2 | 18.70 | 54.38 | 10,469 | 1.00 | — | 1 | ok |
| 6 | 2 | 8.8 | 13.58 | 62.74 | 16,630 | 1.62 | — | 2 | ok |
| 6 | 4 | 16.7 | 14.41 | 70.94 | 17,722 | 2.20 | 35.55 | 4 | ok |
| 6 | 8 | 22.3 | 21.49 | 108.06 | 18,104 | 5.43 | 20.72 | 6 | ok |
| 6 | 12 | 37.3 | 19.28 | 97.74 | 18,249 | 5.12 | 21.21 | 6 | ok |
| 8 | 8 | 22.4 | 21.43 | 86.25 | 14,490 | 4.57 | 20.49 | 8 | ok |
| 8 | 12 | 31.0 | 23.20 | 112.90 | 17,522 | 6.80 | 17.56 | 8 | ok |
| 8 | 24 | 49.1 | 29.31 | 117.96 | 14,490 | 6.73 | 17.65 | 8 | ok |
| **12** | **16** | **26.8** | **35.78** | **120.67** | 12,142 | **9.28** | **13.90** | **12** | **recipe** |
| 12 | 24 | 41.0 | 35.14 | 116.15 | 11,898 | 8.28 | 15.03 | 12 | ok |
| 16 | 24 | 12.2 | — | — | — | 13.15 | — | 16 | **deadlock** |

Rows at 8 and 12 slots use the full task pool; 16 and 24 slots use an
output-capped variant, which is why their `G` is lower. Compare within a pool, or
use in-flight and per-stream.

### Reproducing

```text
pure-decode sweep    N concurrent POSTs to /v1/chat/completions,
                     max_tokens=800, temperature=0.7, one fixed prompt
agent cells          nine heterogeneous repository tasks reading disjoint file
                     sets, rotated across slots, prompt delivered on stdin
sampling             /metrics at 1 s; every reported mean is over 25+ samples
```

---

## The decode model

Every measurement we took lands on one curve. Fitted by least squares on a
pure-decode sweep (fixed 800-token outputs, no agent in the loop) at N = 1, 2, 4, 8:

```text
T_forward(N) = 82.5 + 22.94 * N   ms          L = 3.9 accepted tokens per forward

  82.5 ms   fixed     weight read, amortised across the batch
  22.94 ms  per-seq   attention + MTP verification, not amortisable

per-stream tok/s  = 3900 / (82.5 + 22.94N)
aggregate tok/s   = N * per-stream            asymptote 170
```

**Trust the rates, not the split.** We later caught this fit assuming L=3.9 on
sweep content whose true L is 2.74 (see the draft-depth section) — the constants
absorbed a ~1.4x error that happens to cancel against the real workload's
longer-context attention cost. The rate predictions below are validated end to
end at k=5 on real content and are what we plan with; the physical
interpretation of the two terms, and any cross-k arithmetic derived from them,
is not sound. It cost us one wrong tuning direction before we caught it.

It then predicted real agent cells it was never fitted on:

| N (mean in-flight) | per-stream measured | model | error |
|---:|---:|---:|---:|
| 4.57 | 20.49 | 20.82 | +1.6% |
| 6.73 | 17.65 | 16.46 | +7.2% |
| 6.80 | 17.56 | 16.36 | +7.3% |
| 8.28 | 15.03 | 14.32 | +5.0% |
| 9.28 | 13.90 | 13.20 | +5.3% |

Consistently 5-7% above the model, never below. Independently, vLLM's own
`Avg generation throughput` logger across 2,654 samples, bucketed by
`Running: N`:

```text
N=1  47.6    N=2  63.5    N=3  77.1    N=4  81.8    N=5  99.0
N=6 111.3    N=7 102.2    N=8 119.4    N=15 143.2   N=16 143.3
```

The model was fitted at ~850-token contexts and predicts cells running at
27k-61k contexts to within 1.5%. **Context length does not move the decode
curve** at these lengths.

**Aggregate throughput is set almost entirely by mean in-flight N.** Everything
else we tried moved single digits or nothing.

---

## The two metrics are one metric

We spent real time comparing "aggregate tok/s" against "dispatches per hour" as
if they were independent readings. They are related by an exact identity:

```text
dispatches/hour = 3600 * agg_tok_s / G          G = generated tokens per dispatch
```

Verified to the last digit on every cell:

| cell | disp/hr | agg tok/s | G | 3600*agg/G |
|---|---:|---:|---:|---:|
| C1 | 18.70 | 54.38 | 10,469 | 18.70 |
| C4 | 14.41 | 70.94 | 17,722 | 14.41 |
| C8 | 21.49 | 108.06 | 18,104 | 21.49 |
| C12 | 19.28 | 97.74 | 18,249 | 19.28 |

They are the same measurement divided by a property of the **task**, not of the
server. G is uncontrolled in an agent workload — one prompt produced 18,914 and
30,484 generated tokens on two runs — so both readings inherit that variance.

This bit us three separate times. A seat change that "improved dispatches/hour by
20%" turned out to be a workload 18% smaller; a "+22% faster client host" was the
same artifact, and normalised the other host was 5% ahead.

**The G-independent readings are mean in-flight and per-stream tok/s.** Use those
to compare configurations, or cap output length so G is controlled.

---

## What actually worked

### `max_num_seqs` 8 -> 12: +8.1%

The one server change that paid. Measured on the same task pool:

```text
 8 seats   in-flight 6.80   per-stream 17.56   decode-window agg 119.34
12 seats   in-flight 9.28   per-stream 13.90   decode-window agg 129.06
```

+8.1% aggregate for -20.8% per-dispatch latency. Which side of that you want
depends on whether your work is throughput-shaped or latency-shaped — see the
trade-off section.

### Bypassing the client sandbox: 6.8-9.0x

Not a server finding, but by far the largest single number we measured. Running
the agent CLI with its filesystem sandbox enabled cost **6.8x solo and 9.0x at
eight concurrent instances** on Windows. The same sandbox on Linux cost 1.26x.

It also serialises: concurrency amplification was 2.28x sandboxed against 1.73x
bypassed. If you are benchmarking an agent CLI on Windows and your numbers look
inexplicably bad, check this before touching anything on the server.

---

## What did not work

### Raising the SM clock: +10.4% clock, +2% throughput

The GPUs had been locked to 2,200 MHz during an earlier thermal investigation.
Unlocking moved them to 2,411 MHz against a 3,003 MHz ceiling:

```text
N=1   40.1 vs 40.6 locked   -1.2%
N=2   59.1 vs 58.6          +0.9%
N=4   86.5 vs 84.9          +1.9%
N=8  122.2 vs 119.4         +2.3%
```

The gain rises with N, which is the tell: the fixed 82.5 ms term is the weight
read and is bandwidth-bound, so clock does not touch it. At N=1 that term is 78%
of the forward pass; at N=8, 31%.

Corroborating: under load the GPU reports **96% SM utilisation at 31 W and 68 C**.
Kernels resident, mostly waiting on memory.

**Decode on this hardware is memory-bandwidth-bound.** Clock is not a lever, and
neither is cooling — see below.

### Thermals: never the constraint

We moved the nodes physically, added external cooling, raised a guard threshold,
and lost one complete run to a false trip. The relevant counters, read late:

```text
SW Power Capping        152,078,202 us   = 152 s
SW Thermal Slowdown          73,731 us   = 0.074 s
HW Thermal Slowdown               0 us
```

Under load: 68 C against the only trip point on any ACPI zone, `critical` at
104 C. One node consistently runs 6-8 C hotter than the other on identical
firmware; it never cost throughput.

We had also claimed "zero thermal throttling" repeatedly. It is 74 ms, not zero —
negligible, but not what we said. And the counter that was actually accumulating,
power capping, is 2,063x larger and we never read it until the end.

### `max_num_seqs` 6 -> 8: exactly nothing

Lifted the cap, eliminated queueing (capacity wait 70.5% -> 8.1%), changed
completion rate by **-0.3%**.

The reasoning that justified it: capacity-wait is 70.5%, therefore seats are the
binding constraint, therefore more seats means more throughput. The first two
clauses are true and the conclusion does not follow. **Capacity-wait measures
queueing, not lost capacity.** With the GPU already non-idle 99.7% there was
nothing for extra seats to reclaim — a queue in front of a saturated device is
the expected shape, not a defect.

It was also premature: mean in-flight was 4.57, below even the old cap of 6.
Raising a cap above demand measures nothing.

### `max_num_seqs` 16: engine deadlock

It worked, and it was faster — mean 143.3 tok/s against 119.4 at N=8, matching
the model's predicted +18.4%, peaking at 203.7. For about four minutes.

```text
18:32:30  gen=203.7  running=16
18:33:40  gen= 52.0  running=16     collapse
18:33:50  gen=  0.0  running=16     dead
18:34:35  EngineCore: No available shared memory broadcast block found in 60 seconds
18:38:33  EngineDeadError               4m43s after the engine actually stopped
```

That message is from the **writer**: the engine could not obtain a block to write
the next scheduler output because the workers had stopped consuming. Both TP
ranks stayed alive and silent — no error on either. A process that is alive,
silent, and not draining its input queue is hung inside a step, not crashed.

At the moment of death: 16 running, 7 waiting, **KV at 7.96%**, 96 scheduled
tokens (16 seats x MTP 5+1). Not KV, not thermal.

We formed two explanations and retracted both:

1. *Untuned FlashInfer shapes forcing JIT compilation during inference.* The logs
   do show `No tuned config covers sparse_mla_sm120_decode_dsv4 ... outside the
   tuning bucket range` and `Triton kernel JIT compilation during inference`.
   Refuted by counting: the healthy 6-seat profile produced **more** such
   warnings (11) than the profile that crashed (6), at comparable batch
   dimensions.
2. *That counting refuted it.* Wrong method — the question was timing, not
   frequency. The timing is what actually rules it out: every TileLang compile
   takes exactly 4 s and the last one finished **three minutes** before the
   stall.

Standing hypothesis, unproven: this build carries an unapplied patch that
addresses draft KV by batch-row position rather than by request identity, so
requests finishing mid-flight can cross-contaminate it, with exposure growing in
concurrency. Inconsistent draft state between TP ranks would produce exactly this
divergence. Speculative-decoding acceptance showed no degradation at 8 or 12
seats (58-61%), so nothing visibly suffered below 16.

**If you run this stack at 16 seats, watch for it.**

### Cutting context: <= 6%

Prefill is **6.3%** of inference time, measured directly from
`vllm:request_prefill_time_seconds` against `vllm:request_inference_time_seconds`.
The prefix cache returns **94.1%** unassisted across 1,498 turns. Nominal
input:output is 50.4:1; after cache hits the ratio that actually costs compute is
**3.0:1**.

Per turn: 36,368 prompt tokens, of which 2,155 are fresh, producing 721 generated
tokens. Combining the time split with the token counts, **one decode token costs
44.5x one prefill token**. There is no version of context reduction worth more
than a few percent on a decode-bound workload behind a 94% cache.

### Draft depth (`MTP_NUM_TOKENS`): the answer depends on your content, and a
### synthetic benchmark will give you the wrong one

We went around this one twice, and both passes are worth recording because the
second refuted our own first correction.

**Acceptance is a property of the content, not the model.** The same k=5
configuration, measured live:

```text
per-position survival        pos0   pos1   pos2   pos3   pos4      L
real coding-agent turns     86.6%  70.9%  56.2%  43.6%  33.2%    3.905
synthetic benchmark prompt  72.4%  47.5%  28.9%  16.0%   9.4%    2.744
```

Real agent content — structured, repetitive, full of tool output — drafts far
better at depth than a temperature-0.7 free-form prompt. Any acceptance number
quoted without naming its content is not comparable to yours.

**Forward time scales with draft depth.** Measured at N=12 by varying only k:
T_forward = 221 / ~200 / 162 ms for k = 5 / 3 / 2. Each draft token is a
sequential draft-model pass plus a wider verify.

**So the trade inverts between workloads.** On the synthetic sweep, k=2 beat
k=5 by ~9% aggregate at N=12 — the deep positions it gives up are nearly
worthless there. Priced for real content (L ratio 3.905/2.575 against the
~59 ms forward-time saving on a ~340 ms real forward), the same change projects
**~20% slower**. The faster community configurations running k=2 are not wrong;
they are measured on content where k=2 is right. Ours is not that content, and
k stays 5.

This also dissolved most of our gap to the fastest published dual-GB10 figure
(195 tok/s at c=16 against our 143.3): extrapolating our own k=2 sweep to N=16
lands at ~176 — the residual is ~10%, not 36%, and the configuration that
produces their number would slow our lane down.

### Prefill chunking (`long_prefill_token_threshold`): premise disproved

Suspected of gating admission — at 1,024 tokens per step a cold 36,368-token
prompt needs 36 steps to enter, and we did observe seats sitting free with
requests queued. Disproved by measurement: at 12 seats the queue was **1.94**,
in-flight **9.28**, and peak reached the full 12. If admission were gated,
requests would pile up while seats sat idle. They do not. The 94% cache hit rate
also means the average turn presents only ~2,155 fresh tokens, about two steps.

### `PIECEWISE` CUDA graph mode: -46%

Short and long medians fell to 14.70 and 13.82 tok/s, -46.19% and -48.34%,
with the KV pool unchanged. Regular (non-breakable) CUDA graphs are correct for
this model on SM12x.

### Six-flag tuning bundle: all four cells regressed

No-spec medians fell 38.60% short and 42.03% long; the speculative arm fell
14.25% and 20.81%. Bisecting one flag at a time, only
`gpu_memory_utilization` 0.80 -> 0.835 survived: +21.67% KV pool for -1.2%
decode.

### Greedy draft sampling: rejected on quality

Higher aggregate decode rate and higher acceptance than probabilistic sampling,
but a tool-calling suite regressed 58/60 -> 56/60 — one case found the right
contact and failed to complete the action chain. Speed at the cost of correctness
is not a trade a coding agent can take.

---

## The trade nobody mentions

Aggregate throughput and per-dispatch latency pull in opposite directions, and
every "+N% throughput" result here is also a "-M% latency" result:

| N | aggregate tok/s | per-stream | generation per dispatch | vs N=8 agg | vs N=8 latency |
|---:|---:|---:|---:|---:|---:|
| 4 | 89.5 | 22.38 | 10.8 min | -24% | -35% |
| **8** | **117.3** | **14.66** | **16.5 min** | — | — |
| 16 | 138.8 | 8.68 | 27.8 min | +18% | **+69%** |
| 32 | 152.8 | 4.78 | 50.6 min | +30% | **+207%** |

Every doubling buys less and costs more: 8->16 is +18% aggregate for +69%
latency; 16->32 is +10% for +82%; 32->64 is +5% for +90%.

Which side you want depends on the shape of the work. A serial chain — survey,
then plan, then execute, each waiting on the last — is set by per-dispatch
latency, not aggregate. Background load on such a chain:

```text
alone (N=1)         19.6 min
under N=8 load      49.4 min     2.5x
under N=16 load     83.5 min     4.3x
under N=32 load    151.7 min     7.7x
```

**N should track the parallelism the work actually has, not be maximised.**
Concurrency still wins outright for genuinely independent work — 8 concurrent is
3.15x faster than serial — but past the point where your work is actually
parallel, you are only lengthening everyone's critical path.

---

## Operational notes

**Metrics stay live and frozen when the engine dies.** After the deadlock, the
API server kept answering `/metrics` with `Running: 16, KV 7.9%` for 4m43s. Every
gauge held its last value. The only tell is a monotonic counter that stops
advancing: check `vllm:generation_tokens_total` across samples while
`num_requests_running > 0`.

**A single instantaneous metrics read is not usable.** `num_requests_running` is
noisy enough that every conclusion here rests on 25-40 samples.

**The agent CLI leaks processes.** It finished tasks, wrote deliverables, printed
closing summaries, and did not exit — we found survivors five hours old. A
benchmark harness that waits on process exit will never return; ours reported a
cell as unfinished twenty minutes after it had finished. Reclaim on evidence of
completion (deliverable written, log idle) rather than on process state.

**Failed runs still report throughput.** The cell that deadlocked reported
118.08 dispatches/hour — 24 dispatches failing quickly. Check the success flag
before reading any headline number.

---

## Where the remaining headroom is

Not on the server. At the settled operating point the model puts us at 129 tok/s
against a 170 tok/s asymptote, and the gap is **mean in-flight**, which is set by
how many requests the client can keep outstanding. One client host reached 9.28
of 12 seats. A second client host is worth more than any remaining server knob,
and costs nothing on the inference side.

The one server-side avenue we did not exhaust: the FlashInfer autotuner asks
explicitly for a wider tuning pass (`expand tuning_buckets / max_num_tokens`) and
the JIT monitor asks for more warmup shapes. Given that decode here is
bandwidth-bound, we do not expect much, but we did not test it.

---

## Settled configuration

```text
max_model_len                    1048576
max_num_seqs                          12
max_num_batched_tokens              8192
long_prefill_token_threshold        1024
gpu_memory_utilization             0.835
kv_cache_dtype                nvfp4_ds_mla
block_size                           256
speculative: dspark, num_spec_tokens   5
enable_prefix_caching               true
enable_chunked_prefill              true
async_scheduling                    true
cudagraph_mode        FULL_AND_PIECEWISE   (breakable graphs off)
moe_backend             flashinfer_b12x
tensor_parallel_size                   2   (forced by model size)
GPU clocks                       default   (no lgc lock)
```

Delivered on a real agent workload: **35.78 dispatches/hour, 120.67 aggregate
tok/s, in-flight 9.28, per-stream 13.90 tok/s**, 16 of 16 dispatches completed,
no thermal trip, no KV preemption, prefix cache hit 92.6%.

---

## Caveats on all of the above

Most cells are **single observations**. Where a result is small — the clock A/B,
the 6->8 seat null — a repeat would be worth having, and we did not run one.
Output length is uncontrolled in this workload, which is exactly why we lean on
in-flight and per-stream rather than on aggregates.

The deadlock at 16 seats is **one occurrence**. We did not reproduce it, and the
mechanism is not established.

Numbers are from one hardware pair, one model revision, one runtime build. The
shape of the decode model should transfer; the constants will not.
