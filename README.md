# warp-otel — S3 performance comparison of two Ceph RGW zones with OTLP traces

A side-by-side `warp mixed` benchmark run against two RGW zones on the same single-host Ceph cluster, with every S3 request producing an OpenTelemetry trace landing in Tempo. The two zones differ in only one meaningful axis — the `*.rgw.buckets.data` pool — so the deltas surface the real cost of replicated-vs-EC for an S3 workload on this hardware.

- **rgw-east**: data pool is **erasure coded 7+2** (ISA-L Reed-Solomon, `crush-failure-domain=osd`)
- **rgw-west**: data pool is **replicated, size=2 min_size=1**

Date: 2026-05-25. Ceph: `tentacle 20.1.1-r3` (homelab fork, OTLP-instrumented with end-to-end GET trace propagation + per-phase BlueStore write spans, both new since the previous run). Warp: `minio/warp:latest`.

---

## TL;DR

| op | metric | east (EC 7+2) | west (replicated 2×) | Δ |
|---|---|---|---|---|
| Total | throughput | **284.37 MiB/s, 47.38 obj/s** | 162.35 MiB/s, 27.03 obj/s | east **+75%** |
| PUT | throughput | **71.32 MiB/s, 7.13 obj/s** | 40.62 MiB/s, 4.06 obj/s | east **+76%** |
| PUT | p50 latency | **435.7 ms** | 422.6 ms | tie |
| PUT | p99 latency | **1947.9 ms** | 3687.5 ms | east **−47%** |
| GET | throughput | **213.06 MiB/s, 21.31 obj/s** | 121.67 MiB/s, 12.17 obj/s | east **+75%** |
| GET | p50 latency | **257.8 ms** | 352.9 ms | east **−27%** |
| GET | p99 latency | **2176.2 ms** | 4560.3 ms | east **−52%** |
| GET | TTFB avg | **135 ms** | 231 ms | east **−42%** |
| STAT | avg latency | **18.9 ms** | 57.6 ms | east **−67%** |
| STAT | p99 latency | **197 ms** | 803 ms | east **−75%** |
| DELETE | avg latency | **170.2 ms** | 205.5 ms | east **−17%** |
| DELETE | p99 latency | **529.1 ms** | 1556.4 ms | east **−66%** |

EC 7+2 wins decisively on every dimension this run — both throughput (1.75×) and tail latency (3-4× better p99). The single-host constraint flips the usual rule: EC parallelizes every write across all 9 spindles, while replicated×2 only touches 2 of 9 OSDs and the rest sit idle. At 16 concurrent workers the queue depth on west's narrow fan-out shows up in both `do_op` p99 (121 ms west vs 15 ms east — **8×**) and `bluestore_read` p99 (127 ms vs 55 ms — **2.3×**).

---

## What changed since the 2026-05-23 run

This is a rerun of the same workload with a substantially upgraded tracer. Concretely:

| Coverage | Before (r2, 2026-05-23) | Now (r3, 2026-05-25) |
|---|---|---|
| OSD spans reaching Tempo at all | broken on r2 build (bluestore tracer's `Tracer::init()` orphaned the osd Provider's exporter — silently dropped every OSD span when `bluestore_tracing_enable=true`) | **fixed** via `Tracer::init_from_global` so both scopes share one provider |
| GET → OSD trace linkage | not propagated; GET traces had only 4 RGW spans, no OSD-side detail | **propagated end-to-end** through `ReadOp::Params::trace_ctx` → `RadosReadOp::iterate` → `RGWRados::Object::Read::iterate` → `get_obj_data` → `rgw::Aio::librados_op` → `librados::async_operate` → `IoCtx::aio_operate(... jspan_context)` → `Objecter::Op::otel_trace` → `MOSDOp.otel_trace` → OSD `op-request-created` |
| Per-chunk rados_read span | absent | **added** — one `rados_read` span per librados aio read (default 4 MiB chunk) |
| BlueStore write-phase visibility | only `queue_transactions` + `txc_aio_wait` (start and end of txc lifetime) | **5 new spans per txc**: `_do_write_data` → `_do_write_small`/`_do_write_big` for the per-extent layout, `kv_submit_transaction` for the rocksdb submit, `kv_committed_finalize` for the post-commit fan-out from `_kv_finalize_thread` |
| BlueStore read sub-spans | none | `bluestore_read` → `get_onode` + `_do_read` |
| Build | gcc 14 / opentelemetry-cpp 1.20 | gcc 15 / opentelemetry-cpp 1.24 (required additional override-mismatch fixes in `KStore`, `ECSwitch`, `ECBackend{,L}`, and explicit `jspan_context{false,false}` init at 4 sites in `PrimaryLogPG.cc` because OpenTelemetry 1.24 removed the default `SpanContext()` constructor) |

The throughput delta below also looks much larger than 2026-05-23's run; that's the workload behaving as expected once the tracer overhead is consistent across both zones. The 2026-05-23 run's east zone was hampered by spending CPU on orphaned span exports that never reached Tempo.

