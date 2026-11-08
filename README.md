# Software-Defined Analog Engine (SDAE)

SDAE is an experimental C# library and runtime prototype for expressing conditional selection and bit-state reductions with fixed-width integer operations. Its central techniques are two's-complement masks, Boolean state differences, and SWAR (SIMD Within A Register) population count.

The project is **not Unity-specific**. Unity was used as a convenient test harness for exercising the runtime component and capturing benchmark output. The reusable bit-operation ideas can also be evaluated in other managed applications, subject to their runtime and memory APIs.

> **Status:** Research/prototype code. Benchmark figures below are observations from the included Unity run, not portable performance guarantees or a claim that SDAE is always faster than compiler-optimized code.

## Contents

- [Overview](#overview)
- [Benchmark snapshot](#benchmark-snapshot)
- [How the benchmark works](#how-the-benchmark-works)
- [Algorithm and mathematical details](#algorithm-and-mathematical-details)
- [Runtime pipeline](#runtime-pipeline)
- [Reusable operations](#reusable-operations)
- [Setup and use](#setup-and-use)
- [Applications](#applications)
- [Engineering challenges and limitations](#engineering-challenges-and-limitations)
- [Project files](#project-files)

## Overview

The source contains two related parts:

1. **A microbenchmark** compares a scalar control-flow implementation, a bit-manipulation implementation, and an SDAE implementation over the same `ulong` input sequence. All three calculate the same transformed value and its population count.
2. **A runtime pipeline component** keeps per-channel unmanaged buffers, compares current and previous 64-bit states, packs a validity sentinel and payload, and emits an active-low trigger when the validated word is zero. It can optionally capture audio from the default Windows WASAPI endpoint.

These are not the same workload: the benchmark measures a pure array transformation and reduction, while the runtime component exercises stateful frame processing, buffers, and optional audio capture.

## Benchmark snapshot

The following values are transcribed from the included Unity Console screenshots. That run used **1,000,000 64-bit records**, **5 warmups**, and **30 measurement runs per implementation**.

| Implementation | Mean time | Standard deviation | Reported throughput | Speedup vs. scalar |
|---|---:|---:|---:|---:|
| Traditional control flow | 33.6805 ms | 0.7150 ms | 29.69 million records/s | 1.00x |
| Bit manipulation / SWAR | 7.9720 ms | 0.0958 ms | 125.44 million records/s | 4.23x |
| SDAE active-low MUX / SWAR | 5.2039 ms | 0.1975 ms | 192.16 million records/s | 6.47x |

In this captured run, SDAE was **1.53x** faster than the bit-manipulation baseline, corresponding to **34.72% less mean execution time**. The report also logs a matching checksum (`0x0000000001EC14A9`) and bit-for-bit output parity across all three implementations.

![Unity benchmark setup, parity check, and traditional baseline results](Test%20part%201.png)

![Unity benchmark results for the SWAR and SDAE implementations](Test%20part%202.png)

These are machine- and runtime-specific results. The benchmark labels its throughput `MOps/sec`, but the calculation divides the number of **records** by elapsed time; the table therefore calls it million records/s. The high-entropy input is produced by a deterministic xorshift sequence. The benchmark does not collect hardware performance counters, so the stated branch-misprediction risk is a workload rationale, not a measured misprediction rate. The screenshots do not identify the CPU, Unity version, build configuration, or operating system; reproduce and report those details before making comparisons.

## How the benchmark works

For each input word \(v\), all three implementations compute a transformed word \(t\), add its number of set bits to a total, and store the transformed output. Their shared mapping is:

\[
q(v) =
\begin{cases}
2v \pmod {2^{64}}, & v \mathbin{\&} 1 = 1 \\
v+50 \pmod {2^{64}}, & v \mathbin{\&} 1 = 0
\end{cases}
\]

\[
r(v) =
\begin{cases}
\mathtt{0xAA}, & v \mathbin{\&} 3 = 0 \\
\mathtt{0x55}, & v \mathbin{\&} 3 = 1 \\
\mathtt{0xFF}, & v \mathbin{\&} 3 = 2 \\
\mathtt{0x0F}, & v \mathbin{\&} 3 = 3
\end{cases}
\qquad
t = q(v) \oplus r(v)
\]

The reduction is the sum of population counts:

\[
S = \sum_{i=0}^{N-1} \operatorname{popcount}(t_i)
\]

The traditional version uses `if`/`else`, `switch`, and a scalar bit loop. The comparison version uses a conditional expression, mask selection, and SWAR population count. The SDAE benchmark version builds masks for the two-way and four-way selections, then applies the same SWAR reduction. The implementation uses 64-bit scalar registers; it is **not** a SIMD-vectorized loop.

Benchmark procedure in `SDAEngineBenchmark.cs`:

1. Generate the same xorshift input for all implementations.
2. Execute all three once and verify both checksums and every output element.
3. Warm up each implementation, then collect the configured number of timed samples.
4. Report mean, population standard deviation, minimum, maximum, computed throughput, and measured managed allocations.

The code invokes forced garbage collection during warmup, but the per-run timing and allocation figures should still be independently reproduced in the intended Unity/runtime build before being treated as an allocation or timing guarantee.

## Algorithm and mathematical details

### Two's-complement mask selection

For a Boolean bit \(b \in \{0,1\}\), unsigned modular arithmetic yields:

\[
M_b = 0 - b \pmod {2^{64}} =
\begin{cases}
\mathtt{0x0000000000000000}, & b=0 \\
\mathtt{0xFFFFFFFFFFFFFFFF}, & b=1
\end{cases}
\]

The complementary mask is \(\neg M_b\). A word-level select can then be written:

\[
\operatorname{select}(b,A,B) = (A \mathbin{\&} M_b) \mathbin{|} (B \mathbin{\&} \neg M_b)
\]

The benchmark uses this form to combine both candidate values without expressing an explicit branch in the C# source. Whether the JIT emits branch-free machine code is runtime-, platform-, and context-dependent; inspect generated code or measure on the target before relying on that property.

### Four-way selection

The benchmark derives four equality bits \(e_k = [\,v \mathbin{\&} 3 = k\,]\) for \(k \in \{0,1,2,3\}\), expands each bit to a full-width mask \(M_{e_k}\), and combines the constant choices:

\[
r(v) = (\mathtt{0xAA} \mathbin{\&} M_{e_0})
       |(\mathtt{0x55} \mathbin{\&} M_{e_1})
       |(\mathtt{0xFF} \mathbin{\&} M_{e_2})
       |(\mathtt{0x0F} \mathbin{\&} M_{e_3})
\]

Exactly one equality is true for the low two bits. This is a fixed-width word operation; processing an array of \(N\) values remains \(O(N)\).

### SWAR population count

SWAR computes the number of set bits in one 64-bit word by progressively accumulating counts in 2-, 4-, and 8-bit groups, followed by a multiply-and-shift horizontal sum. For input word \(x\):

\[
x_1 = x - ((x \gg 1) \mathbin{\&} \mathtt{0x5555555555555555})
\]

\[
x_2 = (x_1 \mathbin{\&} \mathtt{0x3333333333333333})
     + ((x_1 \gg 2) \mathbin{\&} \mathtt{0x3333333333333333})
\]

\[
x_3 = (x_2 + (x_2 \gg 4)) \mathbin{\&} \mathtt{0x0F0F0F0F0F0F0F0F}
\]

\[
\operatorname{popcount}(x) =
\left(x_3 \times \mathtt{0x0101010101010101}\right) \gg 56
\]

The low byte of each intermediate holds the relevant partial count; the final multiplication accumulates byte counts so the high byte is the total, in the range 0–64. `Count` implements these operations directly. Its work per word is fixed; `CountBatch` still visits every element, so it is \(O(N)\), not \(O(1)\).

### Boolean frame differences

The runtime pipeline retains the previous payload \(P_{t-1}\) and reads the current payload \(P_t\):

\[
D_t = P_t \oplus P_{t-1}
\qquad
Z_t = \neg(P_t \mathbin{|} P_{t-1})
\]

At each bit, \(D_t\) marks a transition. \(Z_t\) marks a bit that is zero in both frames. These are bitwise state descriptors; they do not, by themselves, construct or evaluate a general Boolean Jacobian.

The code's so-called reversibility sentinel is a heuristic:

\[
R_t = [P_t \ne 0] \lor [Z_t \ne \mathtt{0xFFFFFFFFFFFFFFFF}]
\]

Since \(Z_t\) is all ones exactly when both payloads are all zero, this flag is zero only for consecutive all-zero payloads. It is not a computed determinant \(\det(J_F)\), nor a proof that a transformation is invertible.

The packed word is:

\[
W_t = (R_t \ll 63)
     \mathbin{|} (P_t \mathbin{\&} \mathtt{0x7FFFFFFFFFFFFFFE})
     \mathbin{|} (D_t \mathbin{\&} 1)
\]

The top bit is the validity sentinel, payload bits 1–62 are retained, and bit 0 carries the least-significant transition bit. An expanded sentinel mask \(M_R = 0-R_t\) gates the packed word:

\[
V_t = W_t \mathbin{\&} M_R
\qquad
\text{drop} = [V_t = 0]
\]

The implementation then updates channel bookkeeping and invokes the active-low event when a drop is detected.

## Runtime pipeline

`SDAEngineMonoBehaviour` is a Unity `MonoBehaviour` that:

1. Allocates one 64-byte-aligned native block per configured channel and a separate previous-state array.
2. Initializes each channel's bytes 32–47 as a mask area and bytes 48–51 with an initial flag.
3. On each `Update`, evaluates a frame using an optional caller-provided unmanaged pointer, or attempts to read from the default audio capture endpoint when WASAPI is active.
4. Calculates the XOR/NOR descriptors, applies the sentinel and packing rules described above, and advances the active channel when the validated result is zero.
5. Frees native buffers and stops audio capture in `OnDestroy`.

The current audio setup is Windows-specific and requests 44.1 kHz, stereo, 16-bit PCM. WASAPI capture is opt-in through the serialized `enableWasapiDma` setting, which defaults to `true` in the source. Audio capture is not a general-purpose DMA interface: it uses the Windows audio capture API and a managed COM interop layer.

## Reusable operations

The implementation exposes these static helpers on `SDAEngineMonoBehaviour`:

| Method | Behavior |
|---|---|
| `Select(bool, ulong, ulong)` | Selects one of two words with complementary masks. |
| `Dispatch(bool, Action, Action)` | Invokes one of two callbacks using ordinary C# conditional control flow. It is not branchless and callbacks/closures may allocate. |
| `Count(ulong)` | Counts set bits in one word using SWAR. |
| `CountBatch(ReadOnlySpan<ulong>)` | Sums the population count over a span. |
| `Any(ReadOnlySpan<ulong>, ulong)` | Returns whether any word intersects the supplied mask. There is no `All` helper in the current source. |
| `TransformInPlace(Span<ulong>, ulong, ulong)` | XOR-mutates words whose value intersects the target mask, using mask selection. |

Example:

```csharp
using System;
using SDAE.Unity;

ulong flags = 0b1011;
ulong bitCount = SDAEngineMonoBehaviour.Count(flags);

ulong[] states = { 0x01, 0x10, 0x00 };
bool hasTarget = SDAEngineMonoBehaviour.Any(states, matchMask: 0x10);
ulong totalBits = SDAEngineMonoBehaviour.CountBatch(states);

SDAEngineMonoBehaviour.TransformInPlace(
    states,
    targetMask: 0x01,
    xorMutation: 0xAA);
```

`Select`, `Count`, and the per-word selection in `TransformInPlace` avoid explicit branching in their source expressions. `Dispatch` necessarily chooses between callback invocations, and span-based batch methods are linear scans. Do not infer constant-time behavior for an entire collection from a fixed amount of work on one word.

## Setup and use

### Unity

1. Add `SDAEngineUnified.cs` and `SDAEngineBenchmark.cs` to a Unity project.
2. Ensure unsafe code is enabled for the relevant assembly and that the project has the Unity Collections package/reference required by `UnsafeUtility`.
3. Add `SDAEngineMonoBehaviour` to a GameObject to run the frame pipeline. Configure the channel count and audio capture option in the Inspector.
4. Add `SDAEEngineBenchmarkTester` to a GameObject to run the benchmark on start, or invoke **Run Scientific Benchmark Now** from its component context menu.
5. Run on the target hardware and record CPU, OS, Unity version, scripting backend, build mode, and power/runtime settings with the results.

The runtime component also attempts to set the current process to real-time priority and pin the calling thread to the configured CPU core during initialization. These are privileged, platform-specific controls and can interfere with system responsiveness; use cautiously, and do not treat them as required for the general bit-operation helpers.

### Outside Unity

The integer techniques (mask selection, SWAR count, and span transforms) can be adapted to other C# environments. The current source file is coupled to Unity types, `UnsafeUtility`, and Unity lifecycle methods, so extracting the helpers into a standalone library requires replacing those dependencies and separately validating generated code and performance on the target runtime.

## Applications

Potential areas to evaluate include:

- **Bitsets and rule masks:** count active flags or apply conditional XOR updates across packed state.
- **Deterministic simulation and event processing:** represent compact state transitions as fixed-width words and compare frame-to-frame changes.
- **Protocol and telemetry parsing:** evaluate flags and summarize packed fields in network or device records.
- **Signal-processing prototypes:** encode threshold/state outcomes as bit masks and measure bounded per-word transformations.
- **Game and interactive systems:** use Unity as one host, while keeping reusable integer operations conceptually independent of a game engine.
- **Performance education and compiler experiments:** compare source-level branches, mask expressions, and alternative population-count strategies under controlled workloads.

These are possible application domains, not validated production deployments. Workload characteristics, compiler code generation, memory bandwidth, CPU architecture, and data layout determine whether these techniques are beneficial.

## Engineering challenges and limitations

- **Behavioral parity:** the benchmark compares every output element and checksum before timing, preventing a faster but incorrect transformation from appearing successful.
- **Reducing control-flow sensitivity:** mask-based selection and SWAR avoid the scalar bit-at-a-time reduction and explicit `switch` in the SDAE benchmark path. The JIT may still emit branches for comparisons or other expressions.
- **Separating fixed word cost from data-set cost:** individual 64-bit operations take a bounded sequence of instructions, but processing an arbitrary-length array necessarily scales with its length.
- **Keeping the benchmark interpretable:** all three paths share the same transform and output semantics, and warmup and repeated measurements are used. Results still depend on runtime/JIT, hardware, process scheduling, and measurement overhead.
- **Avoiding overclaiming mathematical machinery:** XOR and NOR are useful Boolean state descriptors; the runtime's current sentinel is a zero-state heuristic, not a Jacobian-matrix determinant.
- **Handling native/platform integration:** aligned unmanaged allocation, CPU affinity, real-time process priority, COM interop, and WASAPI make the runtime component platform- and privilege-sensitive. They should be guarded or adapted before porting.
- **Scope of the benchmark:** it is a single-threaded array microbenchmark. It does not establish end-to-end application throughput, SIMD execution, hardware-level determinism, real-time guarantees, or superiority across processors.

## Project files

- [`SDAEngineUnified.cs`](SDAEngineUnified.cs) — Unity runtime component, frame pipeline, native buffers, audio capture interop, and reusable bit-operation helpers.
- [`SDAEngineBenchmark.cs`](SDAEngineBenchmark.cs) — dataset generation, three equivalent benchmark paths, parity checks, timing, statistics, and report output.
- [`sdae_technical_documentation.md`](sdae_technical_documentation.md) — earlier technical notes; consult the current source and this README where its claims differ from the implementation.
- [`Test part 1.png`](Test%20part%201.png) and [`Test part 2.png`](Test%20part%202.png) — benchmark run screenshots.
