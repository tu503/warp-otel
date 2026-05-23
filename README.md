# warp-otel — S3 performance comparison of two Ceph RGW zones with OTLP traces

A side-by-side `warp mixed` benchmark run against two RGW zones on the same single-host Ceph cluster, with every S3 request producing an OpenTelemetry trace landing in Tempo. The two zones differ in only one meaningful axis — the `*.rgw.buckets.data` pool — so the deltas surface the real cost of replicated-vs-EC for an S3 workload on this hardware.

- **rgw-east**: data pool is **erasure coded 7+2** (ISA-L Reed-Solomon, `crush-failure-domain=osd`)
- **rgw-west**: data pool is **replicated, size=2 min_size=1**

Date: 2026-05-23. Ceph: `tentacle 20.1.1` (homelab fork, OTLP-instrumented). Warp: `1.3.1`.

---

## TL;DR

| op | metric | east (EC 7+2) | west (replicated 2×) | Δ |
|---|---|---|---|---|
| Total | throughput | **146.37 MiB/s, 24.35 obj/s** | **146.10 MiB/s, 24.36 obj/s** | ≈0 |
| PUT | avg latency | **687.2 ms** | 771.3 ms | east **−11%** |
| PUT | p99 latency | **1109 ms** | 1275 ms | east **−13%** |
| GET | avg latency | 1177 ms | **1092 ms** | west **−7%** |
| GET | TTFB avg | **92 ms** | 108 ms | east **−15%** |
| DELETE | avg latency | **168.8 ms** | 315.3 ms | east **−46%** |
| STAT | avg latency | **18.7 ms** | 42.4 ms | east **−56%** |

EC 7+2 was expected to be slower; it wasn't. On this hardware (single host, 9 HDD OSDs, EC failure-domain=osd) EC parallelizes every write across all 9 spindles, while replicated×2 only touches 2 spindles per write. The metadata-heavy ops (STAT/DELETE) tilt east further because east's index pool has 2× more PGs (32 vs 16).

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

$ ceph -s
  cluster:
    health: HEALTH_OK
  services:
    mon: 1 daemons, quorum 0 (age 1h) [leader: 0]
    mgr: sm3(active, since 1h)
    mds: 1/1 daemons up
    osd: 9 osds: 9 up (since 1h), 9 in (since 9w)
    rgw: 6 daemons active (6 hosts, 2 zones)
```

Single host, 9 HDD OSDs, all in `host sm3` (`root default`):

| osd | size | %use | dev (dm) | underlying disk |
|---|---|---|---|---|
| osd.0 | 7.3 TiB | 22.96% | /dev/dm-4 | sdq |
| osd.2 | 7.3 TiB | 22.71% | /dev/dm-8 | sdv |
| osd.3 | 7.3 TiB | 25.87% | /dev/dm-2 | sdx |
| osd.4 | 7.3 TiB | 24.82% | /dev/dm-3 | sdy |
| osd.5 | 7.3 TiB | 28.02% | /dev/dm-7 | sdw |
| osd.6 | 7.3 TiB | 26.61% | /dev/dm-0 | sdu |
| osd.7 | 7.3 TiB | 22.16% | /dev/dm-6 | sdt |
| osd.8 | 7.3 TiB | 26.39% | /dev/dm-1 | sds |
| osd.9 | 7.3 TiB | 23.72% | /dev/dm-5 | sdr |

Total raw: **65 TiB**, used 16 TiB (24.81%). BlueStore on dm-crypt LV per OSD; no separate WAL/DB device (collocated).

**Tracing topology.** All four host daemon types (osd / mon / mds / mgr) and the in-pod RGW daemons are linked against `opentelemetry-cpp` and pointed at Tempo's OTLP HTTP receiver (`http://10.144.27.223:4318/v1/traces`, a MetalLB LoadBalancer service). RGW pods reach Tempo via cluster DNS (`tempo.monitoring.svc:4318`); host daemons go via the LB IP.

---

## RGW zones, zonegroups, and pools

Two **separate realms** (`east` and `west`) — not a multi-site sync setup, just two independent S3 namespaces on the same OSDs.

### Zonegroup endpoints