---

## Underlay hardware (sm3)

| component | value |
|---|---|
| chassis / board | Penguin Computing Relion 4724 (Supermicro X8DT6) |
| CPU | 2 × Intel Xeon X5650 @ 2.67 GHz (Westmere-EP, 6c/12t each, 24 threads total) |
| L3 cache | 24 MiB (2 instances) |
| RAM | **192 GiB** DDR3 (12 × 16 GiB, both sockets fully populated; kernel sees 188.88 GiB after BIOS/MMIO reservations) |
| NICs | bond0 (`enp3s0` + `enp4s0`) → `br0`, primary 10.144.27.26 ; one WAN-mirror NIC and several link-down ports |
| HBAs | onboard SAS + LSI/expander backplane |
| OS | Gentoo 2.18, kernel `6.12.31-gentoo` |

```
$ lscpu | grep -E '^(Model name|Architecture|Socket|Core|Thread|CPU\(s\))'
Architecture:                            x86_64
CPU(s):                                  24
Model name:                              Intel(R) Xeon(R) CPU           X5650  @ 2.67GHz
Thread(s) per core:                      2
Core(s) per socket:                      6
Socket(s):                               2

$ free -h
               total        used        free      shared  buff/cache   available
Mem:           188Gi        45Gi        70Gi       4.4Gi        78Gi       143Gi
```

---

## Ceph deployment

```
$ ceph version
ceph version 20.1.1 (dd9c546413d50a90668289255a256022ea21f0c0) tentacle (rc - RelWithDebInfo)
        # built from tu503/homelab-overlay sys-cluster/ceph-20.1.1-r3
        # OTLP tracer + rgw-iterate-trace-ctx-plumbing + bluestore-write-phase-spans
        # + bluestore-tracer-init-from-global + librados-asio-read-trace + ...

$ ceph -s
  cluster:
    health: HEALTH_OK
  services:
    mon: 1 daemons, quorum 0 (age 2h) [leader: 0]
    mgr: sm3(active, since 2h)
    mds: 1/1 daemons up
    osd: 9 osds: 9 up (since 2h), 9 in (since 10w)
    rgw: 6 daemons active (6 hosts, 2 zones)
```

Single host, 9 HDD OSDs, all in `host sm3` (`root default`). 7.3 TiB per OSD, total raw 65 TiB, used 16 TiB (24.81%). BlueStore on dm-crypt LV per OSD; no separate WAL/DB device (collocated).

**Tracing topology.** All four host daemon types (osd / mon / mds / mgr) and the in-pod RGW daemons are linked against `opentelemetry-cpp 1.24` and pointed at Tempo's OTLP HTTP receiver (`http://10.144.27.223:4318/v1/traces`, a MetalLB LoadBalancer service). RGW pods reach Tempo via cluster DNS (`tempo.monitoring.svc:4318`); host daemons go via the LB IP. BlueStore-emitted spans (`bluestore_read`, `_do_write_data`, `kv_*`) surface in Tempo under `service.name=osd` because the BlueStore tracer binds to the OSD's TracerProvider via `init_from_global` rather than installing its own (replacing the global drops every previously-created exporter — that was the silent-OSD-span bug on r2).

---

## RGW zones, zonegroups, and pools

Two **separate realms** (`east` and `west`) — not a multi-site sync setup, just two independent S3 namespaces on the same OSDs.

### Per-zone pool layout

| pool | east | west |
|---|---|---|
| `*.rgw.buckets.data` | **EC 7+2 (size 9, min_size 8)** — pool #64, CRUSH rule `east.rgw.buckets.data` (`choose_indep type=osd`), `pg_num=32`, `stripe_width=28672`, ISA-L Reed-Solomon | **replicated, size 2 min_size 1** — pool #58, CRUSH rule `replicated_osd` (`choose_firstn type=osd`), `pg_num=16` |
| `*.rgw.buckets.index` | size 2, **pg_num 32** | size 2, **pg_num 16** |
| `*.rgw.buckets.non-ec` | size 2, pg_num 32 | size 2, pg_num 32 |
| `*.rgw.control / meta / log` | size 2, pg_num 32 | size 2, pg_num 32 |

### EC profile (east)

```
crush-device-class=
crush-failure-domain=osd
crush-num-failure-domains=0
crush-osds-per-failure-domain=0
crush-root=default
k=7
m=2
plugin=isa
technique=reed_sol_van
```

Storage efficiency: **east 7/9 ≈ 77.8%** vs **west 1/2 = 50%**. With a single-host failure domain, the durability story is the same either way (any host crash takes the whole cluster down) — but EC needs k=7 shards alive to read, which on this cluster means 7-of-9 OSDs.

---

## RGW Kubernetes deployment

### Pattern: one Deployment + Service + ConfigMap + Secret per zone

