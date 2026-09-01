# RHOAI Resource Quota for Multi-Tenancy

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-25 |
| Scope          | Operator (RHOAI multi-tenancy resource allocation) |
| Status         | Draft |
| Authors        | [Lindani Phiri](@lphiri), [Chris Sams](@csams) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAIRFE-2922](https://redhat.atlassian.net/browse/RHAIRFE-2922), [RHAISTRAT-2554](https://redhat.atlassian.net/browse/RHAISTRAT-2554) |
| Other docs:    | operator/ODH-ADR-Operator-0015-multi-tenancy-framework.md; operator/multi-tenancy-strategy.md; Tenancy Control Plane design (multitenant-gpuaas: PLAN.md, API-REFERENCE.md, KUEUE-API-REFERENCE.md) |

## What

This ADR adds resource quota and fairness to the RHOAI multi-tenancy framework
defined in [ODH-ADR-Operator-0015](ODH-ADR-Operator-0015-multi-tenancy-framework.md).
That ADR established the tenant identity, hierarchy, delegation, isolation, and
observability substrate but deliberately deferred resource allocation. This ADR
fills that gap by layering an allocation model on the existing tenant tree.

Two complementary mechanisms are introduced:

* **Accelerator/compute quota via Kueue.** The tenant hierarchy maps onto Kueue:
  parent `PlatformTenant`s become Cohorts, leaf/standalone `PlatformTenant`s
  become ClusterQueues, and `TenantProject`s become LocalQueues. Quota,
  borrowing, lending, and preemption for GPU and other batch-admitted compute are
  enforced by Kueue at runtime; the control plane enforces the administrative
  allocation budget (capacity) at admission through the existing validating
  webhook.
* **Namespace guardrails via Kubernetes `ResourceQuota` and `LimitRange`.** Each
  `TenantProject` optionally generates a `ResourceQuota` (CPU, memory, storage,
  object counts) and a `LimitRange` on its namespace, covering the non-batch,
  directly-scheduled workloads that Kueue does not admit.

The two are additive and non-overlapping: Kueue governs what batch workloads may
*run* against a shared accelerator budget; `ResourceQuota` bounds what any
workload may *request* within a namespace. No existing 0015 CRD is reworked; this
ADR adds optional allocation fields that were intentionally left out of the
hierarchy CRDs.

## Why

[ODH-ADR-Operator-0015](ODH-ADR-Operator-0015-multi-tenancy-framework.md) landed
the tenant model without resource protection, an explicit and accepted scope
boundary. Its own Risks section records the consequence: "without quota, this
framework does not prevent noisy-neighbor contention." This ADR closes that gap.

The near-term driver is GPU-as-a-service (GPUaaS): customers want governed GPU
quota with credible consumption visibility, and they will not pay for a shared
platform they cannot trust to enforce fair allocation. GPUs are scarce and
expensive, so the platform must let organizations divide a fixed accelerator
budget across teams, reclaim idle capacity through borrowing, and preempt fairly
under contention. Kubernetes `ResourceQuota` alone cannot express borrowing,
lending, preemption, or gang scheduling, and it operates per namespace rather
than across an organizational tree.

Building a bespoke quota engine would duplicate capabilities Kueue already ships:
Cohort/ClusterQueue hierarchical quota, fair sharing, borrowing with limits, and
priority-based preemption. RHOAI already depends on Kueue for distributed
workloads. Reusing it keeps the platform aligned with a maintained upstream and
avoids reinventing admission control.

At the same time, not every workload is batch-admitted through Kueue.
Interactive workbenches, model servers, and ad-hoc pods schedule directly, so a
namespace still needs classic `ResourceQuota`/`LimitRange` guardrails to bound
total CPU/memory/storage and object counts. Both mechanisms are needed for
complete coverage.

## Goals

* Provide hierarchical accelerator/compute quota over the `PlatformTenant` tree
  by mapping it onto Kueue Cohorts, ClusterQueues, and LocalQueues.
* Enforce the delegation invariant at admission: the sum of a parent's
  children's allocations can never exceed the parent's capacity, per resource
  type.
* Delegate runtime quota, borrowing, lending, and preemption to Kueue rather than
  a bespoke engine.
* Offer named isolation presets for quota sharing (hard/guaranteed/balanced/
  burst) with a raw Kueue override escape hatch.
* Support borrowing between tenants that do not share a parent, via an opt-in
  shared pool decoupled from the admin hierarchy, without ever risking a lender's
  guaranteed quota.
* Generate per-`TenantProject` Kubernetes `ResourceQuota` and `LimitRange` for
  directly-scheduled workloads.
* Gate accelerator access with DRA DeviceClass restriction as a parent-constrained
  subset, paired with the compute quota it governs.
* Keep resource self-escalation impossible: capacity and hardware scope are
  parent-owned, consistent with the 0015 authorization model.
* Extend the 0015 CRDs additively (optional fields only) so existing tenants
  continue to operate unchanged.
* Feed per-tenant quota and utilization metrics into the 0015 observability
  tiers, including the filtering relay required for central (all-tenant) Kueue
  and DCGM metric sources.

## Non-Goals

* The tenant hierarchy, delegation, network isolation, observability substrate,
  and MaaS provisioning. Those are owned by
  [ODH-ADR-Operator-0015](ODH-ADR-Operator-0015-multi-tenancy-framework.md) and
  are prerequisites, not part of this decision.
* Multi-cluster quota federation. Single-cluster scope; cross-cluster fair
  sharing is deferred to a future MultiKueue-based design.
* Shipping or repackaging Kueue. Red Hat Build of Kueue (RHBoK) is a required
  dependency consumed as-is; this ADR does not change how Kueue is packaged or
  installed.
* Billing and chargeback systems. This framework enforces quota and emits the
  consumption metrics; FinOps tooling is downstream.
* A new quota API distinct from Kueue's. Raw overrides pass through to Kueue's
  own types rather than reinventing them.
* Cost-based or spot/preemptible scheduling policy. Preemption here is Kueue
  priority/fair-share preemption, not cloud cost optimization.

## How

### Relationship to ODH-ADR-Operator-0015

0015 kept the hierarchy CRDs free of resource-allocation fields precisely so this
ADR could add them without reworking them. This ADR therefore extends the
existing CRDs additively:

* `PlatformTenant.spec` gains optional `capacity` (parent budget), `quota` (leaf
  guaranteed allocation), and `hardware` (resource flavors, DeviceClasses).
* `TenantProfile.spec` gains optional `isolation` (preset) and `fairShareWeight`.
* `TenantProject.spec` gains optional `resourceLimits` (namespace
  `ResourceQuota`).

When these fields are absent, behavior is exactly as in 0015: a pure hierarchy
with no quota. When present, the controller additionally generates and reconciles
the Kueue and `ResourceQuota` resources described below. Enabling this ADR's
capability requires RHBoK to be present; without it the controller ignores the
quota fields and raises a warning condition rather than failing the tenant.

### Mapping the hierarchy onto Kueue

* Parent `PlatformTenant` -> Cohort (nested under the parent's Cohort).
* Leaf/standalone `PlatformTenant` -> ClusterQueue, with
  `nominalQuota = guaranteed`; a leaf's ClusterQueue joins its parent's Cohort, a
  standalone's ClusterQueue joins no Cohort.
* `TenantProject` -> Namespace + LocalQueue pointing at the tenant's
  ClusterQueue.

**Capacity** is a control-plane concept that does not exist in Kueue: the maximum
total allocation a parent allows its children to divide. It is enforced at
admission by the webhook (`sum(children allocations) <= parent capacity` per
resource type), not by Kueue quota. All generated Cohorts use `nominalQuota = 0`;
sharing happens through ClusterQueue-level lending of unused guaranteed quota.
This keeps capacity a clean per-resource-type API concept and avoids Kueue's
per-(flavor, resource) distribution problem.

### Isolation presets

Named presets on `TenantProfile.spec.isolation` configure the generated
ClusterQueue (leaf) or Cohort (parent) borrowing/lending limits and preemption
policy, with a raw Kueue override available on `TenantProfile.spec.overrides`:

* `hard` — fully isolated; no borrowing or lending.
* `guaranteed` — donates unused quota to siblings but never borrows.
* `balanced` — full sharing within the Cohort (Kueue default).
* `burst` — hoards its own quota but can borrow up to a cap for peak demand.

### Cross-tenant borrowing and shared pools

Kueue borrowing only happens inside a shared Cohort: two ClusterQueues can lend
and borrow each other's idle quota only when they sit in the same Cohort subtree.
So "borrowing across tenants" reduces to whether the tenants share a Cohort
ancestor, which yields three cases:

* **Siblings under one parent.** Already supported. The parent's Cohort is the
  borrowing pool, and the `balanced`/`burst` presets are the knobs that enable and
  cap it. This is the common case and needs nothing new.
* **Cousins (different parents, same ancestor).** With hierarchical Cohorts
  (v1beta2), borrowing propagates up to the lowest common ancestor's Cohort,
  bounded by the `borrowingLimit`/`lendingLimit` configured at each level.
* **Unrelated tenants (different roots or standalones).** No shared Cohort, so no
  borrowing by default. Enabling it requires the explicit shared-pool mechanism
  below.

For the unrelated case, borrowing is modeled as an explicit **shared pool** that
maps to a Kueue Cohort **decoupled from the admin parent/child tree**, the
resource-side analogue of 0015's `networkGrants`. A tenant opts its ClusterQueue
into a named pool; the pool's Cohort is where cross-branch borrowing happens.
Org structure (who administers whom) and sharing structure (who lends to whom)
stay separate concerns, so unrelated tenants can share a burst pool without being
reparented. Each member sets its own `lendingLimit` (how much idle quota it
donates) and `borrowingLimit` (how much it may take), and membership is opt-in
from each side so a tenant cannot unilaterally freeload on another's idle
accelerators.

Two invariants hold regardless of how a pool is formed:

* **Guaranteed quota is never at risk.** Kueue lends only *idle* capacity and
  reclaims it by preempting borrowed workloads when the owner submits work. A
  lender never loses its `guaranteed` allocation; the borrower must tolerate
  preemption (exactly what `burst` implies).
* **The capacity ceiling is unaffected.** Runtime borrowing temporarily exceeds a
  tenant's `guaranteed` quota but does not change administrative allocation, so
  the webhook invariant (`sum(children) <= parent capacity`) is orthogonal to
  borrowing.

### Namespace guardrails via ResourceQuota and LimitRange

For workloads that schedule directly rather than through Kueue admission, each
`TenantProject` optionally generates, on its namespace:

* a `ResourceQuota` from `spec.resourceLimits` (CPU, memory, ephemeral/persistent
  storage, and object counts such as pods, services, PVCs); and
* a `LimitRange` supplying default requests/limits so unbounded pods do not
  consume the whole quota.

These bound total namespace consumption independently of Kueue and require no
Kueue involvement, so they function even when a `TenantProject` runs no batch
workloads. The controller owns and reconciles these objects; direct edits are
protected by the same managed-resource protection as other generated resources.

### DeviceClass gating

When DRA is present, a validating webhook on `ResourceClaim` and
`ResourceClaimTemplate` restricts DeviceClass usage in a tenant's namespaces to
the tenant's `spec.hardware.deviceClasses`, which must be a subset of the
parent's. This ties physical accelerator access to the same parent-constrained
delegation as compute quota. Without DRA, DeviceClass gating is inactive and the
webhook passes such claims through.

### Enforcement layers

Delegation is protected by four layers:

1. **RBAC** grants tenant admins CRUD on `TenantProfile`/`TenantProject` and
   create-only on child `PlatformTenant`, but no access to Kueue CRs.
2. **The validating webhook** enforces the capacity ceiling and hardware-subset
   rules on every create/update, in addition to the 0015 authorization checks.
3. **Managed-resource protection** prevents direct edits to controller-owned
   Kueue resources, `ResourceQuota`s, and `LimitRange`s outside the tenancy API.
4. **Kueue runtime** enforces actual usage against the quotas the controller
   derived from the validated spec.

Capacity and hardware scope live on `PlatformTenant`, which is parent-owned in
0015, so a tenant admin cannot escalate their own quota by editing their
hierarchy node.

### Observability integration

This ADR reintroduces the central-metric handling that 0015 removed as
unnecessary for namespace-scoped metrics. Kueue controller metrics and the DCGM
GPU exporter run in platform namespaces and emit series for all tenants from one
endpoint, so a tenant's dedicated Prometheus cannot scrape them directly without
seeing other tenants' data. For tenants on observability tier 3, the controller
deploys an OTel Collector (filtering relay) that scrapes those shared endpoints,
filters to the subtree's ClusterQueue/Cohort names and namespaces, and
remote-writes the filtered stream into the tenant's Prometheus. Tier 2 dashboards
gain quota-utilization, borrowing/lending, and admission-latency panels sourced
from Kueue metrics (`kueue_cluster_queue_resource_usage`,
`kueue_cluster_queue_nominal_quota`, `kueue_admission_wait_time_seconds`) and GPU
metrics (`DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_FB_USED`).

### Example custom resources

**Parent `PlatformTenant`** (a root with a capacity budget and hardware catalog):

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: PlatformTenant
metadata:
  name: research-division
spec:
  displayName: "Research Division"
  capacity:
    resources:
      - type: nvidia.com/gpu
        amount: "64"
  hardware:
    resourceFlavors:
      - name: a100-80gb
    deviceClasses:
      - nvidia-a100
```

**Leaf `PlatformTenant`** (a quota holder; `guaranteed` must fit within the
parent's remaining capacity, enforced by the webhook):

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: PlatformTenant
metadata:
  name: nlp-team
spec:
  displayName: "NLP Team"
  parent: research-division      # immutable; creates the Cohort nesting
  quota:
    resources:
      - name: gpu
        type: nvidia.com/gpu
        guaranteed: "16"
        flavor: a100-80gb
  hardware:
    deviceClasses:
      - nvidia-a100              # must be a subset of the parent's
  sharing:
    pools:                       # opt into cross-tenant borrowing pools
      - name: gpu-burst-pool     # shared Kueue Cohort, decoupled from parent tree
        lendingLimit: "8"        # idle GPUs this tenant will donate
        borrowingLimit: "8"      # extra GPUs this tenant may take
```

**`TenantProject`** (namespace guardrails for directly-scheduled workloads):

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: TenantProject
metadata:
  name: sentiment-analysis
spec:
  tenant: nlp-team              # immutable
  resourceLimits:              # generates a namespace ResourceQuota + LimitRange
    requests.cpu: "32"
    requests.memory: 128Gi
    requests.nvidia.com/gpu: "8"
    count/pods: "50"
```

## Open Questions

* **Capacity vs. Kueue nominalQuota.** Confirm the "all Cohorts nominalQuota=0,
  share via ClusterQueue lending" model against real GPUaaS scenarios, including
  multi-flavor accelerators, before committing the webhook enforcement algorithm.
* **RHBoK version floor.** Establish the minimum RHBoK/Kueue version (Cohort
  v1beta2) and how it is guaranteed present, since it is provisioned out of band
  rather than via OLM dependency resolution.
* **ResourceQuota scope selector.** Whether generated `ResourceQuota`s should use
  scope selectors (e.g. to exclude best-effort or terminating pods) or apply flat
  namespace totals.
* **Interaction between Kueue admission and ResourceQuota.** A namespace with both
  a LocalQueue and a `ResourceQuota` is subject to both; confirm the guidance so
  batch workloads are not double-counted or unexpectedly blocked.
* **Preset defaults.** Confirm the borrowing/lending/preemption parameters behind
  each isolation preset with the Distributed Workloads team.
* **Shared-pool consent and ownership.** Confirm how a cross-tenant borrowing pool
  is created and who authorizes membership: two-sided opt-in from each member, a
  pool-owner role, or a common-ancestor admin. Also whether the pool is its own CR
  or a field on `TenantProfile`/`PlatformTenant`, and how a member is safely
  removed from a pool without evicting in-flight borrowed workloads
  ungracefully.
* **Adoption of existing ClusterQueues.** Whether and how to adopt hand-created
  Kueue ClusterQueues into the hierarchy, and the migration checklist to avoid
  workload eviction on a hard reconcile.

## Alternatives

**Kubernetes `ResourceQuota` alone (no Kueue).** Per-namespace hard caps are
simple and always available, but cannot express an organizational tree, cascading
capacity ceilings, borrowing of idle capacity, lending, or preemption. GPU
sharing degrades to static partitioning, which strands scarce accelerators. We
use `ResourceQuota` for namespace guardrails but rely on Kueue for accelerator
allocation and fairness.

**Bespoke quota and consumption engine.** Build allocation enforcement and fair
sharing directly in the controller (the strategy's status-rollup approach). This
duplicates Kueue's Cohort/ClusterQueue quota, borrowing, and fair-sharing, which
RHOAI already ships and maintains, and would carry its own scheduling-correctness
risk. We map to Kueue instead.

**Static per-tenant node pools / cluster partitioning.** Physically dedicate
nodes per tenant. Strong isolation, but wastes expensive accelerators when a
tenant is idle and cannot express soft sharing. Reserved for customers who
explicitly require hardware isolation, not the default model.

**Fold quota back into ODH-ADR-Operator-0015.** The earlier 0015 draft carried
this design inline. It was removed to keep the foundational tenant model small,
land it first, and avoid coupling tenant identity to a specific quota engine and a
required Kueue dependency. Keeping quota as a separate, layered ADR preserves that
separation and lets the tenant model ship independently.

**MultiKueue for cross-cluster fair sharing now.** Deferred. Single-cluster quota
must be proven before taking on cross-cluster admission and its added operational
surface.

## Security and Privacy Considerations

* **No self-escalation of quota.** `capacity`, `quota`, and `hardware` live on
  `PlatformTenant`, whose create/update requires an *ancestor's* admins per the
  0015 authorization model. A tenant admin cannot raise their own allocation;
  child creation is additionally checked via SubjectAccessReview against the
  parent.
* **Delegation invariant.** Four enforcement layers (RBAC, webhook capacity
  check, managed-resource protection, Kueue runtime) prevent a tenant admin from
  granting more resources than the parent allows.
* **Hardware access is parent-constrained.** DeviceClass gating restricts each
  tenant to a subset of the parent's accelerator classes, preventing access to
  hardware outside the delegated scope.
* **Cross-tenant borrowing is consent-based and non-destructive.** Shared-pool
  membership is opt-in per side, so a tenant cannot draw on another's idle quota
  without that tenant lending into the pool. Because Kueue lends only idle
  capacity and reclaims it by preemption, joining a pool never lets another tenant
  consume a member's guaranteed allocation, and it grants no namespace or data
  access across the pool.
* **Metric isolation for shared sources.** The tier 3 filtering relay ensures a
  tenant's dedicated Prometheus receives only its own subtree's Kueue/DCGM series,
  so central all-tenant metric endpoints do not leak cross-tenant consumption
  data.

## Risks

* **Kueue is a required dependency provisioned out of band.** Quota generation
  needs RHBoK (Cohort v1beta2) with no OLM dependency to guarantee presence or
  version. *Mitigation:* out-of-band provisioning installs a compatible RHBoK
  before the capability is enabled; a version guard surfaces
  `KueueVersionUnsupported` at reconcile time.
* **Adoption hard-reconcile can evict workloads.** Adopting an existing
  ClusterQueue overwrites its config to match the tenant spec. *Mitigation:*
  documented migration checklist; adopt with a matching spec, then adjust
  gradually.
* **Double-counting between Kueue and ResourceQuota.** A namespace subject to both
  admission mechanisms could unexpectedly block workloads. *Mitigation:* clear
  guidance on which workloads route through LocalQueues vs. direct scheduling;
  validation in the webhook where feasible.
* **Capacity validation cost at scale.** Reading siblings on every create/update
  stresses the webhook on large trees. *Mitigation:* informer-cache reads and
  label-based lookups; monitor webhook p99 against the 0015 scale targets.
* **Preset misconfiguration strands or starves resources.** Wrong
  borrowing/preemption tuning can hoard or starve GPUs. *Mitigation:* conservative
  preset defaults reviewed with DW; raw override reserved for advanced users.

## Stakeholder Impacts

| Group                                   | Key Contacts       | Date       | Impacted? |
| --------------------------------------- | ------------------ | ---------- | --------- |
| Platform / rhods-operator               | Platform team      | 2026-08-25 | Yes |
| Distributed Workloads / Kueue (RHBoK)   | DW team            | 2026-08-25 | Yes |
| Hardware / Accelerators (DRA, DCGM)     | Accelerator team   | 2026-08-25 | Yes |
| Observability (COO, OTel, Perses)       | Observability team | 2026-08-25 | Yes |
| Multi-Tenancy (ODH-ADR-Operator-0015)   | Platform team      | 2026-08-25 | Yes |
| DataScienceCluster / DSCI API           | Platform team      | 2026-08-25 | Yes |
| Security / Compliance                   | Security team      | 2026-08-25 | Yes |
| Dashboard (odh-dashboard)               | Dashboard team     | 2026-08-25 | Yes |
| Model Serving (kserve)                  | Model Serving team | 2026-08-25 | Maybe |
| Data Science Pipelines / MLflow / Feast | Component teams    | 2026-08-25 | Maybe |
| Documentation                           | Docs team          | 2026-08-25 | Yes |
| UX Design                               | UXD team           | 2026-08-25 | Yes |

Notes:

* **Platform / rhods-operator:** Extends the 0015 CRDs with allocation fields and
  the controller with Kueue/ResourceQuota generation and webhook capacity
  enforcement.
* **Distributed Workloads / Kueue:** Hard dependency. The framework generates
  Cohorts, ClusterQueues, and LocalQueues and requires RHBoK (Cohort v1beta2);
  preset semantics need DW review.
* **Hardware / Accelerators:** DRA DeviceClass gating and DCGM GPU metrics for
  enforcement and consumption.
* **Observability:** The tier 3 filtering relay and quota/GPU dashboards depend on
  this ADR's metric sources.
* **Multi-Tenancy (0015):** Foundational dependency; this ADR consumes its tenant
  identity, hierarchy, webhook, and observability tiers.
* **Security / Compliance:** Quota self-escalation prevention and hardware-scope
  delegation are enforced here.
* **Dashboard:** Quota allocation and utilization views; capacity-budget editing
  UI.
* **Model Serving / Component teams:** Workloads run within tenant quotas and
  namespace `ResourceQuota`s; must account for their footprint.

## References

* [ODH-ADR-Operator-0015](ODH-ADR-Operator-0015-multi-tenancy-framework.md) —
  RHOAI Multi-Tenancy Framework (tenant identity this ADR builds on)
* Multi-tenancy strategy: `operator/multi-tenancy-strategy.md`
* Tenancy Control Plane design (multitenant-gpuaas): `PLAN.md`,
  `API-REFERENCE.md`, `KUEUE-API-REFERENCE.md`
* [RHAIRFE-2922](https://redhat.atlassian.net/browse/RHAIRFE-2922) — Tenant
  Management Control Plane for Multi-Team RHOAI Deployments
* [RHAISTRAT-2554](https://redhat.atlassian.net/browse/RHAISTRAT-2554) — Org
  hierarchy and delegated administration
* Related ADRs: ODH-ADR-Operator-0007 (Auth CRD), ODH-ADR-Operator-0012 (Gateway
  API authentication)
* Red Hat Build of Kueue (RHBoK) / upstream Kueue (Cohort v1beta2)
* Competitive references: Run:ai Department/Project quota; upstream Kueue Cohort
  fair sharing

## Reviews

| Reviewed by | Date | Notes |
| ----------- | ---- | ----- |
|             |      |       |
