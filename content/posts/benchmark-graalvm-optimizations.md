---
date: "2026-06-21T18:00:00+01:00"
lastmod: "2026-06-21T18:00:00+01:00"
draft: false
categories: ["GraalVM", "Java", "Native Image", "Performance"]
tags: ["native-image", "graalvm", "jvm", "spring-boot", "spring-ai", "build-time", "startup-time", "memory-usage", "benchmarking", "mcp-server", "java-21"]
title: "Benchmark: GraalVM Native Image Optimizations"
summary: "Benchmarking GraalVM native-image optimizations for build time, startup time, binary size, and idle memory. Comparing dynamic vs static builds and various optimization levels."
---

Model Context Protocol (MCP) servers exhibit an unusual runtime profile. A server is typically spawned afresh for each editor session and then remains largely idle, processing JSON-RPC messages that arrive in irregular bursts. Under these conditions, two characteristics dominate the user-perceived cost of the process: the latency of startup and the resident memory retained while the server is otherwise inactive. Sustained throughput, the metric most benchmarks optimize for, is comparatively unimportant.

Ahead-of-time (AOT) compilation via GraalVM native-image is the canonical remedy for both. However, "native-image" denotes a family of configurations rather than a single target. A build is parameterized by a linking strategy (`dynamic`, `static-musl`, or `static-nolibc`) and an optimization level (`-O0` through `-O3`, together with `-Os` for binary size and `-Ob` for build throughput). Each combination resolves the trade-off between build time, binary size, startup latency, and idle memory differently. This article reports an empirical evaluation of the full configuration space, conducted on a Spring Boot MCP server, with the JVM as a baseline.

{{< center_image src="/images/posts/benchmark-graalvm-optimizations/linking-options.png" alt="GraalVM linking options, from Alina Yurenko's Devoxx talk “Deep Dive: GraalVM in Practice”" width="100%" >}}

{{< admonition info "Experimental setup" true >}}
- **Hardware:** `docker-node` — 32 vCPU, 15,831 MB RAM
- **Application:** MCP server on Spring Boot 4.0 (Spring AI 2.0.0-M4)
- **Toolchain:** native images built with GraalVM 25; JVM baseline on Java 21 (HotSpot); built with Maven
- **Design:** 6 optimization levels × 3 linking strategies, plus a JVM baseline → **19 configurations**
- **Replication:** 10 iterations per configuration → **190 measured runs**
- **Instrumentation:** CPU utilization and resident set size (RSS) sampled from operating-system counters at a fixed interval over a 60 s observation window per run
- **Build outcomes:** all configurations compiled and started successfully

Ten iterations per configuration is the basis for treating the results as evidence rather than anecdote. The sample is sufficient to estimate medians and the 95th percentile with stable rank ordering, such that the comparative conclusions are robust to re-execution.
{{< /admonition >}}

## Key findings

The principal measurements are collected in the table below.

| Metric                      | JVM                      | Best native                   | Production (`dynamic / -O3`) |
| --------------------------- | ------------------------ | ----------------------------- | ----------------------------- |
| Build time (median)         | **5.75 s**               | 60.65 s (`dynamic / -Ob`)     | 110.95 s                      |
| Startup time (median)       | 1,809 ms                 | **33 ms** (several combos)    | 33 ms                         |
| Artifact size               | 49.3 MB _(jar only)_     | **65.4 MB** (`dynamic / -Os`) | 142.1 MB                      |
| Idle memory (RSS, median)   | 252.4 MB                 | **79.0 MB** (`dynamic / -Os`) | 106.7 MB                      |
| Peak CPU at startup         | **790 %** _(multi-core)_ | ~80 % _(single core)_         | ~80 %                         |
| Time to idle (CPU < 1 %)    | 2,251 ms                 | **101 ms**                    | 101 ms                        |

The JVM is superior on exactly one dimension — build time — where its advantage is substantial. Native image dominates the remaining dimensions by margins large enough that the ordering is unambiguous without formal significance testing: startup is reduced by approximately 55× (subject to the measurement caveat below), and idle memory by approximately 3×.

{{< admonition warning "A caveat on the reported startup metric" >}}
Spring reports a 33 ms startup time for the native build, and this figure is tempting to cite in isolation. Doing so is misleading. The operating system continues to register CPU activity for a further ~68 ms after Spring emits its `Started` log line — lazy initializers, residual classpath scanning, and reflection-cache population. The defensible cold-start figure is therefore ~101 ms rather than 33 ms.