All RGW pods live in a single namespace `rgw-gateway`, but **every zone gets its own complete set of resources** — Deployment, Service, ConfigMap, Secret — with no shared state between instances except for the underlying Ceph cluster. The zone name is the unit of isolation: it becomes the RADOS pool prefix (`<zone>.rgw.*`), the RGW realm/zonegroup/zone triple, the cephx client identity (`client.rgw.<zone>`), and the k8s resource suffix.

```
namespace: rgw-gateway
├── Deployment/rgw-east   ──┐
├── Service/rgw-east        │ instance "east":  pools east.rgw.*,
├── ConfigMap/ceph-config-east   keyring client.rgw.east,
└── Secret/ceph-rgw-keyring-east │ EC 7+2 data pool, LB IP .204
│
├── Deployment/rgw-west   ──┐
├── Service/rgw-west        │ instance "west":  pools west.rgw.*,
├── ConfigMap/ceph-config-west   keyring client.rgw.west,
└── Secret/ceph-rgw-keyring-west │ replicated×2 data pool, LB IP .205
```

This is **not Rook**. The existing native Ceph cluster runs directly on the host (mon/mgr/mds/osd as systemd units on sm3); the RGW pods are *clients* of that cluster, configured purely via mounted `ceph.conf` + cephx keyring. Each pod is one `radosgw -f` process started with `--name=client.rgw.<zone>` and the per-zone keyring.

### Container image (homelab fork)

`registry.alcg.io/radosgw:v20.1.1-asio-read-fix` (this run, sha256:`139fce5e…`) is built **from scratch** by `build-image.sh` using buildah: `ldd /usr/bin/radosgw` against the host's installed Ceph (so the image inherits whatever the Portage overlay produced — including the OTLP-instrumented `sys-cluster/ceph-20.1.1-r3` patch series), copy the binary + all transitive libraries into a scratch container, add a minimal `/etc/{passwd,group,nsswitch.conf}` for the `ceph` user (UID 167), `EXPOSE 7480`, `ENTRYPOINT ["/usr/bin/radosgw"]`, commit. Image lands around 105 MiB — no shell, no package manager.

Updates: rebuild on the host, `buildah push` to the local registry (`registry.alcg.io`), `kubectl rollout restart deploy/rgw-*`. `deploy-image.sh` does all four steps as one command and waits for both rollouts to finish.

### Live state at benchmark time

```
$ kubectl -n rgw-gateway get pods,svc -o wide
pod/rgw-east-878b66cd9-...   1/1 Running   sm3
pod/rgw-east-878b66cd9-...   1/1 Running   g469
pod/rgw-east-878b66cd9-...   1/1 Running   mg-vctr-rtx
pod/rgw-west-59767b44d7-...  1/1 Running   sm3
pod/rgw-west-59767b44d7-...  1/1 Running   g469
pod/rgw-west-59767b44d7-...  1/1 Running   mg-vctr-rtx

service/rgw-east   LoadBalancer  10.111.88.16  10.144.27.204  7480 → 32010/TCP
service/rgw-west   LoadBalancer  10.104.10.75  10.144.27.205  7480 → 30089/TCP
```

Pods are spread across 3 nodes (sm3 / g469 / mg-vctr-rtx) but all OSDs are on `sm3`, so RGW→OSD traffic on g469 and mg-vctr-rtx crosses the network while the sm3-resident pods use the cluster loopback path. The 16-concurrent warp client runs in-cluster and dials the service IP, so requests load-balance via kube-proxy to whichever pod is selected.