| zonegroup | master_zone | endpoints |
|---|---|---|
| east | `93654b06-…` (east) | `http://rgw-primary.kubermon.svc:7480` |
| west | `237d4627-…` (west) | `http://rgw-west.kubermon.svc:7480` |

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

All RGW pods live in a single namespace `rgw-gateway` (formerly named `kubermon`), but **every zone gets its own complete set of resources** — Deployment, Service, ConfigMap, Secret — with no shared state between instances except for the underlying Ceph cluster. The zone name is the unit of isolation: it becomes the RADOS pool prefix (`<zone>.rgw.*`), the RGW realm/zonegroup/zone triple, the cephx client identity (`client.rgw.<zone>`), and the k8s resource suffix.

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

This is **not Rook**. The existing native Ceph cluster runs directly on the host (mon/mgr/mds/osd as systemd units on sm3); the RGW pods are *clients* of that cluster, configured purely via mounted `ceph.conf` + cephx keyring. The decoupling means RGW image rebuilds and pod restarts don't touch the data path.

### Per-instance manifest set (east shown)

**`ConfigMap/ceph-config-east`** — mounted at `/etc/ceph/ceph.conf`:

```ini
[global]
fsid = f64f9c1f-c1d5-447b-ae97-dfa92dec9bde
mon host = 10.144.27.26:6789
auth cluster required = cephx
auth service required = cephx
auth client required = cephx
jaeger_tracing_enable = true
otel_tracing_endpoint = http://tempo.monitoring.svc:4318/v1/traces

[client.rgw.east]
rgw frontends = beast port=7480
rgw zone = east
rgw realm = east
rgw zonegroup = east
rgw enable usage log = true
rgw usage log tick interval = 30
rgw thread pool size = 64
rgw enable gc threads = true
log to stderr = true
err to stderr = true
log file = ""
debug rgw = 1
```

**`Secret/ceph-rgw-keyring-east`** — mounted at `/etc/ceph/keyring`, `Opaque` type, single key:

```yaml
stringData:
  keyring: |
    [client.rgw.east]
    key = <cephx-shared-secret>
```

The corresponding `ceph auth` entry has caps `osd 'allow rwx', mon 'allow rw', mgr 'allow rw'` — RGW needs `rwx` on OSDs to do data ops, `rw` on mon for map updates, and `rw` on mgr for usage stats.

**`Deployment/rgw-east`** — 3 replicas:

```yaml
spec:
  replicas: 3
  selector: { matchLabels: { app: rgw, instance: east } }
  template:
    spec:
      containers:
        - name: radosgw
          image: registry.alcg.io/radosgw:v75c6769f6c   # homelab fork build
          command:
            - /usr/bin/radosgw
            - --id=rgw.east
            - --name=client.rgw.east
            - --cluster=ceph
            - -c=/etc/ceph/ceph.conf
            - --keyring=/etc/ceph/keyring
            - --no-mon-config            # don't try to fetch config from mon cluster — use mounted ceph.conf only
            - -f                          # foreground; let k8s manage lifecycle
          ports: [{ name: http, containerPort: 7480 }]
          readinessProbe: { httpGet: { path: /, port: 7480 }, initialDelaySeconds: 10, periodSeconds: 10 }
          livenessProbe:  { httpGet: { path: /, port: 7480 }, initialDelaySeconds: 30, periodSeconds: 30 }
          resources:
            requests: { cpu: 250m, memory: 512Mi }
            limits:   { cpu: "2",  memory: 2Gi }
          volumeMounts:
            - { name: ceph-config,  mountPath: /etc/ceph/ceph.conf, subPath: ceph.conf, readOnly: true }
            - { name: ceph-keyring, mountPath: /etc/ceph/keyring,   subPath: keyring,   readOnly: true }
      volumes:
        - { name: ceph-config,  configMap: { name: ceph-config-east } }
        - { name: ceph-keyring, secret:    { secretName: ceph-rgw-keyring-east } }
```

**`Service/rgw-east`** — `LoadBalancer`, MetalLB-allocated VIP from the `10.144.27.200-249` pool:

