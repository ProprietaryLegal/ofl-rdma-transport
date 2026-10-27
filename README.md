# DeepSeek-V4.1-Flash on DGX Spark / GB10 and x86: ofl RDMA request/response transport

A small C API places response bytes into caller-owned host buffers over RDMA RC or TCP. It carries remote Engram lookups and drafter requests in a system with two DGX Spark (GB10) nodes and an x86 offload host with RTX 3090s. Python bindings, a discovery probe and correctness tests are included.

**Hugging Face:** [Card and files](https://huggingface.co/ProprietaryLegal/ofl-rdma-transport) · **Stack overview:** [GitHub](https://github.com/ProprietaryLegal/DeepSeek-V4.1-Flash-2xDGX-Spark-x86-offload) · **Collection:** [All entries](https://huggingface.co/collections/ProprietaryLegal/deepseek-v41-flash-on-2x-dgx-spark-x86-offload-6adab604a517b426ded3916e)

<!-- pli:blurb:start -->

> Published by [PLI Labs](https://proprietarylegal.ai/labs.html), the open research arm of [Proprietary Legal Intelligence](https://proprietarylegal.ai). PLI builds AI systems around a legal organization's own work product, combining retrieval, staged workflows, validation, and attorney review. Visit [proprietarylegal.ai](https://proprietarylegal.ai) to explore our work and discuss what your organization needs.

<!-- pli:blurb:end -->

<!-- pli:update:2026-10-27:start -->

**Update 2026-10-27:** the full DSV41 source is now public — [dsv41-offload](https://github.com/ProprietaryLegal/dsv41-offload) (every component; `production`, `main` and per-experiment `lane/*` branches), with focused repos for the [measurement harness](https://github.com/ProprietaryLegal/dsv41-window-harness) and [sampled drafting](https://github.com/ProprietaryLegal/dsv41-sampled-drafting). Status and lessons: [2026-10-27 post](https://github.com/ProprietaryLegal/pli-labs/blob/main/research/dgx-spark-offload/2026-10-27-status-and-code.md).

<!-- pli:update:2026-10-27:end -->


## Status

2026-10-23 — production: TCP and RDMA are verified on the tested direct links, and ofl has carried the production offloads since 2026-10-22; this is a specialized transport, with no general-network security qualification.

## What it is

One client connection submits `(op, generation, payload)` and receives `(status, bytes)`. The caller registers destination memory and supplies up to 16 ranges. Responses go directly into those ranges.

The little-endian protocol has a versioned 64-byte header. A TCP bootstrap exchanges service identity, queue-pair parameters and bounded buffer limits. An RDMA request is a SEND into a pre-posted receive slot. The server writes response data, optionally pipelined as pieces become ready, then writes the completion header with RDMA WRITE WITH IMMEDIATE on the same reliable-connected queue pair. Relaxed ordering is disabled, so completion follows data placement. The bootstrap socket remains open for liveness.

The client busy-polls against a deadline. Timeout breaks the queue pair before returning, preventing late writes into reused buffers. Reconnect creates a new queue pair and re-registers regions. Handler refusals can leave the connection usable. The TCP backend uses the same API and copies received data into the supplied ranges.

Submission/wait tickets allow bounded requests in flight; handling remains ordered per connection. `ofl_client_payload_buf` exposes a stable send-slot payload area to avoid a request copy. Each client is used by one thread at a time; server handlers may run concurrently on different connections.

## Why

Engram and speculative drafting need bounded latency and exact response placement. Reusing registered host buffers avoids per-request registration and allows pipelined row transfers. Fatal timeout semantics make local fallback safe without accepting a late remote response.

## When to use it

Use it for private, direct RoCE links when the response belongs in pre-registered host staging memory. Choose the TCP backend for diagnosis or an explicitly implemented application fallback. Reconnection and fallback policy belong to the application.

## When not to use it

Do not expose it to untrusted networks. It provides no encryption or authentication of peers. It is not a general messaging fabric or a GPU-memory transport, and adding hardware bandwidth alone does not remove inference scheduling overhead.

## Requirements

- Linux on x86 or aarch64, a C++ compiler, pthreads and libibverbs development/runtime packages. Python bindings use ctypes.
- RoCE-capable NICs and a configured IPv4 direct link for RDMA. The library discovers the matching RoCE v2 GID from the bound local address and rejects mismatched overrides.
- Host memory with a lifetime covering all in-flight writes, sufficient memory-lock allowance, and negotiated request/response capacities.
- Tested hardware: ConnectX-7 on DGX Spark and ConnectX-5 Ex on x86, 40 Gb/s DAC links; Ethernet MTU 9000, active verbs MTU 4096. The path uses the smaller active verbs MTU.

## Measured effect

These are our host-memory measurements. Perftest's small-message latency convention differs from full application RTT. No row below is an inference speedup claim. Context is not applicable unless explicitly stated. Dates are UTC.

The link suite ran on 2026-10-22 with ConnectX-7 ↔ ConnectX-5 Ex, 40 Gb/s links and Ethernet MTU 9000. Single-link cases used iteration counts; simultaneous cases used overlapping three-second windows.

| Payload / operation / concurrency | Spark A link | Spark B link |
|---|---:|---:|
| Perftest RDMA send 64 B; one link at a time | 1.38 µs average | 1.38 µs average |
| RDMA write 64 MiB Spark → x86; one link at a time | 39.23 Gb/s | 39.23 Gb/s |
| RDMA write 64 MiB x86 → Spark; one link at a time | 39.21 Gb/s | 39.21 Gb/s |
| Simultaneous 64 MiB RDMA read/write direction cases | 78.38 Gb/s aggregate across both links | Included in aggregate |

The link suite completed 195 normal-path cases. Per-case iteration counts differ; the report does not provide a common repeated-run count or uncertainty for these headline cells. The alternate PCIe-function route failed queue-pair setup under its existing IPv6 configuration and has no bandwidth result.

The ofl ping-pong suite ran on 2026-10-22 on the same Spark/x86 links with the serve idle and one client per tested link. One recorded suite; no independent repeat count is reported. Values are full RTT in µs.

| Payload | RDMA p50 A / B | RDMA p99 A / B | TCP p50 A / B |
|---|---:|---:|---:|
| 64 B echo | 5.6 / 6.6 µs | 17.1 / 9.1 µs | 34.7 / 30.5 µs |
| 288-row response | 32.9 / 32.3 µs | 42.2 / 37.8 µs | 113 / 148 µs |
| 4,096-row response | 371 / 358 µs | 393 / 375 µs | 1,301 / 1,018 µs |
| 16,384-row response | 1,086 / 1,041 µs | 1,109 / 1,058 µs | 5,269 / 3,227 µs |
| 49,152-row response | 3,027 / 2,866 µs | 3,082 / 2,893 µs | 11,395 / 13,615 µs |

The application tests checked every response byte. These full RTTs must not be confused with the lower perftest send latency.

The CPU-only x86 ConnectX-5 Ex NIC-loopback test ran on 2026-10-22 with one client, a 184,416 B request and a 24 B reply. Each cell averages two matching ABBA case percentiles; inference context does not apply.

| Scheduling / wait mode | Copy p50 | In-place p50 |
|---|---:|---:|
| Burst, unpinned | 22.865 µs | 18.235 µs |
| Own caller/server pinned, 42-ms cadence, busy poll | 36.435 µs | 26.185 µs |
| Same pinning and cadence, event wait | 104.299 µs | 94.720 µs |

The loopback experiment covered 40 ABBA cases, 8,160 measured requests and 480 warmups, with zero bad checks. These percentiles are averages of matching case percentiles, not pooled percentiles. NIC loopback traverses no physical cable and includes neither Spark staging nor drafter compute.

| Date / application conditions | Finding |
|---|---|
| 2026-10-22–2026-10-23; combined remote Engram + one-card RTX 3090 drafter; GB10 TP2; short through 300K plus concurrent capacity suite; suite counters, not independent repeats | Engram rank 0: 15,358 lookups, 37.9 M rows per table, mean ~205–210 µs, one 62 ms outlier; 0 fallbacks and 0 transport errors |
| 2026-10-22; guarded disk-to-disk Engram delivery over the direct link; one file, verification included, no inference context | Shard 47: 101,535,150,936 bytes in 359.931 s, 282.10 MB/s; delivery rate, not an RDMA bandwidth benchmark |

The suite counter is rank-specific; it does not establish that each rank had the same row count. End-to-end transfer includes disk reads and hashing. The later shard transfer paused for production traffic and is not comparable to an isolated link rate.

## Upstream claims

Not applicable. These tables contain our measurements. DeepSeek, HeNryous, vLLM and exllamav3 supply model/engine components used by the applications, not performance claims for this transport. No GPUDirect capability is claimed: all ofl regions are host memory.

## Deploy

From the repository root:

```bash
make -C code/transport
code/transport/ofl_probe
```

Use [ofl.h](code/transport/ofl.h) as the API reference and [API.md](code/transport/API.md) for protocol details. Select `OFL_BACKEND_RDMA` or `OFL_BACKEND_TCP`, set `server_ip`, `local_ip` and an unused port, and use the same service name and compatible limits at both ends. Servers refuse wildcard binding; bind the intended direct-link address. Register the destination, submit a call, check its return code and handler status, and trust only the returned response length.

Python:

```bash
export OFL_LIB="$PWD/code/transport/libofl.so"
export PYTHONPATH="$PWD/code/transport"
python -c 'import ofl; print(ofl.OP_ECHO)'
```

The protocol guide contains deployment-specific example addresses; replace them with your own. It also retains older test-count wording; the reviewed count below is newer.

## Rollback

Drain callers before closing the application connection and server. Select TCP only if the application's tested fallback policy permits it, or disable the remote feature in the consuming engine. The transport does not itself restart a service or switch applications to TCP. After a fatal timeout, reconnect explicitly before reusing the connection.

## Limits and known issues

- Private links only; service-name matching is not peer authentication.
- Host buffers only. Any CUDA pinning or registration integration belongs to the application.
- Busy polling consumes CPU while waiting; event sleep trades CPU use for wake latency.
- Request and response sizes, registered ranges and in-flight capacity are bounded at connection setup. Buffer ownership must survive completion or connection teardown.
- One connection handles requests in submission order. This is not an unordered multiplexed RPC engine.
- Large transfers share the Spark PCIe uplink with tensor-parallel traffic. Physical line rate is not application or inference throughput.

## Repository layout

| Path | Purpose |
|---|---|
| [code/transport/ofl.h](code/transport/ofl.h) | C API, errors and wire structures |
| [code/transport/ofl.cpp](code/transport/ofl.cpp) | RDMA and TCP client/server implementations |
| [code/transport/ofl.py](code/transport/ofl.py) | ctypes binding |
| [code/transport/ofl_pingpong.cpp](code/transport/ofl_pingpong.cpp) | Byte-checked application RTT microbenchmark |
| [code/transport/ofl_probe.c](code/transport/ofl_probe.c) | Read-only local device/GID discovery |
| [code/transport/test_ofl.py](code/transport/test_ofl.py) | Timeout, late-write, refusal and reconnect checks |
| [code/transport/API.md](code/transport/API.md), [Makefile](code/transport/Makefile) | Protocol guide and build rules |

## Tests

Build first, then use an available local RoCE interface and unused port. The CPU suite exercises TCP and same-host RDMA; it cannot run on an ordinary machine without a usable RoCE device.

```bash
OFL_LIB="$PWD/code/transport/libofl.so" \
  python code/transport/test_ofl.py --ip LOCAL_DIRECT_IP --port 18601
```

The independent review recorded 26/26 checks passing, including no late data after timeout and stable payload-buffer addresses across reconnect. The script exits zero when all checks pass and nonzero otherwise. No NIC tests or service actions were run for this documentation update.

## Credits and licences

PLI Labs code is Apache-2.0. There are no vendored third-party files or upstream patch sets in this transport directory. libibverbs/rdma-core is an external system dependency with its own licence terms. See [NOTICE](NOTICE) and the full [Apache-2.0 text](LICENSES/Apache-2.0.txt). DeepSeek supplies the model; HeNryous and sudoingX supply the DGX Spark serving lineage, vLLM the engine, and turboderp/exllamav3 the quantized drafter runtime used by consuming applications.

## Related

| Entry | GitHub | Hugging Face |
|---|---|---|
| Async host-flag handoff | [Source](https://github.com/ProprietaryLegal/dsv41-async-hostflag-handoff) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-async-hostflag-handoff) |
| Remote RTX 3090 drafter | [Source](https://github.com/ProprietaryLegal/dsv41-remote-dspark-drafter-3090) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-remote-dspark-drafter-3090) |
| NVLink drafter | [Source](https://github.com/ProprietaryLegal/dsv41-dspark-drafter-tp2-nvlink) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-dspark-drafter-tp2-nvlink) |
| Remote Engram | [Source](https://github.com/ProprietaryLegal/dsv41-remote-engram-rdma) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-remote-engram-rdma) |
| Native SM86 drafter | [Source](https://github.com/ProprietaryLegal/dsv41-native-1017-drafter-sm86) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-native-1017-drafter-sm86) |
| Benchmark tools | [Source](https://github.com/ProprietaryLegal/dsv41-longctx-bench-tools) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-longctx-bench-tools) |
| Exact acceptance ceiling | [Source](https://github.com/ProprietaryLegal/dsv41-exact-acceptance-ceiling) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-exact-acceptance-ceiling) |
| Full stack overview | [Source](https://github.com/ProprietaryLegal/DeepSeek-V4.1-Flash-2xDGX-Spark-x86-offload) | [Card](https://huggingface.co/ProprietaryLegal/DeepSeek-V4.1-Flash-2xDGX-Spark-x86-offload) |
| Offload findings | [Source](https://github.com/ProprietaryLegal/dsv41-spark-offload-findings) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-spark-offload-findings) |

Upstream forks: [HeNryous lineage](https://github.com/ProprietaryLegal/deepseek-v41-1017-dgx-spark) · [sudoingX lineage](https://github.com/ProprietaryLegal/dgx-spark-deepseek-v41).

[Collection](https://huggingface.co/collections/ProprietaryLegal/deepseek-v41-flash-on-2x-dgx-spark-x86-offload-6adab604a517b426ded3916e) · [Hub](https://github.com/ProprietaryLegal/pli-labs/tree/main/research/dgx-spark-offload).

<!-- pli:footer:start -->

---

## About PLI Labs

“Your proprietary work product is your proprietary intelligence.” PLI Labs is the open research arm of Proprietary Legal Intelligence. We publish legal model prunes, serving recipes for owned hardware, and open-source tools so readers can inspect our methods and test our claims before hiring us.

Founded by a practicing trial lawyer, PLI builds AI systems around a legal organization's own work product: retrieval, staged workflows, validation, and attorney review. The approach grew from systems running on live matters in a litigation practice. Our consulting work supports deployment through approved frontier APIs, private cloud, or open-weight models on hardware you own. Explore the research or contact us about your organization's work.

[Website](https://proprietarylegal.ai) · [PLI Labs](https://proprietarylegal.ai/labs.html) · [Research](https://proprietarylegal.ai/research/index.html) · [GitHub](https://github.com/ProprietaryLegal) · [Hugging Face](https://huggingface.co/ProprietaryLegal) · [Contact](https://proprietarylegal.ai/contact.html)

Proprietary Legal Intelligence LLC is a technology consultancy, not a law firm, and does not provide legal services or legal advice.

<!-- pli:footer:end -->