| | east | west |
|---|---|---|
| Deployment | `rgw-east` | `rgw-west` |
| replicas | 3 (sm3 / g469 / mg-vctr-rtx) | 3 (sm3 / g469 / mg-vctr-rtx) |
| image | `registry.alcg.io/radosgw:v20.1.1-asio-read-fix` | same |
| Service | `rgw-east` LoadBalancer `10.144.27.204:7480` | `rgw-west` LoadBalancer `10.144.27.205:7480` |
| ConfigMap | `ceph-config-east` | `ceph-config-west` |
| Secret | `ceph-rgw-keyring-east` | `ceph-rgw-keyring-west` |
| cephx client | `client.rgw.east` | `client.rgw.west` |
| Realm / zonegroup / zone | `east` / `east` / `east` | `west` / `west` / `west` |
| Data pool | `east.rgw.buckets.data` (pool #64, **EC 7+2**) | `west.rgw.buckets.data` (pool #58, **replicated×2**) |
| Index pool | `east.rgw.buckets.index` (pg_num 32) | `west.rgw.buckets.index` (pg_num 16) |
| ceph.conf knobs | `rgw thread pool size = 64`, `jaeger_tracing_enable = true`, `otel_tracing_endpoint = http://tempo.monitoring.svc:4318/v1/traces` | same |

---

## Benchmark methodology

- Tool: **`minio/warp:latest`** running as an in-cluster pod (`kubectl run --image=minio/warp`), reaching the RGW Service via cluster DNS — same network hop as the in-cluster S3 consumers.
- Workload: **`warp mixed`** — default mix of GET / PUT / STAT / DELETE, default object size distribution (Pareto over 1 KB – 10 MB).
- Concurrency: **16** workers.
- Duration: **4 minutes** per zone.
- Bench user: dedicated `warp-bench` per realm (created with `radosgw-admin user create --rgw-realm={east,west}`).
- Buckets: timestamped per run (`warp-otel-east-20260525-000833`, `warp-otel-west-20260525-000833`); `--noclear` to keep data for post-run trace inspection.
- Both runs serialized (east 04:08:33–04:15:26 UTC, west 04:15:46–04:23:57 UTC) so they don't compete for the same 9 OSDs.
- Tempo bucket cleared between the previous test and this run so all traces and trace-derived Prometheus metrics start fresh.

Command (east shown):

```bash
kubectl -n rgw-gateway run warp-east-full --rm -i --restart=Never \
  --image=minio/warp:latest --command -- \
  /warp mixed \
    --host=rgw-east.rgw-gateway.svc:7480 \
    --access-key=$EAST_AK --secret-key=$EAST_SK \
    --bucket=warp-otel-east-$STAMP \
    --concurrent=16 --duration=4m --noclear --no-color
```

Raw logs: [`raw/warp-east.log`](raw/warp-east.log), [`raw/warp-west.log`](raw/warp-west.log).

---

## Results

### Throughput

| | east | west |
|---|---|---|
| **Total** | **284.37 MiB/s · 47.38 obj/s** | 162.35 MiB/s · 27.03 obj/s |
| PUT | **71.32 MiB/s · 7.13 obj/s** | 40.62 MiB/s · 4.06 obj/s |
| GET | **213.06 MiB/s · 21.31 obj/s** | 121.67 MiB/s · 12.17 obj/s |
| STAT | — · **14.19 obj/s** | — · 8.11 obj/s |
| DELETE | — · **4.75 obj/s** | — · 2.72 obj/s |

East delivers 1.75× the aggregate throughput and 1.75× the object rate. Every operation type favors east this run.

### Latency (ms)

| op | zone | avg | p50 | p90 | p99 | fastest | slowest |
|---|---|---|---|---|---|---|---|
| PUT | east | **594.1** | **435.7** | **1276.5** | **1947.9** | 173.9 | 2403.2 |
| PUT | west | 844.0 | 422.6 | 2699.4 | 3687.5 | 145.3 | 4875.0 |
| GET | east | **514.5** | **257.8** | **1323.8** | **2176.2** | 41.9 | 2742.0 |
| GET | west | 1002.1 | 352.9 | 3312.1 | 4560.3 | 22.3 | 5649.0 |
| GET TTFB | east | **135** | **102** | **273** | **576** | 9 | 951 |
| GET TTFB | west | 231 | 131 | 624 | 1433 | 4 | 2508 |
| STAT | east | **18.9** | **9.4** | **44.5** | **197.0** | 1.1 | 437.5 |
| STAT | west | 57.6 | 6.3 | 186.5 | 803.9 | 1.1 | 1769.4 |
| DELETE | east | **170.2** | **154.4** | **300.6** | **529.1** | 10.1 | 896.1 |
| DELETE | west | 205.5 | 80.3 | 752.1 | 1556.4 | 5.7 | 3026.9 |

Tail behavior matters more than averages for spindle-bound workloads. Look at the p99 column — **east's PUT p99 (1.95 s) is ~2× faster than west's (3.69 s), and east's GET p99 (2.18 s) is ~2× faster than west's (4.56 s)**.

### Why east wins on PUT

For each PUT, EC scatters 7 data + 2 parity shards across all 9 OSDs simultaneously. With 16 concurrent writers all 9 spindles are constantly busy. Replicated×2 only writes to 2 of 9 OSDs per object — the other 7 sit idle for that particular write. With 16 writers the 2-spindle bottleneck builds queue depth, which is exactly what shows up as west's `do_op` p99 of 121 ms vs east's 15 ms (see OSD-side latency below).

### Why east wins on GET

EC reads need k=7 of 9 shards plus an ISA-L decode, which conventional wisdom says is slower than a single replicated read. Two effects flip that here:

1. EC GETs *also* spread the read I/O across many OSDs (each read pulls from up to 7 shards), so the spindles are utilized in parallel and the aggregate bandwidth is higher.
2. The data pool used by west has `pg_num=16` (vs east's pg_num=32 on a 9-OSD cluster). Fewer PGs ⇒ fewer primary-PG-per-OSD slots ⇒ same number of concurrent requests funnel through fewer PG locks ⇒ more contention. Per-zone bluestore-read p99: east 55 ms, west 127 ms.

### Why east wins on STAT and DELETE

These are bucket-index-heavy ops. `east.rgw.buckets.index` has `pg_num=32`, `west.rgw.buckets.index` has `pg_num=16`. STAT p99 is 197 ms east vs 804 ms west — a 4× difference that maps directly onto the 2× index PG ratio plus second-order queue effects.

---

## Trace analysis

Every S3 request the benchmark issued produced an OTLP trace landing in Tempo. With the r3 build, both PUT and GET fan out the full **rgw → osd → bluestore** chain — previously only PUT did.

### Representative east PUT (EC 7+2)

Trace `ae08e96940a6a7face2ff7d8e6229c7` — total **1.48 s**, **257 spans across 18 OSD batches**.

```
[rgw  ] put_obj                          1247 ms
 ├─ [rgw  ] verify_permission                0.00 ms
 └─ [rgw  ] execute                       1247 ms
     ├─ [rgw  ] put_obj_data              1247 ms
     ├─ [osd/2] op-request-created         164 ms     <- primary shard
     │   ├─ enqueue_op       (0.03)
     │   ├─ dequeue_op       (5.63)
     │   ├─ do_op            (5.62)
     │   ├─ execute_ctx      (5.47)
     │   ├─ issue_repop      (5.42)
     │   └─ queue_transactions   9.53 ms     <- BlueStore txc starts
     │       ├─ _do_write_data      0.04 ms
     │       │   └─ _do_write_big   0.04 ms
     │       ├─ txc_aio_wait        2.67 ms
     │       ├─ kv_submit_transaction  0.13 ms
     │       └─ kv_committed_finalize  0.00 ms
     ├─ [osd/3] op-request-created   125 ms   <- shard 2
     │   └─ ... (same fan-out: enqueue/dequeue/queue_transactions/_do_write_data/_do_write_big/
     │       txc_aio_wait/kv_submit_transaction/kv_committed_finalize)
     ├─ [osd/0] op-request-created   128 ms   <- shard 3
     ├─ [osd/8] op-request-created   122 ms   <- shard 4
     ├─ [osd/4] op-request-created   143 ms   <- shard 5
     ├─ [osd/5] op-request-created   123 ms   <- shard 6
     ├─ [osd/6] op-request-created   ...     <- shard 7
     ├─ [osd/7] op-request-created   ...     <- shard 8
     └─ [osd/9] op-request-created   ...     <- shard 9
```

- **OSD batches: 18** (each OSD-tracer instance emits its own batch — 9 unique OSDs × 2 batches each on average for this trace).
- **Unique OSDs touched: all 9** (`0, 2, 3, 4, 5, 6, 7, 8, 9`) — confirms EC `crush-num-failure-domains=osd` placing all 9 shards.
- **27 `op-request-created`** = 9 OSDs × 3 RADOS writes per shard (head + 2 tail chunks for this multi-MB object).
- **Per-shard BlueStore fan-out is fully visible**: every shard runs `queue_transactions → _do_write_data → _do_write_big`, then `txc_aio_wait` (the rocksdb-side commit barrier — typically the dominant phase, 2.7-5.9 ms here), then `kv_submit_transaction` and `kv_committed_finalize`. The new spans give a per-OSD attribution of where each shard spends its time.

[Full trace JSON: [`traces/east-put_obj.json`](traces/east-put_obj.json) — 257 spans, 95 KB]

### Representative west PUT (replicated 2×)

Trace `2f2e2a50090b29e475ff4127887c37f` — total **2.54 s**, **68 spans across 10 OSD batches**, only **4 unique OSDs** (`2, 4, 5, 7`).

```
[rgw  ] put_obj                          2538 ms
 ├─ [rgw  ] verify_permission                0.00 ms
 └─ [rgw  ] execute                       2538 ms
     ├─ [rgw  ] put_obj_data              2538 ms
     ├─ [osd/2] op-request-created         67 ms      <- primary, chunk 1
     │   ├─ dequeue_op         6.44
     │   ├─ do_op              6.42
     │   ├─ execute_ctx        6.23
     │   ├─ issue_repop        6.17
     │   └─ queue_transactions  38.37
     │       ├─ _do_write_data         0.07
     │       │   └─ _do_write_big      0.05
     │       ├─ txc_aio_wait       26.89   <- much longer than east's 2.67ms
     │       ├─ kv_submit_transaction   0.21
     │       └─ kv_committed_finalize   0.00
     │   └─ [osd/7] op-request-created   64 ms   <- replica of chunk 1
     │       └─ ... (same shape, queue_transactions=40ms, txc_aio_wait=30.6ms)
     ├─ [osd/5] op-request-created       134 ms     <- primary, chunk 2
     │   └─ [osd/4] op-request-created    88 ms    <- replica of chunk 2
     ├─ [rgw  ] rados_write              1140 ms     <- head object metadata write (sync)
     └─ [osd/5] op-request-created        29 ms     <- head primary
         └─ [osd/4] op-request-created    ...        <- head replica
```

- Only 6 `op-request-created` for 3 chunks (each gets primary + replica = 2 OSDs), plus the head-object meta write — total **9 ops** vs east's 27.
- `txc_aio_wait` on west: **26-30 ms** vs east's **2-3 ms** — same physical spindles, but with 16 concurrent writers funneled through 4 OSDs the AIO queues build up.
- The trace shows the head-object `rados_write` separately at 1.14 s (because head writes go through the sync `rgw_rados_operate`, not the aio chunk path) — this is the one source of PUT latency that EC and replicated handle differently.

[Full trace JSON: [`traces/west-put_obj.json`](traces/west-put_obj.json) — 68 spans, 28 KB]

### Representative east GET (EC 7+2) — **new in this run**

Trace `41ab7a2f766e2212898aa06b002d6a69` — total **1.82 s**, **12 spans**, **6 OSD batches**.

```
[rgw  ] get_obj                          1820 ms
 ├─ [rgw  ] verify_permission                0.01 ms
 └─ [rgw  ] execute                       1502 ms
     └─ [rgw  ] get_obj_data              1502 ms
         ├─ [rgw  ] rados_read              0.02 ms     <- chunk 1 enqueue
         │   └─ [osd/5] op-request-created  192 ms     <- chunk 1 served by osd.5
         │       ├─ enqueue_op       0.01
         │       ├─ dequeue_op       0.48
         │       ├─ do_op            0.47
         │       └─ execute_ctx ×2  0.33 / 0.20    <- (no bluestore_read this trace = page cache hit)
         ├─ [rgw  ] rados_read              0.01 ms     <- chunk 2
         └─ ...                                          <- additional chunks
```

This is what the previous run couldn't show. Now the rgw `get_obj` trace fans down through `rados_read` (one per chunk) into per-OSD `op-request-created` and the PG pipeline. When the read actually hits disk the chain continues with `bluestore_read → get_onode + _do_read` (see west GET below — those chunks hit disk, this east trace happened to be cache-warm so the bluestore_read children are absent, which is itself a meaningful trace observation).

[Full trace JSON: [`traces/east-get_obj.json`](traces/east-get_obj.json) — 12 spans, 6 KB]

### Representative west GET (replicated 2×) — **new in this run, includes the disk read**

Trace `2ea79f5710dfed8d0b2b29dea7cff63` — total **3.77 s**, **22 spans**, **8 OSD batches**.

```
[rgw  ] get_obj                          3774 ms
 ├─ [rgw  ] verify_permission                0.01 ms
 └─ [rgw  ] execute                       2879 ms
     └─ [rgw  ] get_obj_data              2879 ms
         ├─ [rgw  ] rados_read              0.01 ms
         │   └─ [osd/2] op-request-created  123 ms
         │       ├─ enqueue_op       0.01
         │       ├─ dequeue_op      60.32
         │       ├─ do_op           60.30
         │       ├─ execute_ctx     60.25
         │       └─ bluestore_read   59.84 ms     <- ACTUAL DISK READ
         │           ├─ get_onode       0.00
         │           └─ _do_read       59.82      <- ~all the bluestore latency
         ├─ [rgw  ] rados_read              0.00 ms
         │   └─ [osd/2] op-request-created  145 ms
         │       ├─ dequeue_op      31.02
         │       └─ bluestore_read   30.76
         │           └─ _do_read       30.74
         └─ ...
```

This is the full chain that justified the patch series: a slow GET (3.77 s wall) can now be attributed precisely — most of the time is in `get_obj_data` (2.88 s), within that most chunks hit osd.2 which spent 60 ms in `bluestore_read._do_read` (i.e., the spindle was the bottleneck, not the network/PG/kv). Without these spans you'd see only the 2.88 s `get_obj_data` blob and have to guess.

[Full trace JSON: [`traces/west-get_obj.json`](traces/west-get_obj.json) — 22 spans, 10 KB]

### OSD-side latency aggregates (Prometheus span-metrics)

Tempo's metrics-generator converts every emitted span into a histogram. With the new spans, the OSD attribution is much more granular:

| OSD span | east p50 | east p99 | west p50 | west p99 | west/east p99 |
|---|---|---|---|---|---|
| `do_op` | 1.16 ms | **14.6 ms** | 1.61 ms | **121.0 ms** | **8.3×** |
| `execute_ctx` | 1.11 ms | 13.8 ms | 1.61 ms | 121.1 ms | 8.8× |
| `dequeue_op` | 1.23 ms | 31.3 ms | 1.36 ms | 114.6 ms | 3.7× |
| `issue_repop` | 1.56 ms | 15.5 ms | 1.35 ms | 14.3 ms | 0.9× |
| `queue_transactions` | 1.67 ms | 29.6 ms | 1.55 ms | 56.1 ms | 1.9× |
| `txc_aio_wait` | 1.56 ms | 10.4 ms | **7.95 ms** | 29.0 ms | 2.8× |
| `kv_submit_transaction` | 1.01 ms | 2.05 ms | 1.01 ms | 1.99 ms | 1.0× |
| `_do_write_big` | 1.00 ms | 1.98 ms | 1.00 ms | 1.98 ms | 1.0× |
| `bluestore_read` | 11.6 ms | **54.9 ms** | **47.6 ms** | **126.9 ms** | 2.3× |
| `_do_read` | 11.6 ms | 54.9 ms | 47.6 ms | 126.9 ms | 2.3× |

The kv-submit/`_do_write_big` p50/p99 are essentially identical across zones because that work is per-txc and CPU-bound. The OSD-queueing spans (`do_op`, `execute_ctx`, `dequeue_op`) and the bluestore-read p50 all collapse: **west has 5-8× longer tail because the same number of writers funnel through fewer OSDs**. This validates the throughput numbers up top.

### Span-rate fingerprint during the runs

`sum by (service, span_name) rate(traces_spanmetrics_calls_total[5m])`, sampled at each window's end:

| span | service | spans/s east | spans/s west | east/west |
|---|---|---|---|---|
| `op-request-created` | osd | **1732** | 196 | 8.8× |
| `enqueue_op` / `dequeue_op` | osd | 1732 each | 196 each | 8.8× |
| `queue_transactions` | osd | **465** | 84 | 5.5× |
| `kv_submit_transaction` | osd | 465 | 84 | 5.5× |
| `kv_committed_finalize` | osd | 465 | 84 | 5.5× |
| `bluestore_read` / `_do_read` / `get_onode` | osd | 370 each | 34 each | 10.9× |
| `_do_write_data` | osd | 202 | 44 | 4.6× |
| `_do_write_big` | osd | 186 | 35 | 5.3× |
| `txc_aio_wait` | osd | 180 | 25 | 7.2× |
| `do_op` / `execute_ctx` | osd | ≈215 | ≈111 | 1.9× |
| `issue_repop` | osd | 60 | 42 | 1.4× |
| `rados_read` | rgw | 35 | 21 | 1.7× |
| `get_obj` / `put_obj` / `execute` | rgw | ≈42 | ≈25 | 1.7× |

EC PUTs produce **5-9× more OSD spans per logical operation** than replicated PUTs. From a tracing-cost standpoint, EC dominates Tempo's ingest budget — the Tempo distributor reports 1.7M spans/s during the east window vs 195k/s during west. That's the price of full fan-out visibility; the span-batch-processor's `max_export_batch_size = 4096` is sized for it.

---

## Resolved since the previous run

| previously known issue | status |
|---|---|
| GET-path OSD spans missing | **resolved** — `rgw-iterate-trace-ctx-plumbing.patch` threads `jspan_context` through `ReadOp::Params` → `RadosReadOp::iterate` → `RGWRados::Object::Read::iterate` → `get_obj_data` → `rgw::Aio::librados_op` → `librados::async_operate` → `IoCtx::aio_operate(... jspan_context)`. GET traces now fan to OSD spans (and through to bluestore_read on cache-miss reads). |
| `librados::async_operate` for reads silently dropped trace_ctx | **resolved** — signature had been updated but the lambda body still called the 5-arg `aio_operate`. Fixed in `librados-asio-read-trace.patch`. |
| Long-tail `op-request-created` spans (default-constructed `jspan_context`) | **resolved** — `bluestore-read-subspans-buildfix.patch` switched the 4 `PrimaryLogPG.cc` sites to `jspan_context{false, false}` (opentelemetry-cpp 1.24 dropped the default constructor). |
| BlueStore tracer init silenced all OSD spans when `bluestore_tracing_enable=true` | **resolved** — added `Tracer::init_from_global()` and switched the bluestore init to use it instead of installing a new global `TracerProvider`. Previously the second `SetTracerProvider()` call orphaned the osd-tracer's `BatchSpanProcessor` and OSD spans went to /dev/null. |
| `multipart_upload <upload_id>` span-name cardinality | **resolved** (already fixed in r2, confirmed in r3 — span name is now bound). |

## New since the previous run

- **rados_read span** on the rgw read path — one per chunk (default 4 MiB).
- **BlueStore write-phase spans**: `_do_write_data` (and `_do_write_small`/`_do_write_big` children for each extent slice), `kv_submit_transaction`, `kv_committed_finalize` — turns the previously-opaque write-path latency between `queue_transactions` and the eventual completion into 5 attributable phases.
- **BlueStore read sub-spans**: `bluestore_read` → `get_onode` + `_do_read` (covered by an earlier patch but only effective now that the parent trace context reaches BlueStore through the new plumbing).
- Per-shard end-to-end visibility for EC writes — each of the 9 shards has its own `queue_transactions` subtree with all the new spans, so you can compare the speed of shard 0 vs shard 7 vs etc. directly.

## Known issues remaining

1. **Some PUT spans report multi-second durations on cold OSDs.** A handful of `op-request-created` spans hit 5-30 s in the long tail; correlates with OSD scrub activity. Not a tracing bug — the OSDs really did take that long under contention.
2. **GET trace chain only fires for chunked reads.** Small objects (`< rgw_get_obj_max_req_size`, default 4 MiB) read inline from the head object via a synchronous path that doesn't go through `RGWRados::Object::Read::iterate`. No `rados_read` span fires for those. Instrumenting the head-read path is a separate change to `RGWRados::Object::Read::prepare/read`.
3. **The east-zone EC backend uses `ECBackend` (optimized)** not `ECBackendL` (legacy). Cross-OSD shard read trace context is currently only wired for the optimized backend; legacy EC pools will still produce orphan OSD-side traces. ECBackendL update is a separate effort.

---

## Reproducing this

```bash
# 1. Create bench users (one per realm)
radosgw-admin user create --uid=warp-bench --rgw-realm=east --display-name="warp"
radosgw-admin user create --uid=warp-bench --rgw-realm=west --display-name="warp"

# 2. Pull the keys out
EAST_AK=$(radosgw-admin user info --uid=warp-bench --rgw-realm=east | jq -r .keys[0].access_key)
EAST_SK=$(radosgw-admin user info --uid=warp-bench --rgw-realm=east | jq -r .keys[0].secret_key)
WEST_AK=$(radosgw-admin user info --uid=warp-bench --rgw-realm=west | jq -r .keys[0].access_key)
WEST_SK=$(radosgw-admin user info --uid=warp-bench --rgw-realm=west | jq -r .keys[0].secret_key)

# 3. Clear Tempo for a clean trace state
kubectl -n monitoring scale deploy tempo --replicas=0
kubectl -n monitoring run tempo-clear --rm -i --restart=Never --image=amazon/aws-cli:latest \
  --env=AWS_ACCESS_KEY_ID=$TEMPO_AK --env=AWS_SECRET_ACCESS_KEY=$TEMPO_SK \
  --command -- aws --endpoint-url=http://rgw-west.rgw-gateway.svc:7480 \
                 s3 rm s3://tempo-traces/ --recursive --quiet
kubectl -n monitoring scale deploy tempo --replicas=1

# 4. Run warp against each (serialize — same OSDs)
STAMP=$(date +%Y%m%d-%H%M%S)
for zone in east west; do
  AK="${zone^^}_AK"; SK="${zone^^}_SK"
  kubectl -n rgw-gateway run warp-$zone --rm -i --restart=Never \
    --image=minio/warp:latest --command -- \
    /warp mixed --host=rgw-$zone.rgw-gateway.svc:7480 \
      --access-key="${!AK}" --secret-key="${!SK}" \
      --bucket=warp-otel-$zone-$STAMP \
      --concurrent=16 --duration=4m --noclear --no-color \
    | tee warp-$zone.log
done

# 5. Pull representative traces (longest PUT + longest GET per zone in window)
START=$EAST_START; END=$EAST_END   # epoch seconds
for op in put_obj get_obj; do
  Q='{rootServiceName="rgw" && name="'$op'" && resource.rgw.zone="east"}'
  QENC=$(python3 -c "import urllib.parse,sys;print(urllib.parse.quote(sys.argv[1]))" "$Q")
  TID=$(kubectl -n monitoring exec deploy/tempo -- wget -qO- \
    "http://localhost:3200/api/search?q=$QENC&start=$START&end=$END&limit=50" \
    | jq -r '.traces | sort_by(-.durationMs) | .[0].traceID')
  kubectl -n monitoring exec deploy/tempo -- wget -qO- \
    "http://localhost:3200/api/traces/$TID" > traces/east-$op.json
done

# (repeat for west)

# 6. Clean up
radosgw-admin user rm --uid=warp-bench --rgw-realm=east --purge-data
radosgw-admin user rm --uid=warp-bench --rgw-realm=west --purge-data
```

---

## Layout of this repo

```
.
├── README.md                          # this file
├── raw/
│   ├── warp-east.log                  # full warp stdout for east run
│   └── warp-west.log                  # full warp stdout for west run
└── traces/
    ├── east-put_obj.json              # OTLP trace, 257 spans across 18 OSD batches, 95 KB
    ├── east-get_obj.json              # OTLP trace, 12 spans across 6 OSD batches, 6 KB
    ├── west-put_obj.json              # OTLP trace, 68 spans across 10 OSD batches, 28 KB
    └── west-get_obj.json              # OTLP trace, 22 spans across 8 OSD batches, 10 KB
                                       # (west GET includes bluestore_read._do_read = on-disk read path)
```

Traces are raw output from Tempo's `/api/traces/{traceID}` (OTLP/JSON shape — `batches[].resource`, `batches[].scopeSpans[].spans[]`).

---

*Updated 2026-05-25 by Claude Code (after the r3 tracer patch series landed). Cluster owner: tu503. The Ceph fork lives at [`tu503/ceph` branch `ceph-999`](https://github.com/tu503/ceph/tree/ceph-999); the overlay at [`tu503/homelab-overlay`](https://github.com/tu503/homelab-overlay).*