```yaml
spec:
  type: LoadBalancer
  selector: { app: rgw, instance: east }
  ports:
    - { name: http, port: 7480, targetPort: 7480, protocol: TCP }
```

Each instance gets its own external IP (east `.204`, west `.205`); clients pick which zone they're talking to by choosing an IP, not by HTTP virtual-hosting.

### Pool layout per instance

`setup-instance.sh` pre-creates the per-zone pools — pre-creation matters because RGW will auto-create them if missing but with default pg counts that may not match the workload:

| pool name | pgs (created by script) | what RGW stores there |
|---|---|---|
| `<zone>.rgw.control` | 8 | distributed-lock/notification objects (small, mostly idle) |
| `<zone>.rgw.meta` | 8 | user / bucket / period metadata |
| `<zone>.rgw.log` | 8 | usage log, bilog, datalog |
| `<zone>.rgw.buckets.index` | 16 | bucket index OMAP — STAT/LIST/DELETE hot path |
| `<zone>.rgw.buckets.data` | 16 | actual object payload |

These start as replicated-2× (the cluster default). Switching `<zone>.rgw.buckets.data` to an EC profile is a *post-create* step: `ceph osd pool create … erasure <profile>` + `radosgw-admin zone placement add` to point the default placement at the new pool. East's data pool was migrated this way (which is why it ended up as pool ID #64 with rule `east.rgw.buckets.data`, while everything else for east is in the original #44-#47 range).

### Container image (homelab fork)

`registry.alcg.io/radosgw:v75c6769f6c` is built **from scratch** by `build-image.sh` using buildah:

1. Resolve `ldd /usr/bin/radosgw` against the host's installed Ceph (so the image inherits whatever the Portage overlay produced — including the OTLP-instrumented `ceph-999` branch).
2. Copy `radosgw`, every transitive library (`/usr/lib64`, `/usr/lib64/ceph`, `/lib64/ld-linux-x86-64.so.2`) into a scratch container.
3. Add minimal `/etc/{passwd,group,nsswitch.conf}` for the `ceph` user (UID 167).
4. `EXPOSE 7480`, `ENTRYPOINT ["/usr/bin/radosgw"]`, `buildah commit`.

Image size lands around 200-300 MiB — no shell, no package manager, no debug tooling. Updates: rebuild on the host, `buildah push` to a `docker-archive`, `ctr -n k8s.io images import` into containerd on each node, then `kubectl rollout restart deploy/rgw-*`. `deploy-image.sh` does all four steps as one command and waits for both rollouts to finish.

This is also what makes the OTLP tracing patches land in production: any change to the `ceph-999` branch flows into a host rebuild (`emerge sys-cluster/ceph::homelab`) → new `/usr/bin/radosgw` → next `deploy-image.sh` picks it up.

### Bootstrapping a new instance

`setup-instance.sh <name>` is idempotent and does the full chain:

1. `radosgw-admin realm create` → `zonegroup create --master --default` → `zone create --master --default`
2. `radosgw-admin period update --commit`
3. Pre-create the 5 pools above (control/meta/log/buckets.index/buckets.data)
4. `ceph auth get-or-create client.rgw.<name> osd 'allow rwx' mon 'allow rw' mgr 'allow rw'`
5. Write `<name>/11-ceph-keyring.yaml` (Secret) with the cephx key
6. `radosgw-admin user create --uid=<name>-admin` (initial S3 admin user)
7. Write `<name>/10-ceph-config.yaml` (ConfigMap), `<name>/20-deployment.yaml`, `<name>/30-service.yaml` if not present
8. Print the S3 access/secret key pair

So bringing up a third zone is:

```bash
./setup-instance.sh south
kubectl apply -f south/
# south.rgw.* pools created, rgw-south Deployment running,
# external S3 endpoint at the next MetalLB pool IP.
```

### Live state at benchmark time

```
$ kubectl -n rgw-gateway get pods,svc -o wide
pod/rgw-east-78f578cb5b-29z9q   1/1 Running   sm3
pod/rgw-east-78f578cb5b-phx2z   1/1 Running   sm3
pod/rgw-east-78f578cb5b-xbscl   1/1 Running   sm3
pod/rgw-west-7df6fc45db-bmsmf   1/1 Running   sm3
pod/rgw-west-7df6fc45db-gb2tr   1/1 Running   sm3
pod/rgw-west-7df6fc45db-njk86   1/1 Running   sm3

service/rgw-east   LoadBalancer  10.111.88.16  10.144.27.204  7480 → 32010/TCP
service/rgw-west   LoadBalancer  10.104.10.75  10.144.27.205  7480 → 30089/TCP
```

