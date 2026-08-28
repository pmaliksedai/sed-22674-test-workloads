# SED-22674 test workloads — inflated HPA targets + externally scaled workloads

Manifests for exercising two `sedai-core` commits on a real GKE cluster, with a focus on the
**availability** action path.

| Commit | Title |
|---|---|
| `f761f26` | Honor HPA targets above 100% with a request/limit decision tree (#19161) |
| `7d97fab` | Allow horizontal scaling for externally scaled workloads (#19191) |

Both already have KWOK/BDD coverage in core
(`KubeInflatedHpaTargetAvailability.feature`, `KubeInflatedHpaTargetProfiling.feature`,
`KubeServiceTest`). These manifests are the real-cluster equivalent: instead of a metric profile
declaring "CPU is at 85% of the limit", the pods actually burn CPU and hold memory.

## What each scenario proves

| File | Scenario | Guardrail branch / gate | Expected post-fix result |
|---|---|---|---|
| `01` | CPU saturation, HPA target 170%, limit movable | node **[2]**, limit grows | `SET_KUBERNETES_CPU_LIMIT` / `KUBERNETES_CPU_SATURATION`, with `request × 1.70 × 1.05 ≤ limit` |
| `02` | Memory saturation, HPA target 170% | node **[2]**, memory path | memory recommendation with the same inequality |
| `03` | **Control** — HPA target 70% | node [2] skipped (`isAboveBasis` false) | unchanged from pre-fix behaviour |
| `04` | CPU saturation + `honorCpuRequestLimitRatio` | node **[1]** | deployed 2.0 limit/request ratio held — availability never reached this setting before |
| `05` | CPU saturation, limit injected by a LimitRange | node **[2]**, request gives | request lowered to `500 / (1.70 × 1.05) = 280m` |
| `06` | Argo Rollout, `spec.replicas` unset (1 pod) | `getEffectiveReplicaCount` tier 2 | workload is actionable; no "Resource Doesn't have any Replicas" |
| `07` | Argo Rollout via `workloadRef` (2 pods) | same, plus profiling eligibility | as above, and profiling is reachable |
| `08` | **Control** — `replicas: 0` | all three tiers return 0 | still excluded, `ZERO_REPLICAS` blocker intact |

## Why the numbers are what they are

**Saturation threshold.** `kubernetesCPUSaturationPercent` / `kubernetesMemorySaturationPercent` are
`0.75`, evaluated as `max_over_time(max_usage / limit)` across a 90-minute window
(`kubernetesCPUSaturationCheckWindowLengthInMins: 90`, `KubernetesTimeSeriesHelpers`). A single-threaded
busy loop is throttled to exactly the CPU limit, so the CPU workloads sit at `1.0` — well clear.
The memory workload holds 430Mi of a 512Mi limit = `0.84`.

**Why `250m / 500m` with `averageUtilization: 170`.** An HPA compares usage against the *request* only,
so `170% × (250m / 500m)` encodes "scale out at 85% of the limit". That is the geodis shape, and it is
what the KWOK fixture `test-inflated-hpa-target-deployment.yaml` uses. Requests stay above `100m`
because `k8sMinCPURequestInCores` is `0.1` in both prod and aggressive config.

**Why `maxReplicas: 3`.** With usage pinned at the limit the CPU workloads sit at 200% of request, above
the 170% target, so their HPAs will scale out. The cap bounds what this costs you. Memory (`02`) sits at
168%, just under its target, so it stays at 2 replicas.

## Deploy

```bash
# Argo Rollouts controller — only needed for 06 and 07
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

kubectl apply -f 00-namespace.yaml
kubectl apply -f .
```

Minimal subset if the cluster is small — `01` (the core fix) and `06` (the replica fix):

```bash
kubectl apply -f 00-namespace.yaml -f 01-cpu-saturation-inflated-hpa.yaml -f 06-externally-scaled-rollout.yaml
```

**Capacity.** Everything applied is roughly **6 vCPU and 4 GiB of actual burn** at steady state
(five CPU scenarios × up to 3 pods × 500m, plus the memory holder). Budget a node pool that can take
that on top of the Sedai agent — e.g. 3 × `e2-standard-4`.

**Use a Standard GKE cluster, not Autopilot.** Autopilot rewrites requests/limits and constrains
`LimitRange`, which breaks scenario `05` and can quietly change the request/limit spread the other
scenarios depend on.

## Verify the preconditions before trusting a result

```bash
# 05 — the limit must be injected, not declared
kubectl -n sedai-inflated-hpa-frozen get deploy frozen-limit-cpu-sat \
  -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'      # requests only
kubectl -n sedai-inflated-hpa-frozen get pod -l app=frozen-limit-cpu-sat \
  -o jsonpath='{.items[0].spec.containers[0].resources.limits.cpu}{"\n"}' # 500m

# 06 / 07 — spec.replicas must be EMPTY. This is the entire premise of the PR-1 test.
kubectl -n sedai-inflated-hpa get rollout externally-scaled-rollout    -o jsonpath='{.spec.replicas}{"\n"}'
kubectl -n sedai-inflated-hpa get rollout externally-scaled-workloadref -o jsonpath='{.spec.replicas}{"\n"}'

# CPU is actually pinned at the limit
kubectl -n sedai-inflated-hpa top pod
```

## What to look for in core logs

Every guardrail node logs its own decision — at `info` when it changed the pair, `debug` when it did not.

```
# 01 — limit raised to cover the scale-out point
"the HPA scales out at .*above the .* limit, so it could never fire. This change already raises the limit"

# 04 — ratio held (node 1)
"holding the deployed limit/request ratio of 2.00"

# 05 — request lowered instead (node 2, frozen limit)
"The limit is frozen for this container, so lowering the request to"

# 05 variant with averageUtilization 450+ — the conflict warning
"keeping the HPA target reachable needs a request of .* below the configured minimum"

# 06 / 07 — replica count resolved from running pods (tier 3 only)
"reports no spec/status replicas but has .* discovered pods; treating it as externally scaled"

# 08 must still say this; 06 and 07 must not
"Resource Doesn't have any Replicas"
```

## Timing

`settings.sedai.io/metricsEvaluation.minPeriodInHours: "1"` on every workload shortens the
metrics-evaluation warm-up. Allow roughly:

- ~15 min for discovery and topology refresh
- ~1–2 h before the saturation window has enough data
- then `MONITOR_KUBERNETES_CONTAINER_SATURATION_MINION` produces the availability recommendation

`availability.configMode: AUTO` executes the remediation. Switch it to `CO_PILOT` if you want the
recommendation surfaced for approval instead of applied.

## Caveats

- **Scenario `06`/`07` depends on argo-rollouts leaving `spec.replicas` nil.** Some versions write it
  back. The precondition check above is not optional — if it prints a number, that scenario is testing
  nothing.
- **Tier 3 of `getEffectiveReplicaCount` (discovered pods) is hard to reach on a real cluster**, because
  a Rollout with running pods normally reports `status.availableReplicas`. Tier 2 is what these
  manifests exercise; tier 3 is covered by the unit tests in `KubeServiceTest`. To force tier 3, add a
  readiness probe that never passes — pods run, `availableReplicas`/`readyReplicas` stay 0.
- **The `honorCpuRequestLimitRatio` annotation key in `04` is by convention** (`enableVerticalScaling.*`,
  matching the `doNotDownsizeCpuLimit` key used elsewhere). If your build does not pick it up, set the
  setting on the resource in the Sedai UI instead.
