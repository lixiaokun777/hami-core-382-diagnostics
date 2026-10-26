# HAMi-core #382: matched startup captures

Collected for https://github.com/Project-HAMi/HAMi-core/issues/382, following the maintainer's filtered-trace request: https://github.com/Project-HAMi/HAMi-core/issues/382#issuecomment-6407824853.

This repository/release contains sanitized diagnostic artifacts only. No model weights, images, credentials or raw internal support bundle are published.

## Observed diagnostic outcomes

| Mode | Outcome | Observation |
|---|---|---:|
| HAMi injected/enforced | No successful health before intentional, maintainer-approved stop | 3649.01 s (approximately 60 min 49 s) |
| Effective bypass, CUDA_DISABLE_CONTROL=true | Health HTTP 200; completion HTTP 200, 16 output tokens, nonempty | 855.74 s (14 min 16 s) |

These are instrumented diagnostic outcomes. The earlier untraced 23 min / 7 min startup comparison remains the performance reference. The enabled outcome is censored, not a completed startup duration.

| Observed event / metric | Injected/enforced | Effective bypass |
|---|---:|---:|
| Non-thread clone/clone3 returns in selected filter | 92 | 2527 |
| Thread birth events | 6059 | 8911 |
| Successful nvrtc_compile exec events / unique task IDs | 77 / 77 | 177 / 177 |
| Completed compiler exec-to-exit median | 52.16 s | 8.87 s |
| Completed compiler exec-to-exit p95 | 88.49 s | 12.23 s |
| HAMi allocation failure log messages | 80 | 0 |
| PyTorch allocation warning log messages | 80 | 80 |

Compiler lifetimes include actual compilation, and the two runs completed different portions of startup. These are observed distributions, not matched-kernel or isolated cuInit benchmarks.

The last observed compiler exit in the enabled trace was 578.49 s after application exec; no further successful nvrtc_compile execs were recorded before the cutoff. The remaining interval to cutoff was 3070.52 s. Other process activity remains available in the full selected trace and exec-program counts.

Artifacts: [GitHub Releases](https://github.com/lixiaokun777/hami-core-382-diagnostics/releases). SHA-256 checksums are supplied for both individual data files and release assets.

## Method

Both runs use the same eight-H100 Kubernetes node, image, model arguments and GPU resource limits as the issue, with fresh application compilation caches and `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:False`. Host filesystem page cache was not flushed, clocks/CPU/NUMA placement were not pinned, and the enabled-control run preceded the disabled-control run.

The selected syscall set is exactly the requested set:

```
execve,clone,clone3,exit_group,flock,fcntl,openat
```

A full unfiltered host-attach attempt was stopped after the maintainer requested this filter; those preliminary/invalid/preflight attempts are not included in the matched results.

To avoid the severe perturbation of trapping every syscall in attach mode, the final matched runs launch vLLM under `strace --seccomp-bpf -f -tt -T -e trace=...`. The tracer runs as the application's parent in the container PID namespace. The same host-provided strace 5.16 and its private loader/dependencies were mounted for the tracer in both runs; those private loader options are not inherited by the application after exec. The workload image itself was not rebuilt, and no extra capability was added.

The enabled diagnostic run mounts `/etc/ld.so.preload` containing the deployed `/usr/local/vgpu/libvgpu.so`, and verifies that the actual application and sampled compiler children map that library. The final `CUDA_DISABLE_CONTROL=true` bypass run does **not** force the preload: absence of the library in application/child mappings is the required runtime check. An earlier attempted control that retained the forced preload still produced HAMi allocation errors and was rejected; it is not included in the matched dataset.

Thus the treatment is **injected/enforced library versus effective library bypass**, matching the operational distinction in the original report. The model/image/GPU reservation and tracer syscall/log settings remain matched. The preload mount is intentionally absent in the bypass treatment; this is disclosed rather than claiming that only a variable changed while a library was still forced in. The enabled tracer itself also inherits the preload, which is an additional instrumentation effect. These diagnostic captures do not replace the untraced startup/performance measurements in the issue.

The gated launcher starts the tracer only after log following is in place. The launcher's environment declares the bypass variable; `strace -E` sets the **application** environment to `LIBCUDA_LOG_LEVEL=4` and either unsets `CUDA_DISABLE_CONTROL` (enabled run) or sets it to `true` (disabled run). The actual application environment, library mapping state and seccomp state are saved in `*-capture-settings.json`. The Pod startup allowance was increased equally for both runs; model, memory utilization, TP size and GPU limits were unchanged.

Level-4 logging remains configured for the application during both diagnostic runs; the provided verbose log excerpt is the first ten minutes, as requested. The application startup log preserves non-debug messages through the recorded capture cutoff. The complete sanitized level-4 startup logs are also included as optional `*-debug-full.log.gz` files, so later profiling windows can be inspected without another collection.

## Files

- `enabled-trace.txt.gz` / `disabled-trace.txt.gz`: complete selected syscall records through first successful health check; decompress with `gzip -dc FILE`.
- `*-debug-first10min.log.gz`: sanitized first ten minutes of the corresponding level-4 log, or through the recorded capture cutoff if earlier.
- `*-debug-full.log.gz`: optional complete sanitized level-4 log through the recorded capture cutoff.
- `*-application-startup.log`: non-debug startup messages through the recorded capture cutoff, including warnings/errors and stage summaries.
- `*-processes.json`: successful exec events, process births, thread births and exit events. Trace task IDs are **container namespace IDs**, not host PIDs.
- `*-compiler-env-samples.jsonl`: up to ten observed compiler-process environment/mapping snapshots. Host and container IDs are explicitly distinguished. This is a sample; the trace provides event counts under the requested filter.
- `*-capture-settings.json`: runtime evidence, tracer command and application environment.
- `*-result.json`, `*-smoke.json`, `summary.json`, `environment.json`: timing, completion validation, aggregate counts and version/hash evidence.
- Sanitized diagnostic Deployment specs and source capture scripts may be supplied as supplementary context.

Trace timestamps are time-of-day **UTC**. CRI log prefixes use **UTC+08:00**; vLLM's inner timestamps are UTC. Compare using the timezone offset, not literal printed clock strings. Gate/ready timestamps use the same worker clock.

The enabled run is bounded by the maintainer-approved approximately 60-minute observation window and is marked stopped before readiness, not a completed startup time or fatal model failure. The confirmation is https://github.com/Project-HAMi/HAMi-core/issues/382#issuecomment-6408918797. The bypass run stops at first health. Counts and trace excerpts use those recorded cutoffs, excluding subsequent completion smoke requests and shutdown. The requested filter does not include fork/vfork/execveat, so reported clone-based process birth events are observations under this filter, not a guarantee of every possible process creation mechanism. NVRTC counts refer to successful execve records for the compiler binary. Process births and thread births are separate; NVRTC exec events and unique NVRTC task IDs are also separate. NVRTC exec-to-exit duration includes actual compilation and is **not** a direct measurement of cuInit alone. Summed syscall elapsed durations across concurrent tasks are **not** CPU time or overall elapsed time.

## Redaction

Internal addresses, GPU UUIDs and private storage/registry paths are replaced. Exec argument strings in the trace are masked; the sanitized workload configuration supplies the relevant arguments separately. Any traced read/write/network buffers are masked as well (these calls are outside the final selected filter). Ordering, task IDs, syscall names, return values, lock addresses/arguments and durations are retained. No raw runtime inspection or Docker environment dump is attached.