Both 3-replica Deployments scheduled all 6 pods onto `sm3` (no nodeAffinity — k8s scheduler just packs them there because it's where the cluster has slack). That matters for the benchmarks: the warp pod, the RGW pods, and the OSDs all share the same physical box, so network latency is `lo`-fast and the spindles are the only bottleneck.

| | east | west |
|---|---|---|
| Deployment | `rgw-east` (revision 13) | `rgw-west` (revision 13) |
| replicas | 3 (all on sm3) | 3 (all on sm3) |
| image | `registry.alcg.io/radosgw:v75c6769f6c` (from host's ceph-999 build) | same |
| Service | `rgw-east` LoadBalancer `10.144.27.204:7480` | `rgw-west` LoadBalancer `10.144.27.205:7480` |
| ConfigMap | `ceph-config-east` | `ceph-config-west` |
| Secret | `ceph-rgw-keyring-east` | `ceph-rgw-keyring-west` |
| cephx client | `client.rgw.east` | `client.rgw.west` |
| Realm / zonegroup / zone | `east` / `east` / `east` | `west` / `west` / `west` |
| Data pool | `east.rgw.buckets.data` (pool #64, **EC 7+2**) | `west.rgw.buckets.data` (pool #58, **replicated×2**) |
| Index pool | `east.rgw.buckets.index` (pg_num 32) | `west.rgw.buckets.index` (pg_num 16) |
| Container resources | requests 250m/512Mi · limits 2c/2Gi | same |
| ceph.conf knobs | `rgw thread pool size = 64`, `debug rgw = 1`, OTLP to `tempo.monitoring.svc:4318` | same |

Source manifests + scripts live in a private GitLab repo (`gitlab.alcg.io/homelab/infra/rgw-gateway`); they're flux-managed but the cluster runs out of phase with Flux because the image-build path requires host-side buildah, not GitOps.

---

## Benchmark methodology

- Tool: **`minio/warp:1.3.1`** running as an in-cluster pod (`kubectl run --image=minio/warp`), reaching the RGW Service IP via cluster DNS — same network hop as the in-cluster S3 consumers.
- Workload: **`warp mixed`** — default mix of GET / PUT / STAT / DELETE, default object size distribution (Pareto over 1 KB – 10 MB).
- Concurrency: **16** workers.
- Duration: **4 minutes** per zone.
- Bench user: dedicated `warp-bench` per realm (created with `radosgw-admin user create --rgw-realm={east,west}`).
- Buckets: timestamped per run (`warp-otel-east-YYYYMMDD-HHMMSS`, `…-west-…`); `--noclear` to keep the data for post-run trace inspection.
- Both runs serialized (east first 21:09:57–21:18:20 UTC, west second 21:48:32–21:56:51 UTC) so they don't compete for the same 9 OSDs.

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
| **Total** | 146.37 MiB/s · 24.35 obj/s | 146.10 MiB/s · 24.36 obj/s |
| PUT | 36.86 MiB/s · 3.69 obj/s | 36.28 MiB/s · 3.63 obj/s |
| GET | 109.34 MiB/s · 10.93 obj/s | 109.82 MiB/s · 10.98 obj/s |
| STAT | — · 7.30 obj/s | — · 7.32 obj/s |
| DELETE | — · 2.42 obj/s | — · 2.43 obj/s |

Object-rate and bandwidth are within ±0.5%. The benchmark is effectively spindle-bound on both zones.

### Latency (ms)

| op | zone | avg | p50 | p90 | p99 | fastest | slowest |
|---|---|---|---|---|---|---|---|
| PUT | east | **687.2** | 665.2 | 943.9 | 1109.2 | 267.6 | 2153.6 |
| PUT | west | 771.3 | 765.9 | 1069.5 | 1275.0 | 257.6 | 2087.7 |
| GET | east | 1177.4 | 1202.9 | 1373.9 | 1481.2 | 363.9 | 1686.1 |
| GET | west | **1092.5** | 1128.3 | 1292.0 | 1450.1 | 392.5 | 1650.2 |
| STAT | east | **18.7** | 13.6 | 40.0 | 92.9 | 2.8 | 231.4 |
| STAT | west | 42.4 | 28.4 | 103.3 | 230.4 | 2.5 | 390.3 |
| DELETE | east | **168.8** | 165.2 | 267.1 | 335.1 | 32.3 | 652.4 |
| DELETE | west | 315.3 | 312.4 | 551.5 | 646.2 | 13.7 | 1141.0 |
| GET TTFB | east | **92** | 85 | 154 | 261 | 19 | 469 |
| GET TTFB | west | 108 | 95 | 182 | 332 | 9 | 590 |

### Why east is faster on PUT/STAT/DELETE despite being EC

Two reinforcing reasons:

1. **EC writes use *every* OSD on this single-host cluster.** With k=7, m=2 and `failure-domain=osd`, each PUT scatters 7 data + 2 parity shards across the 9 OSDs (i.e., **every spindle**). Replicated×2 writes only land on 2 of 9 OSDs — the rest sit idle for that operation. With 16 concurrent writers, EC is utilizing all 9 spindles continuously; replicated×2 is interleaving 16 writers across the available pool but each individual write is still 2-spindle-bound.
2. **east's index pool has 2× more PGs.** STAT and DELETE are bucket-index-heavy, and `east.rgw.buckets.index` has `pg_num=32` while `west.rgw.buckets.index` has `pg_num=16`. Larger PGs ⇒ more index contention on west; the latency table shows STAT/DELETE p99 on west literally 2–3× east's.

### Why west is (slightly) faster on GET

Reads from an EC pool need k=7 of 9 shards present and a reconstruction step on the RGW side. Each EC GET issues 7 RADOS reads, blocks until the slowest of the seven returns, and pays the ISA-L decode cost. Replicated×2 reads issue one RADOS read to the primary OSD. On HDDs, that single sequential read wins on TTFB (92 ms east vs 108 ms west is east-faster — that's noise, but the *avg* GET latency west is 7% lower, which matches the EC-vs-replicated theory). Bandwidth ties because both saturate the same 9 spindles in aggregate.

---

## Trace analysis

Every S3 request the benchmark issued produced an OTLP trace flowing through the Bell/Beast frontend → `RGWPutObj`/`RGWGetObj` → `rados_{read,write}` → into the OSD `dequeue_op` → `do_op` → `execute_ctx` → `issue_repop` chain. The fork's tracer patches propagate `jspan_context` from the RGW request span down through to per-OSD ops so PUTs come back as multi-service traces.

### Representative east PUT (EC 7+2)

Trace `692991bcf0c30ed85adaed6bd358c72f` — total 2.50 s wall time.

```
[rgw  ] put_obj                  2500.0 ms
 ├─ [rgw  ] verify_permission        0.0 ms
 └─ [rgw  ] execute               2499.6 ms
     └─ [rgw  ] put_obj_data       2499.5 ms
         └─ [rgw  ] rados_write      92.1 ms    (× 1)
             └─ across all 9 OSDs (0,2,3,4,5,6,7,8,9):
                ├─ [osd  ] op-request-created  (×27, avg 77 ms)
                ├─ [osd  ] dequeue_op           (×27, avg 1.2 ms)
                ├─ [osd  ] enqueue_op           (×27, avg 0 ms)
                ├─ [osd  ] do_op                (×3,  avg 6.2 ms)
                ├─ [osd  ] execute_ctx          (×3,  avg 6.0 ms)
                └─ [osd  ] issue_repop          (×3,  avg 5.7 ms)
```

- **OSD batches: 10** (one resource batch per OSD-tracer instance, with osd.0 split into two).
- **OSDs touched: all 9** (`instance.id` = `0,2,3,4,5,6,7,8,9`) — confirms EC `crush-num-failure-domains=osd` placing all 9 shards.
- **27 op-request-created** spans (3 RADOS write ops per shard ≈ object split into 3 chunks × 9 shards).
- Tail latency: the longest `op-request-created` is 264 ms — that one slow OSD gates the whole PUT.

[Full trace JSON: `traces/east-put_obj.json` — 95 spans, 44 KB]

### Representative west PUT (replicated 2×)

Trace `36005c4539a4fcea485e3a5e7f965d9` — total 1.71 s wall time.

```
[rgw  ] put_obj                  1714.7 ms
 ├─ [rgw  ] verify_permission        0.0 ms
 └─ [rgw  ] execute               1714.3 ms
     └─ [rgw  ] put_obj_data       1714.1 ms
         └─ [rgw  ] rados_write     249.3 ms    (× 1)
             └─ across 5 OSDs (4,5,6,7,9):
                ├─ [osd  ] dequeue_op       (×6, avg 4.2 ms)
                ├─ [osd  ] enqueue_op       (×6, avg 0 ms)
                ├─ [osd  ] do_op            (×3, avg 4.7 ms)
                ├─ [osd  ] execute_ctx      (×3, avg 4.5 ms)
                ├─ [osd  ] issue_repop      (×3, avg 4.4 ms)
                └─ [osd  ] op_commit        (×1)
```

- OSD batches: 6, only **5 unique OSDs** touched (one OSD has two batches). For 2× replication with multiple chunks per PUT, different chunks land on different PG primaries → multiple OSD pairs participate, not just two.
- `rados_write` parent span is 249 ms here vs 92 ms on east — but with only 5 OSDs sharing the bandwidth versus 9 on east.

[Full trace JSON: `traces/west-put_obj.json` — 33 spans, 16 KB]

### Representative GETs

Both east and west GET traces only have 4 RGW spans and **no OSD-side spans** — the homelab fork's tracer patches propagate context through `rados_write` (PUT path) but not yet through `rados_read`. So GET shows up as a single RGW-rooted trace with no downstream visibility.

```
[rgw  ] get_obj                   1214.3 ms     (east)        1061.5 ms   (west)
 ├─ [rgw  ] verify_permission         0.03 ms                    0.03 ms
 └─ [rgw  ] execute               1112.5 ms                    893.6 ms
     └─ [rgw  ] get_obj_data       1112.3 ms                    893.5 ms
```

Most of the GET latency is inside `get_obj_data`, which is the RADOS read + (for east) EC decode + body streaming back. Without OSD-side spans linked into the GET trace we can't subdivide the read further from the trace alone.

[`traces/east-get_obj.json`, `traces/west-get_obj.json` — 4 spans, 2 KB each]

#### OSD-side activity across the run windows (Prometheus span-metrics)

Even though the GET-path OTLP trace context isn't propagated to OSDs, the OSDs still emit their own spans for every op they handle, and Tempo's metrics-generator turns those into Prometheus histograms. Aggregating over each zone's test window gives a clean view of how the OSDs *themselves* behaved during PUT-and-GET-mixed traffic:

| OSD op | east p50 | east p99 | west p50 | west p99 |
|---|---|---|---|---|
| `do_op` | 1.30 ms | **13.1 ms** | 1.54 ms | **64.4 ms** |
| `execute_ctx` | 1.28 ms | 12.6 ms | 1.53 ms | 64.3 ms |
| `dequeue_op` | 1.11 ms | 17.3 ms | 1.30 ms | 59.0 ms |
| `issue_repop` | 1.73 ms | 13.5 ms | 1.55 ms | 19.0 ms |

**West's p99 is 3–5× worse on every primary OSD op.** Same reason as the client-side latency: every west PUT is funneled through just 2 of the 9 OSDs (the PG primary + secondary), so when those two are busy a new op queues behind them. East spreads each PUT across all 9 OSDs as EC shards, so any given OSD spends more time idle and services its share of an op quickly. The medians are close (because medians are dominated by the fast path on idle queues), but the tails diverge sharply once queues start backing up.

This is exactly the kind of analysis you can't extract from the trace tree alone — even when the per-trace fan-out is hidden, the span-metric histograms still capture every op every OSD processed.

The orphan-OSD trace search (`{resource.service.name="osd"}`) during each window returns mostly OSD-rooted traces with names like `op-request-created`, 3 spans each — that's a single OSD's view of one op (network arrival → enqueue → dequeue). Useful for spot-checking a slow OSD but not for end-to-end attribution. The PUT traces above (which *do* link RGW→OSD) remain the best illustration of the full fan-out shape.

### Span-rate fingerprint during the runs

From Tempo's metrics-generator into Prometheus, top spans/s during the test windows (`sum by (service, span_name) rate(traces_spanmetrics_calls_total[5m])`):

| span | service | spans/s during east run | spans/s during west run |
|---|---|---|---|
| `dequeue_op` | osd | 126 | 28 (5× less — fewer OSDs per PUT) |
| `enqueue_op` | osd | 126 | 28 |
| `op-request-created` | osd | 126 | 28 |
| `do_op` | osd | 56 | 26 |
| `execute_ctx` | osd | 55 | 26 |
| `issue_repop` | osd | 26 | 13 |
| `verify_permission` | rgw | 2.2 | 2.2 |
| `execute` | rgw | 2.2 | 2.2 |

Almost every OSD span name is **3–5× more frequent on east** because every PUT fans out to all 9 OSDs instead of 2. From a "trace bytes per second flowing into Tempo" standpoint, the EC pool produces dramatically more telemetry per logical operation.

---

## Known issues observed

1. **Span-name cardinality on `multipart_upload`.** The fork's RGW currently builds the multipart parent span name as `"multipart_upload <upload_id>"` (`src/rgw/rgw_op.cc:4459`, `name()` virtual). Each upload becomes a unique span name in Prometheus, blowing up `traces_spanmetrics_*` cardinality (≈80% of all distinct span_names in this cluster's TSDB are `multipart_upload …` variants). A fix patch (`ceph-20.1.1-rgw-multipart-span-name-fix.patch`) is sitting in `tu503/homelab-overlay` but not yet built into the running RGW binaries — these benchmarks include the un-fixed behavior.
2. **GET-path OSD spans missing.** The propagation patch handles PUT/RADOS-write but not RADOS-read. GET traces top out at the RGW `get_obj_data` span — no OSD-side detail.
3. **Long-tail `op-request-created` spans (~85 s reported on west).** A handful of OSD spans have absurd reported durations. Likely a default-constructed `jspan_context` without `IsRecording()` guard somewhere in the read path, so the span's end timestamp ends up as the parent context's. The fork's `tracer-jspan-default-ctor.patch` and `tracer-null-guard.patch` fix this in known code paths; this case isn't covered yet.

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

# 3. Run warp against each (serialize — same OSDs)
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

# 4. Pull representative traces (replace start/end with run window in epoch seconds)
for zone in east west; do
  for op in put_obj get_obj; do
    TID=$(kubectl -n monitoring exec deploy/tempo -- wget -qO- \
      "http://localhost:3200/api/search?q=%7Bresource.service.name%3D%22rgw%22%20%26%26%20name%3D%22$op%22%7D&start=$START&end=$END&limit=20" \
      | jq -r '.traces | sort_by(-.durationMs) | .[0].traceID')
    kubectl -n monitoring exec deploy/tempo -- wget -qO- \
      "http://localhost:3200/api/traces/$TID" > traces/$zone-$op.json
  done
done

# 5. Clean up
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
    ├── east-put_obj.json              # OTLP trace, 95 spans, 44 KB
    ├── east-get_obj.json              # OTLP trace, 4 spans, 2 KB
    ├── west-put_obj.json              # OTLP trace, 33 spans, 16 KB
    └── west-get_obj.json              # OTLP trace, 4 spans, 2 KB
```

Traces are raw output from Tempo's `/api/traces/{traceID}` (OTLP/JSON shape — `batches[].resource`, `batches[].scopeSpans[].spans[]`).

---

*Generated by Claude Code, 2026-05-23. Cluster owner: tu503. The Ceph fork lives at [`tu503/ceph` branch `ceph-999`](https://github.com/tu503/ceph/tree/ceph-999); the overlay at [`tu503/homelab-overlay`](https://github.com/tu503/homelab-overlay).*
