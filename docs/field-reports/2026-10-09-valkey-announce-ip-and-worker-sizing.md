# Field report: Valkey announced dead IPs, one worker stalled every actor, and how we size Substrate for customers

| | |
|---|---|
| Date | 2026-10-09 |
| Who | Planeo platform team (planeodev/planeo-infra) |
| Version | substrate chart **v0.0.9** (pinned by kagent 0.10.2), `ateom-gvisor:v0.0.9`, Valkey 8.0 |
| Cluster | EKS 1.36, us-east-1; Karpenter; Cilium (ENI, prefix delegation) |
| Scope of this branch | `release/v0.0.9` + one chart fix (`--cluster-announce-ip $(POD_IP)`) + this report |

Upstream note: the 0.0.x line replaced Valkey with PostgreSQL by **v0.0.30** (2026-09-15), and `main`
(v0.5 alphas) is PostgreSQL too. The fix here matters only to installs still pinned to a Valkey-based
chart (kagent 0.10.2 pins 0.0.9). The sizing half of the report applies to every version: it is about
workers, not the state store.

---

## 1. What happened (timeline, UTC)

| Time | Event |
|---|---|
| 2026-10-08 ~19:46 and ~23:07 | Karpenter moved four of the six `valkey-cluster` pods (0, 1, 2, 5) to other nodes. Each came back with a **new pod IP**. Nothing alerted; `CLUSTER INFO` stayed `cluster_state:ok`. |
| 2026-10-09 07:28:50 | `kubectl apply` a Declarative `SandboxAgent` (`substrate-hello`, kagent 0.10.2). |
| 07:28:52 | The golden actor booted under gVisor, passed readyz, was checkpointed, and the atelet **uploaded the snapshot to GCS within ~1 s** (`pages.img.zstd`, `fscheckpoint.json.zstd`, …). |
| 07:28:52 → | `ate-controller`: `while suspending golden actor: … workflow failed at step FinalizeSuspended: while getting worker for release: while getting worker key "worker:kagent:kagent-default:<pod>": dial tcp 10.0.15.39:6379: i/o timeout`. Retried with backoff. |
| 07:34:21 → | Retries now fail earlier: `CallAteletSuspend: … runsc fscheckpoint: exit status 128` — the sandbox was **already** checkpointed by the first attempt, so a second checkpoint is refused. A consequence, not a gVisor fault. |
| ~08:20 | Root cause found (below). Mitigated live with `valkey-cli config set cluster-announce-ip <podIP>` on the four stale nodes. |
| 08:28:28 | `kubectl delete sandboxagent` — the delete **hangs**: kagent's `kagent.dev/sandbox-agent-substrate-cleanup` finalizer tries to suspend → delete the half-suspended golden actor every 10 s and fails each time. |
| 08:33:31 | kagent gives up after `substrateDeleteTimeout` (5 min): *"substrate cleanup timed out; removing finalizer"*. The ActorTemplate (owned by the SandboxAgent) is garbage-collected. The golden actor is **orphaned** in atespace `ate-golden`. |
| 08:51:35 | Re-apply. New golden actor fails: `ResumeActor … AssignWorker: no free workers available`. The WorkerPool has **one** worker, and its Valkey record still says `"assignment":{"actor":{"atespace":"ate-golden","name":"<orphan>"}}`. |
| ~09:00 | `kubectl delete pod` the worker → the WorkerPool's Deployment recreates it → it registers free → annotate the ActorTemplate to skip the backoff → **Ready in seconds**, golden snapshot in GCS. |
| 09:04 | The permanent fix (this chart change, as a kustomize patch in our repo) rolled all six Valkey pods; most came back on new IPs and **every node announced its real IP**; the SandboxAgent stayed Ready throughout. |

Total time to Ready: ~90 minutes for a step that normally takes ~6 s. Every pod was `Running` the
whole time; there were no Kubernetes Events.

## 2. Root cause: Valkey cluster mode without an announce address