The discrepancy is +24 % on the JVM but **+207 % on native**, because native's small absolute baseline is dominated by a fixed post-`Started` tail. Measured to genuine idle, the JVM-to-native speedup is approximately **22×**, not the headline 55×. The effect remains large; the methodological point is that startup must be measured to quiescence, not to the framework's self-report.
{{< /admonition >}}

## JVM versus native image

The first question is whether the runtime gains justify the considerably longer build. A six-second JVM build becoming a two-minute native build is not self-evidently a good trade.

### Per-metric comparison

{{< center_image src="/images/posts/benchmark-graalvm-optimizations/00_jvm_vs_native.png" alt="JVM median vs the best native combo for each of build time, startup, size, and idle memory" width="100%" >}}

Each panel compares the JVM against the _best_ native configuration for that single metric; consequently the native bar represents a different configuration in each panel. The optimal configurations are not coincident — `-Ob` minimizes build time, `-Os` minimizes both binary size and idle memory, and `-O3` minimizes startup. Build time is the JVM's sole advantage, attributable to the absence of AOT work: Maven performs only compilation and packaging.

The jar's apparent size advantage is an artifact of measurement. The jar excludes the ~300 MB runtime it requires to execute, whereas the native binary is self-contained; the comparison is therefore not like-for-like.

### Startup dynamics

{{< center_image src="/images/posts/benchmark-graalvm-optimizations/00b_jvm_vs_native_burst.png" alt="CPU and memory over the first five seconds: native spikes once and flatlines; the JVM ramps for two full seconds" width="100%" >}}

The first five seconds of execution are the most informative window. The native binary saturates a single core for approximately 100 ms and then quiesces, with resident memory flat at ~86 MB throughout. The JVM, by contrast, remains active for roughly two seconds: CPU occupies the 30–80 % band — peaking at 790 % as Spring loads classes across roughly eight cores concurrently — while memory rises from 50 MB to beyond 200 MB as the bean graph and reflection caches are constructed at runtime.

The JVM's competitive wall-clock startup is thus a product of parallelism rather than economy. It is fast because it expends substantially more CPU work, distributed across cores, not because it performs less work.

### Steady-state behaviour

{{< center_image src="/images/posts/benchmark-graalvm-optimizations/00c_jvm_vs_native_idle.png" alt="Steady-state from 5 to 60 seconds: both idle at 0% CPU, but the JVM holds ~252 MB versus native's ~86 MB" width="100%" >}}

In the idle window, both processes register 0 % CPU, the server being blocked on `read()` awaiting input that does not arrive. Resident memory diverges sharply, however: the JVM stabilizes at ~252 MB whereas native holds ~86 MB. This **~166 MB difference constitutes fixed overhead** retained for the lifetime of every session. For a process that a development environment launches and abandons repeatedly, the cumulative cost is considerable.

### Amortization of build cost

{{< center_image src="/images/posts/benchmark-graalvm-optimizations/00d_break_even.png" alt="Cumulative time vs number of launches: native starts higher but the lines cross around 31 launches" width="100%" >}}

Plotting cumulative time (`build + N × startup`) against the number of launches makes the trade-off explicit. The JVM line begins low at 5.75 s but ascends steeply, at nearly two seconds per launch; the best native line begins at the build cost but is nearly flat thereafter. The two intersect at approximately **31 launches**.

The decision therefore reduces to launch frequency:

- A continuous-integration job that starts the server a few times per release favours the JVM.
- An editor integration that relaunches each session exceeds 31 launches within a single working day, favouring native.

For the production configuration (`dynamic / -O3`, with its heavier 111 s build), the break-even point shifts to approximately 59 launches, which remains attainable within a day under the target workload.

### Decomposition of cold-start latency

{{< center_image src="/images/posts/benchmark-graalvm-optimizations/00e_time_to_interactive.png" alt="Stacked bars splitting startup into Spring's reported time and the hidden post-Started CPU tail" width="100%" >}}

This figure formalizes the earlier caveat. Each bar decomposes cold-start latency into Spring's self-reported component (lower segment) and the CPU-busy tail occurring after the `Started` event (upper segment). On the JVM the tail adds ~442 ms to 1,809 ms, a 24 % underestimate; on native it adds ~68 ms to 33 ms, a 207 % underestimate. The complete bar is the appropriate quantity to report.

The evidence above establishes that, for a frequently relaunched server, native image justifies its build cost. The remaining question concerns the choice among native configurations.

## Optimization within native image

The subsequent analysis excludes the JVM baseline and considers only the 18 native configurations. As a preliminary observation, all 180 native runs compiled and started without failure across every optimization level and linking strategy — a necessary, if unremarkable, precondition for the comparisons that follow.

### Build cost is sensitive to optimization only for dynamic linking

{{< center_image src="/images/posts/benchmark-graalvm-optimizations/02_heatmap_build_time.png" alt="Build-time heatmap: dynamic varies wildly by opt level, both static rows are uniformly ~85 s" width="100%" >}}

A clear interaction emerges. The static linking strategies are nearly invariant with respect to optimization level: the static toolchain dominates the cost at a uniform ~85 s. The `dynamic` strategy, conversely, ranges from 60.65 s at `-Ob` to 110.95 s at `-O3`, the latter being the largest single value in the grid.

Build-time variability follows the same partition. The static configurations are highly reproducible, with a coefficient of variation below 0.5 %, whereas the extreme dynamic levels (`-O0`, `-O3`) exhibit 1.5–3 %. For regression alerting in CI, the static configurations are accordingly the more dependable. (The first iteration of each configuration is not an outlier: a warmup pass populates the Maven cache prior to timed execution, yielding flat per-iteration measurements.)

### Binary size exhibits a 2× range under dynamic linking

{{< center_image src="/images/posts/benchmark-graalvm-optimizations/02_heatmap_binary_size_bytes.png" alt="Binary-size heatmap: static rows flat, dynamic ranges from 65 MB to 142 MB" width="100%" >}}

The pattern mirrors build time. Static linking produces a fixed size irrespective of optimization level — 95.3 MB for nolibc, 91.6 MB for musl — while dynamic linking spans 65.4 MB at `-Os` to 142.1 MB at `-O3`, a **2.2× range**. Where artifact size is constrained, as in container image distribution or serverless cold-start, the relevant control resides in the dynamic builds.

### Startup latency is largely uniform, with two exceptions

{{< center_image src="/images/posts/benchmark-graalvm-optimizations/02_heatmap_startup_time.png" alt="Startup-time heatmap: most configs 33-35 ms, dynamic/-Os slowest at 56 ms" width="100%" >}}

Startup latency is consistent across most configurations, clustering within a narrow 33–35 ms band. Two configurations deviate: `dynamic / -Os` is slowest at 56 ms, indicating that size optimization carries a runtime penalty, while `dynamic / -O3` lies at the fast extreme, the corresponding benefit of aggressive optimization.

Several secondary observations, omitted as separate figures, support this picture. The startup _profile_ is identical across native configurations: a single CPU spike near 50 ms, peaking at ~80 % — a single core — followed by quiescence. The contrast with the JVM's 790 % peak captures the architectural difference succinctly: the JVM parallelizes startup across cores; native does not. Integrating the spike yields total CPU work, by which measure `-Os` consumes roughly twice the CPU-seconds of `-O3`, since prolonged startup keeps a core occupied for longer; the JVM consumes approximately 20× the CPU work of native to reach the same state. Finally, measured to genuine idle rather than to Spring's `Started` event, native converges at ~101 ms against a self-reported 33–56 ms — the fixed ~68 ms tail introduced earlier, now observable in every configuration.

### Resident memory is allocated at startup and held constant

{{< center_image src="/images/posts/benchmark-graalvm-optimizations/14_rss_percentiles.png" alt="RSS at launch, median, and P95 are within ~1 MB for every native combo" width="100%" >}}

This is arguably the most pronounced native-versus-JVM distinction in the study. For every configuration, resident memory at launch, at the median, and at the 95th percentile lies within ~1 MB — native attains its final footprint at the first sample and does not subsequently vary. The JVM, by comparison, ramps from 50 MB to 252 MB over its first few seconds.

The plateau's magnitude depends on the linking strategy: dynamic spans 79–107 MB by optimization level, static-musl settles at 86 MB, and static-nolibc at 94 MB. The footprint is genuinely stable; the memory slope over a full idle minute is indistinguishable from zero for every configuration. **No native configuration leaks memory at idle**, consistent with the expected behaviour of AOT-compiled code: absent background JIT compilation, class unloading, and significant garbage-collection activity, memory is allocated at startup and retained until exit.

### Trade-off frontiers

No configuration is optimal on all dimensions, so configuration selection is properly framed as a bi-objective problem.

{{< center_image src="/images/posts/benchmark-graalvm-optimizations/04a_pareto_startup_vs_memory.png" alt="Startup vs idle memory scatter; the frontier hugs the bottom-left" width="100%" >}}

The first frontier considers startup against idle memory, with the lower-left region preferred. `dynamic / -Os` occupies the memory-minimal extreme at 79 MB but incurs 56 ms startup; the `static-musl` configurations lie marginally off it at ~86 MB and ~33 ms, trading a small memory increment for substantially faster startup. The configuration offering the best balance is **`static-musl / -O1`**: a negligible memory penalty relative to the dynamic equivalent, faster startup, and a more reproducible build.