The chart starts Valkey with `valkey-server /etc/valkey/valkey.conf` and **no `cluster-announce-ip`**,
and `nodes.conf` (the node's own view of the cluster, including its own address) is on the PVC.

When a pod is rescheduled it gets a new IP, reloads `nodes.conf`, and keeps announcing the **old** IP
as `myself`. Its peers learn the new address over the cluster bus; the node itself never does. What
we saw on all six nodes (`valkey-cli cluster nodes`, the `myself` line vs `status.podIP`):

| Node | Announced itself as | Pod IP |
|---|---|---|
| valkey-cluster-0 (primary 0–5460) | 10.0.15.40 | 10.0.10.84 |
| valkey-cluster-1 | 10.0.7.60 | 10.0.8.178 |
| valkey-cluster-2 | 10.0.9.98 | 10.0.8.188 |
| valkey-cluster-5 (primary 5461–10922) | **10.0.15.39** | 10.0.10.95 |
| valkey-cluster-3, -4 | correct (never moved) | |

`ate-api-server`'s cluster client refreshed its slot map from a stale node, so the primary for slot
**6750** (`cluster keyslot worker:kagent:kagent-default:<pod>`) resolved to `10.0.15.39`, a dead
address.

**Fix (this branch):** `command: [valkey-server, /etc/valkey/valkey.conf, --cluster-announce-ip, $(POD_IP)]`,
with `POD_IP` from the downward API. Every start announces the current address; the bus gossips it.
A hostname-based alternative (`cluster-announce-hostname` + `cluster-preferred-endpoint-type hostname`
on the headless Service) also works and survives IP churn without a restart.

**Detection:** `cluster_state:ok` does **not** catch this. Compare each node's `myself` address with its
pod IP:

```bash
for i in 0 1 2 3 4 5; do
  echo "$i $(kubectl -n ate-system get pod valkey-cluster-$i -o jsonpath='{.status.podIP}') \
$(kubectl -n ate-system exec valkey-cluster-$i -c valkey -- valkey-cli cluster nodes | awk '/myself/{print $2}')"
done
```

## 3. What one worker taught us

The first incident was Valkey. The **outage** came from running one worker.

* **A worker holds exactly one actor.** `AssignWorkerStep` (`cmd/ateapi/internal/controlapi/workflow_resume.go`)
  sets a single `Worker.Assignment` on resume. Suspend releases it. Concurrent **running** actors
  (golden builds plus live sessions) therefore equal the number of free workers. Suspended actors
  hold no worker: they are snapshots in the bucket, so their count is bounded only by storage.
* **Nothing reclaims a worker held by a dead actor** in v0.0.9. The only path that releases a stale
  assignment is the *same* actor resuming with a changed pool. An actor that is half-suspended, or
  orphaned by a timed-out finalizer, keeps its worker until the worker **pod** is replaced.
* **With `replicas: 1`, one stuck actor is a full outage**: every golden build and every session for
  every agent fails with `no free workers available`. With N workers, it costs 1/N of capacity.
* **kagent hides the error.** `SandboxAgent` just says `ActorTemplateNotReady`. The real errors are
  only in the `ate-controller`, `ate-api-server` and `atelet` logs.
* **The delete path needs the same healthy state as the create path.** A stuck actor turns
  `kubectl delete` into a 5-minute wait, and an `apply` during that wait is a silent no-op
  (*"currently being deleted … unchanged"*).

## 4. The node we ran on, and what it can hold

Measured on the live node (2026-10-09):

| | |
|---|---|
| Instance | **c6a.large**, on-demand, single AZ (us-east-1c): 2 vCPU, 4 GiB |
| Allocatable | 1930m CPU, 3119 MiB memory, 110 pods |
| DaemonSet overhead on it (requests) | **~490m CPU, ~1032 MiB**: atelet, Cilium + cilium-envoy, kube-proxy, EBS/EFS CSI, Pod Identity agent, Tetragon, the OpenObserve collector, and one kube-state-metrics that landed there |
| Worker pod (WorkerPool `kagent-default`) | requests **250m / 512 MiB**, limit 2 GiB memory, no CPU limit |
| Idle worker usage | 1m CPU, 5 MiB (no actor) |
| atelet usage | 174 MiB |

**Packing by requests** (what the scheduler guarantees):

| Node | Allocatable (approx) | Free after DaemonSets | Workers by CPU | by memory | **Workers per node** |
|---|---|---|---|---|---|
| c6a.large (2 vCPU / 4 GiB) | 1930m / 3.0 GiB | 1440m / 2.0 GiB | 5 | 4 | **4** |
| c6a.xlarge (4 vCPU / 8 GiB) | ~3920m / ~7.0 GiB | ~3430m / ~6.0 GiB | 13 | 11 | **~11** |
| c6a.2xlarge (8 vCPU / 16 GiB) | ~7910m / ~14.9 GiB | ~7420m / ~13.9 GiB | 29 | 27 | **~27** |

The large row is measured; the xlarge and 2xlarge rows are computed from the same reservations. Check
them on the first node of that size.

**Burst is the open number.** Requests are 512 MiB but the limit is 2 GiB, and the node is already
overcommitted on limits (186 % memory). If running actors really use up to their limit, a c6a.large
fits **one** busy actor safely, not four. Before sizing for customers, **measure a running actor**:
`kubectl top pod` on the worker while a session is live, for each agent type (Go ADK, Python ADK,
BYO). Then set the worker `requests.memory` to the p95 and the limit to the p99. gVisor's Sentry adds its
own overhead on top of the app's RSS, so measure under `runsc`, not from a plain container.

**Throughput per worker** (sessions run one after another; kagent suspends after the response):

| Phase | Measured |
|---|---|
| Golden snapshot (first run on this cluster, 2026-09-30) | ~6 s from reconcile to Ready |
| Golden snapshot (this run, once Valkey was fixed) | snapshot uploaded ~1 s after boot |
| Session resume from snapshot (2026-09-30) | ~2.6 s |
| Suspend + upload | ~1 s |

With an average agent turn of **T** seconds, one worker serves about `3600 / (2.6 + T + 1)` sessions per
hour: **~150/h at T = 20 s**, **~56/h at T = 60 s**. The LLM call dominates T.

## 5. Should we start with 3? Yes, but the reason differs per layer

"Odd numbers, minimum 3" is a **quorum** rule. It applies to components that vote, not to stateless
executors. Taking Substrate layer by layer:

| Layer | Votes? | What we found | Recommendation |
|---|---|---|---|
| **Valkey cluster** (state) | **Yes**: a replica is promoted only with a majority of primaries. | 3 primaries + 3 replicas (correct count), but **two of the three primaries were on the same node**, in only 2 AZs, with no anti-affinity, topology spread or PodDisruptionBudget. Losing that node leaves 1 of 3 primaries: no majority, no failover, cluster down. | Keep 3 primaries, **one per AZ**. `podAntiAffinity` on hostname, `topologySpreadConstraints` on zone, a PDB (`maxUnavailable: 1`). Make sure each replica is in a different AZ from its primary. |
| **Workers / gVisor nodes** (compute) | No: stateless; actor state is in the snapshot. | 1 node, 1 AZ, 1 worker. One orphaned actor stopped the platform. | **3 nodes, one per AZ**, at least 1 worker each. You survive an AZ loss at 2/3 capacity, and a stuck actor costs a third, not everything. Scale workers for concurrency, not quorum. |
| **ate-api-server, atenet-router** (control plane) | No (stateless over the store) | 1 replica each | ≥ 2 across AZs for a customer service. |
| **ate-controller** | Leader election | 1 replica | 2 (one active, one standby). |

**Eventual consistency.** Valkey cluster replication is **asynchronous**. A write acknowledged by a
primary that fails before replicating is lost when its replica is promoted. Substrate's workflows are
idempotent steps with `IsComplete` checks, so a lost *status* is usually re-derived. A lost or stale
**worker assignment**, though, is exactly the orphan class above, and nothing in v0.0.9 sweeps it.
Two consequences:

1. Any Valkey-based install needs an operational sweep: workers whose assigned actor no longer exists,
   or has not progressed for N minutes, get their pod recycled.
2. The upstream move to **PostgreSQL** (a single primary with transactional commits, plus a synchronous
   standby if configured) is the stronger consistency model for this state. Plan to follow it as soon
   as the kagent pin allows.

## 6. What it costs (AWS us-east-1, Linux, list prices, 2026-10-09)

| Instance | On-demand $/h | 1-yr reserved $/h | Spot $/h |
|---|---|---|---|
| c6a.large (2 vCPU / 4 GiB) | 0.077 | 0.051 | ~0.030 |
| c6a.xlarge (4 vCPU / 8 GiB) | 0.153 | 0.101 | n/a |
| m6a.xlarge (4 vCPU / 16 GiB) | 0.173 | 0.114 | n/a |

Source: instances.vantage.sh (AWS public price list). Spot is listed for completeness only. Our pool
is on-demand because an interrupted node kills its running actors mid-turn.

**Worker-pool options** (730 h/month; compute only, excluding the LLM, snapshot storage/egress and the
control plane, which runs on the general pool and costs tens of MiB):

| Option | Nodes | Concurrent actors (by requests) | $/month on-demand | $/month 1-yr reserved | $ per concurrent slot / month |
|---|---|---|---|---|---|
| Today | 1 × c6a.large, 1 AZ | 1 configured (4 possible) | **$56** | $37 | $56 (as configured) |
| **Recommended start** | **3 × c6a.large, 3 AZs** | **3 → 12** | **$169** | **$112** | ~$14 at 12 |
| Growth | 3 × c6a.xlarge, 3 AZs | ~33 | $335 | $221 | ~$10 |
| Larger | 3 × c6a.2xlarge, 3 AZs | ~80 | $670 | ~$442 | ~$8 |

**Per 1,000 sessions** (T = 20 s, ~150 sessions/h/worker): the recommended start, fully used (12
workers), serves ~1,800 sessions/h for $0.231/h, about **$0.13 per 1,000 sessions** of sandbox compute.
At a realistic 30 % utilization it is ~$0.43 per 1,000. LLM tokens will dominate either way.

**Two costs still to measure:**

* **Snapshot traffic.** Our snapshots live in GCS while the cluster is on AWS, so every resume reads
  the snapshot across clouds and pays egress. Record the snapshot size per agent type (the `pages.img`
  object in the bucket) and multiply by resumes per month.
* **Real RSS per actor** (section 4). If actors need ~1.5 GiB, the per-node numbers drop by about 3×
  and the cost per slot rises by the same factor.

## 7. Recommendations

1. **Merge the chart fix** (this branch) on any Valkey-based install. Our platform carries it as a
   kustomize patch until our pin moves.
2. **Start customer service with 3 gVisor nodes, one per AZ**, and **≥ 3 workers** (one per node).
   Then raise workers for concurrency using measured actor RSS. Raise the Karpenter NodePool `limits`
   to match: our `cpu: 8` cap stops at 2 × xlarge.
3. **Spread Valkey**: hostname anti-affinity, zone spread, a PDB, one primary per AZ.
4. **Run ≥ 2 replicas** of ate-api-server and atenet-router; 2 of ate-controller.
5. **Alert on** `no free workers available`, on `FinalizeSuspended` errors, and on any Valkey node
   whose `myself` address differs from its pod IP.
6. **Sweep orphaned worker assignments** (a worker assigned to an actor that is gone or has not
   progressed for N minutes, so recycle the worker pod) until upstream adds a reaper or we move to
   the PostgreSQL line.
7. **Measure before quoting customers**: actor RSS under `runsc` per agent type, snapshot size, and
   p50/p95 of resume and suspend under concurrency.