{{< center_image src="/images/posts/benchmark-graalvm-optimizations/04b_pareto_buildtime_vs_size.png" alt="Build time vs binary size scatter; the frontier is entirely dynamic builds" width="100%" >}}

The second frontier, build time against binary size, inverts the conclusion: it is populated **entirely by dynamic configurations**. `-Os` (65 MB, 63 s) and `-Ob` (81 MB, 61 s) define the efficient edge, while no static configuration is Pareto-optimal, each paying the fixed ~85 s toll for a 91–95 MB binary. `dynamic / -O3` is dominated on both axes simultaneously.

That the two frontiers disagree is not incidental; the disagreement constitutes the selection problem. The dynamic configurations favourable on build time and size concede performance at runtime, and conversely. A final consideration for redeployment-intensive pipelines: across 1 to 100 redeployments, total cost is dominated by build time rather than startup — 100 launches at 33 ms total 3.3 s, whereas a single additional build at +50 s costs fifteenfold more. Under frequent redeployment, build time is therefore the appropriate optimization target, the startup differences being below the noise floor at that scale.

## Recommendations

| Objective                                        | Configuration                              | Rationale                                                                  |
| ------------------------------------------------ | ------------------------------------------ | ------------------------------------------------------------------------- |
| **Best all-round** (size, speed, reproducibility)| `static-musl / -O1`                        | ~86 MB, 33 ms, low build variance; well placed on both frontiers.         |
| **Minimal build time**                           | `dynamic / -Ob`                            | 60.6 s build, the lowest; 40 ms startup.                                  |
| **Minimal binary size**                          | `dynamic / -Os`                            | 65.4 MB, at the cost of the slowest startup (56 ms) and most startup CPU. |
| **Minimal startup**                              | `dynamic / -O3` or any `static-*` `-O1`+   | All ~33 ms; `static-musl / -O1` achieves this at lower memory.            |
| **Maximal build reproducibility**                | `static-nolibc / -O3`                      | Lowest build-time coefficient of variation.                               |
| **Current production**                           | `dynamic / -O3`                            | Minimal startup, at a cost in build time (111 s) and size (142 MB).       |

In the absence of a specific constraint, `static-musl / -O1` is the recommended default, with `dynamic / -O3` reserved for cases in which the marginal startup improvement is material. The server measured here is precisely such a case — relaunched once per editor session and otherwise idle, it is dominated by startup latency — and is consequently deployed on `dynamic / -O3`, accepting the longer build and larger binary in exchange for the fastest available start. Two configurations warrant explicit caution: `dynamic / -O3` where peak startup is not required, as it pairs the worst build time and size for little benefit, and `dynamic / -Os` where startup is at all relevant, as it combines the slowest startup with the greatest startup CPU consumption.

{{< admonition warning "Threats to validity" >}}
1. **Scope.** The study measures cold-start and idle cost; it does not measure throughput or tail latency under load, which require a separate workload generator.
2. **Idle assumption.** Every memory and CPU figure characterizes a server awaiting input, not one servicing requests.
3. **Environment.** The measurements were taken under Docker on Windows. A native Linux host (e.g. Hetzner or EC2) should preserve the relative ordering, but absolute values — JVM startup in particular — will differ.
4. **Single execution.** No configuration failed, so the data do not differentiate configurations on stability; this is reassuring but not conclusive.
5. **Interpretation of the JVM CPU peak.** The 790 % peak indicates that the JVM parallelizes startup across roughly eight cores. Combined with its longer active window, this amounts to approximately 20× the total CPU-seconds native expends to reach the same state — its competitive wall-clock startup is fast, but not inexpensive.
6. **JDK version of the baseline.** The JVM baseline runs on Java 21 rather than Java 25, a build constraint rather than a design choice: the application targets Spring Boot 4.0, which depended on a Maven plugin version that did not support Java 25, pinning the toolchain to Java 21. Spring Boot 4.1, released subsequently, adopts a newer Maven plugin version with Java 25 support, enabling a matched-JDK baseline in future runs. Given the magnitude of the differences reported here, a JDK minor-version change to the baseline would not alter the conclusions.
{{< /admonition >}}

## Future work

A follow-up benchmark is warranted on two counts. First, the JDK asymmetry noted above should be eliminated by aligning both sides on a common Java 25 toolchain, now that Spring Boot 4.1 permits it; this removes the sole uncontrolled variable in the present comparison. Second, these measurements predate the stable Spring AI 2.0.0 release — the application here builds against the 2.0.0-M4 milestone — so a re-run would establish whether the stable version changes build, startup, or memory behaviour.
