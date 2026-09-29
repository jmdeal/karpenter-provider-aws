# Offerings: Exposing Instance Configuration

**Audience:** contributors adding a new instance configuration option to `karpenter-provider-aws`.

**Status:** This document is **guidance, not law.** It captures the reasoning we want new designs to
engage with, and the defaults we want them to start from. A design that deviates isn't wrong — it
owes an explanation. Maintainers have the final say on every feature interface.

Karpenter has not consistently followed this guidance historically. Treat this document as the
source of truth for new work, not the existing code.

---

## 1. The InstanceType model

Karpenter's scheduler reasons about two layers.

An **InstanceType** describes a family of possible nodes: its name, resource capacity, overhead, and
a set of **requirements** — the labels a node of this type could carry.

An **Offering** is a concrete launch configuration of that instance type: a specific zone, capacity
type, reservation, and so on, along with a price and an availability signal.

```
InstanceType: m5.large
├── Requirements                          # common to every offering; multi-valued = a choice
│   ├── kubernetes.io/arch                : [amd64]
│   ├── topology.kubernetes.io/zone       : [us-west-2a, us-west-2b]
│   ├── karpenter.sh/capacity-type        : [on-demand, spot, reserved]
│   ├── karpenter.k8s.aws/instance-tenancy: [default, dedicated]    <- instance-type-level choice
│   └── karpenter.k8s.aws/capacity-reservation-id: [cr-abc, cr-def] <- union of all offering values
├── Capacity / Overhead                   # per instance type; an offering may override
└── Offerings                             # concrete launch configurations
    ├── {zone: us-west-2a, capacity-type: on-demand}                  $0.096  available
    ├── {zone: us-west-2a, capacity-type: spot}                       $0.031  available
    ├── {zone: us-west-2b, capacity-type: on-demand}                  $0.096  unavailable (ICE)
    ├── {zone: us-west-2b, capacity-type: spot}                       $0.031  available
    ├── {zone: us-west-2a, capacity-type: reserved, ...id: cr-abc}    ~$0     available
    └── {zone: us-west-2b, capacity-type: reserved, ...id: cr-def}    ~$0     available
```

This is a hierarchy: **properties shared by every offering live on the instance type, and only the
differences are encoded on offerings.** If you treat an offering as a single launch configuration,
then the full set of launch configurations Karpenter can produce is the **cross product** of the
instance type's multi-valued requirements and its offerings. The six offerings above, crossed with
two tenancy values, describe twelve distinct launch configurations.

That cross product is sometimes materialized literally. Partition placement groups expand every
offering into one offering per partition, because the scheduler needs each partition to exist as a
topology domain:

```go
// pkg/providers/instancetype/offering/placement_group_resolver.go (illustrative)
for _, offering := range offerings {
    for partition := 1; partition <= partitionCount; partition++ {
        reqs := scheduling.NewRequirements(offering.Requirements.Values()...)
        reqs.Add(scheduling.NewRequirement(v1.LabelPlacementGroupPartition, corev1.NodeSelectorOpIn, fmt.Sprintf("%d", partition)))
        expanded = append(expanded, &cloudprovider.Offering{Requirements: reqs, /* ... */})
    }
}
```

### 1.1 Invariants

These hold regardless of which surface you choose. A design that breaks one of them will either
silently fail to schedule or launch nodes that don't match what the scheduler simulated.

**The union rule.** The scheduler filters instance types *before* it looks at offerings: an instance
type is discarded if its requirements don't intersect the pod's, and only the survivors have their
offerings checked. An offering-level requirement key must therefore appear on the instance type with
the **union** of every value any of its offerings carries. An offering the instance type doesn't
advertise is unreachable.

**Absence is a value.** Every well-known label must be defined on the instance type's requirements,
even when it has no values — use `DoesNotExist`. The same applies to offerings: an on-demand offering
declares `DoesNotExist` for the capacity reservation keys so that it stays compatible with a pod
requiring those labels to be absent. Omitting the key means "no constraint", which is not the same
statement.

**Unavailable, not absent.** `GetInstanceTypes` always returns every instance type. An option that
makes an instance type unusable — an insufficient-capacity signal, a zonal shift, a NodeClass the
instance type can't satisfy — is expressed by setting `Available: false` on the affected offerings,
not by dropping the instance type. Dropping it degrades scheduling error messages and breaks
consolidation's view of the world.

**Launch candidates are re-derived from the NodeClaim.** The NodeClaim's requirements are the whole
contract between the scheduler and the cloud provider. The offering the scheduler settled on during
simulation is not communicated, and `Create` must not take a dependency on the scheduler's
implementation — which constraints it chose to write, or how it narrowed them. `Create` derives the
compatible instance types and offerings itself from `nodeClaim.Spec.Requirements` and builds its
launch candidates from that set.

Two consequences. First, the compatible set is usually larger than one offering, and that's
useful — every compatible (instance type, zone) pair becomes a `CreateFleet` override, so several
offerings coexist in a single launch and EC2 chooses among them. Second, some dimensions can't
coexist in one launch request and have to be collapsed to a single value: a fleet request has one
tenancy and one capacity type. Which applies is a **per-feature decision**, and a design needs to
state it: can compatible values coexist as alternatives in one launch, or must one be selected? If
selected, what happens when several values are compatible, or none is constrained?

**Labels must round-trip.** Whatever `Create` resolves has to come back as a NodeClaim label, so the
node is labeled with what was actually launched. If a value can't be resolved and reported, it
shouldn't be a label.

---

## 2. Independent vs. dependent options

**Independent** options can be set regardless of the rest of the launch configuration. Instance
tenancy is independent: whatever the zone, capacity type, or placement group, tenancy can be chosen
freely. An independent option is a full cross product, so it belongs on the instance type as a
multi-valued requirement — no offering needs to be created for it.

**Dependent** options are only valid in combination with specific other parameters. Capacity
reservations are zonal: a reservation exists in exactly one zone, for one instance type. The valid
launch configurations form a sparse matrix, and offerings are the cells:

```
m5.large                us-west-2a    us-west-2b    us-west-2c
  on-demand                 ✓             ✓             ✓
  spot                      ✓             ✓             ✓
  reserved / cr-abc         ✓             ·             ·        <- ODCR in us-west-2a
  reserved / cr-def         ·             ✓             ·        <- capacity block in us-west-2b

InstanceType requirements (union of all rows/columns):
  zone                      : [us-west-2a, us-west-2b, us-west-2c]
  capacity-type             : [on-demand, spot, reserved]
  capacity-reservation-id   : [cr-abc, cr-def]

Offerings (the ✓ cells only):    6 offerings, not 12
```

The instance type advertises everything that is possible *somewhere*; the offerings say which
combinations are actually possible. A pod that selects `cr-abc` and `us-west-2b` passes the instance
type filter and then finds no compatible offering — which is the correct outcome, reported as an
unsatisfiable scheduling constraint rather than a failed launch.

---

## 3. Choosing a surface

Three shapes, plus the combination of two of them.

### 3.1 Requirement values on the instance type

*Example: instance tenancy.*

The instance type advertises every value it supports:

```go
// pkg/providers/instancetype/types.go — computeRequirements
scheduling.NewRequirement(v1.LabelInstanceTenancy, corev1.NodeSelectorOpIn,
    string(ec2types.TenancyDefault), string(ec2types.TenancyDedicated)),
```

By default nothing narrows that set, and the choice is driven by the pods:

```yaml
# This workload needs a dedicated instance
apiVersion: v1
kind: Pod
spec:
  nodeSelector:
    karpenter.k8s.aws/instance-tenancy: dedicated
```

An administrator can *choose* to constrain the launch options for a NodePool, which takes the choice
away from the pods that schedule to it:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
spec:
  template:
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-tenancy
          operator: In
          values: ["dedicated"]
```

At launch, `Create` derives the compatible values from the NodeClaim. Tenancy is a select-one
dimension — a fleet request carries a single tenancy — so it needs a documented rule. Its rule is a
preference order with a default:

```go
// pkg/providers/instance/instance.go — getTenancyType (illustrative)
// If the requirement is unset, or allows both values, prefer `default`.
for _, tenancy := range []string{string(ec2types.TenancyDefault), string(ec2types.TenancyDedicated)} {
    if requirement.Has(tenancy) {
        return tenancy
    }
}
```

The resolved value is then echoed back as a label (`labels[v1.LabelInstanceTenancy] = i.Tenancy`), so
the node reflects what was launched.

Use this shape when the option is a small, closed set of values and the choice is genuinely
per-workload.

### 3.2 Distinct offerings

*Examples: capacity reservations, placement group partitions.*

Use offerings when the option is **dependent** on other launch parameters, or when it changes an
offering's price, availability, or advertised resources. Offerings are the only layer that can carry
those differences:

```go
// pkg/providers/instancetype/offering/reserved_capacity_resolver.go (illustrative)
offering := &cloudprovider.Offering{
    Requirements: scheduling.NewRequirements(
        scheduling.NewRequirement(karpv1.CapacityTypeLabelKey, corev1.NodeSelectorOpIn, karpv1.CapacityTypeReserved),
        scheduling.NewRequirement(corev1.LabelTopologyZone, corev1.NodeSelectorOpIn, reservation.AvailabilityZone),
        scheduling.NewRequirement(cloudprovider.ReservationIDLabel, corev1.NodeSelectorOpIn, reservation.ID),
        // ...
    ),
    Price:               price,
    Available:           /* compatible && has capacity && not expiring && not zonal-shifted */,
    ReservationCapacity: reservationCapacity,
}
```

New offerings are added by implementing an `OfferingResolver` and registering it with the offering
provider. Resolvers run in order, each receiving the offerings produced by the previous step, so a
resolver can either append new cells (reserved capacity) or expand existing ones (placement group
partitions).

Offerings can also diverge in what they advertise, via `CapacityOverride` and `OverheadOverride`.
That makes them the only place to model **mutually exclusive views of the same hardware.** The
expected shape for GPU device configuration is a label per driver whose values name the mode —
starting with something like `device-plugin` and `dra` — backed by a fan-out resolver (the same shape
as the placement group resolver in §1) that produces one offering per mode, each advertising the
resources that mode exposes. A multi-valued requirement on the instance type cannot express this: the
modes disagree about capacity, and capacity only varies at the offering layer.

### 3.3 NodeClass configuration

*Examples: `blockDeviceMappings`, `networkInterfaces`, `cpuOptions`, `connectionTracking`.*

Every configuration option is a dimension you could vary a node along. Some are low cardinality
(tenancy: two values). Some are effectively infinite (block device mappings: an arbitrary list of
devices, sizes, volume types, and encryption settings). Enumerating offerings for a high-cardinality
dimension is not viable, and neither is enumerating label values.

NodeClass configuration collapses the dimension instead: for a given `EC2NodeClass`, the option has
exactly one value, and every instance type returned for that NodeClass is locked to it. Multiple
configurations in one cluster are expressed as multiple NodeClasses, and workloads select between
them through a NodePool or a NodeClass label.

Where the NodeClass configuration makes some instance types unusable, express that through a
compatibility check, which flows into offering availability rather than removing the instance type:

```go
// pkg/providers/instancetype/compatibility/compatibility.go (illustrative)
func (c nestedVirtualizationCheck) compatibleCheck(info ec2types.InstanceTypeInfo) bool {
    if c.cpuOptions == nil || lo.FromPtr(c.cpuOptions.NestedVirtualization) != "enabled" {
        return true
    }
    return info.ProcessorInfo != nil && lo.Contains(
        info.ProcessorInfo.SupportedFeatures,
        ec2types.SupportedAdditionalProcessorFeatureNestedVirtualization,
    )
}
```

A capability label can sit alongside NodeClass configuration: `karpenter.k8s.aws/instance-hypervisor`
lets workloads select nitro instance types, while `connectionTracking` — which requires a nitro
hypervisor — is configured on the NodeClass. Keep the distinction clear. A capability label is a
**fact** about an instance type: one value per type, derived from `DescribeInstanceTypes`, and it
selects instance types rather than launch configurations. A label with several values on one instance
type is a **choice**, and belongs in §3.1 or §3.2.

### 3.4 Combined: NodeClass constrains, labels select

There is no existing example of the fully-combined pattern, but it's the recommended way to add
administrator control to an option that is already label-driven, without a breaking change.
Hypothetically, for tenancy:

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
spec:
  # default | dedicated | dynamic; nil is equivalent to dynamic
  instanceTenancy: dedicated
```

| `spec.instanceTenancy` | Advertised values for `karpenter.k8s.aws/instance-tenancy` | Effect |
|---|---|---|
| `nil` (default) | `[default, dedicated]` | Today's behavior, unchanged |
| `dynamic` | `[default, dedicated]` | Explicitly opts into per-pod selection |
| `default` | `[default]` | Pods requesting `dedicated` become unschedulable on this NodeClass |
| `dedicated` | `[dedicated]` | All nodes from this NodeClass are dedicated |

The NodeClass narrows the set of values the instance type advertises; pods select within whatever
remains. Narrowing the advertised set is what makes the conflict surface in the right place: a pod
requiring `dedicated` against a NodeClass that advertises only `default` finds no compatible instance
type and **stays unschedulable** — no NodeClaim is created and then fails to launch.

Because the field's nil value preserves the existing advertised set, adding it is non-breaking. Note
the direction of travel: **NodeClass field first, labels later, is a compatible evolution; labels
first, then narrowing them, is not.**

### 3.5 Rules of thumb

- **Labels** are appropriate when the configuration is **user-driven**, or when per-pod dynamism is
  required rather than per-NodePool/NodeClass. Users drive them through node affinity. Cluster
  administrators *can* restrict affinity with a validating admission policy, but labels should be
  treated as user-facing configuration by default.
- **NodeClass configuration** is appropriate when the configuration space is too high cardinality to
  enumerate as label values, or when the option must be locked down to the cluster administrator.
- **When in doubt, choose NodeClass configuration.** It minimizes the exposed configuration surface,
  it is self-documenting through the CRD (labels are documented only indirectly, through CEL
  validation on the NodePool and NodeClaim CRDs and through the website), and it leaves an
  upgrade path to labels open.
- **Use distinct offerings whenever the option is dependent on other launch parameters, or changes
  price, availability, or advertised resources.** Prefer them over encoding the same information
  implicitly, even when an implicit encoding would be cheaper.
- **Don't surface one dimension with two competing authorities.** If an option appears on both the
  NodeClass and as a label, the NodeClass must constrain and the label must select within that
  constraint (§3.4).

### 3.6 Discouraged: signalling through pod resource requests

An older pattern encodes dynamic configuration as an extended resource: every offering advertises
the resource in memory, pods request it, and the provider infers the configuration at launch from
`nodeClaim.Spec.Resources`. Dynamic EFA works this way (`vpc.amazonaws.com/efa`).

**Not recommended for new features.** It has real performance benefits — it avoids multiplying the
offering count — but it hides the configuration from the scheduler and gives users no selector for
it. A planned refactor will add a post-processing layer that makes distinct offerings and these
"implied offerings" functionally identical, so new work should be modeled as distinct offerings; that
is also the only model that can express mutually exclusive resources (§3.2).

---

## 4. Wiring checklist

The touchpoints a new option generally has to hit. Not every option touches every one.

| Step | Where | Notes |
|---|---|---|
| 1. API surface | `pkg/apis/v1/ec2nodeclass.go` | NodeClass field, CEL validation, defaulting. New fields are hashed by default; `hash:"ignore"` opts out, which means a change to the field won't drift existing nodes. |
| 2. Label registration | `pkg/apis/v1/labels.go` | Add to `WellKnownLabels`; add to `karpv1.WellKnownValuesForRequirements` for a closed value set so NodePool/NodeClaim CEL validation rejects typos. |
| 3. Instance type requirements | `computeRequirements` in `pkg/providers/instancetype/types.go` | Advertise the union of all offering values, or `DoesNotExist`. |
| 4. Offerings | new `OfferingResolver` in `pkg/providers/instancetype/offering/`, registered on the provider | **Only needed when the option applies to a subset of offerings** — i.e. a dependent option. An independent option like instance tenancy never appears in offerings. Set `Requirements`, `Price`, `Available`, and any capacity/overhead override; resolvers must be deterministic and cheap. |
| 5. Offering cache key | `newCacheKeyBuilder` in `base_resolver.go` | **Any NodeClass field that affects offerings must be in the cache key**, or stale offerings will be served across NodeClasses. |
| 6. Instance type compatibility | `pkg/providers/instancetype/compatibility/` | For NodeClass configuration that makes some instance types unusable. |
| 7. Launch resolution | `pkg/providers/instance/`, `pkg/providers/launchtemplate/` | The deterministic decision rule for unconstrained/multi-valued requirements, plus the EC2 API plumbing. |
| 8. Label round-trip | `pkg/cloudprovider/cloudprovider.go` | Resolved value set on the NodeClaim so `List`/`Get`/registration agree with `Create`. |
| 9. Drift | NodeClass hash and/or `IsDrifted` | Decide explicitly whether a change to this option should replace existing nodes. |
| 10. Docs | `website/content/en/preview/` | NodeClass field reference and/or the labels list; a task page if the feature needs a walkthrough. |

Two cost considerations worth stating in any design:

- **Offering count is multiplicative.** Offerings are recomputed per NodeClass across every instance
  type, zone, and capacity type. An expansion that multiplies the offering count (like partition
  placement groups) should only apply when the feature is actually configured, and should be feature
  gated where it's new.
- **Cache correctness beats cache hit rate.** A missing cache key component is a correctness bug; an
  extra one only costs recomputation.

---

## 5. Evaluation criteria

Use this to review a design (or an implementation) against this guidance. Each item is a check, with
what a passing answer looks like. Items are **advisory**: a design may fail one and still be the
right call, if it says why. Flag the gap and the reasoning, don't block on the letter of the rule.

### Surface selection

- **S1 — A surface is named and justified.** The design states whether the option is exposed as
  instance-type requirement values, distinct offerings, NodeClass configuration, or a combination,
  and says why. *Fails if the choice is implicit.*
- **S2 — Cardinality is addressed.** The design states how many values the dimension can take. A
  high-cardinality or open-ended dimension exposed as label values needs an explicit justification.
- **S3 — Audience matches the surface.** Label-driven options are justified by per-pod or user-driven
  need; options that must be administrator-controlled are on the NodeClass.
- **S4 — Dependency is classified.** The design says whether the option is independent of the rest of
  the launch configuration or dependent on it. Dependent options are modeled as distinct offerings,
  not as instance-type requirement values.
- **S5 — Divergent price, availability, or resources implies offerings.** If the option changes any
  of those, it is modeled as distinct offerings.
- **S6 — Single authority per dimension.** If the option appears on both the NodeClass and as a
  label, the NodeClass constrains and the label selects within that constraint; the precedence is
  documented.

### Model correctness

- **M1 — Union rule.** Every offering-level requirement key is advertised on the instance type with
  the union of all values its offerings carry. *Failure mode: instance types filtered out before
  offerings are consulted; the feature appears to do nothing.*
- **M2 — Absence declared.** Keys with no values are declared `DoesNotExist` on both the instance
  type and on offerings that don't carry them. *Failure mode: pods requiring the label be absent
  match offerings they shouldn't, or vice versa.*
- **M3 — Unavailable, not absent.** Instance types are never dropped from `GetInstanceTypes` to
  express unavailability; `Available: false` is used instead.
- **M4 — Coexist or select, stated.** For each new key, the design says whether multiple compatible
  values can coexist as alternatives in a single launch request (like zones) or whether one must be
  selected (like tenancy). If selected, the rule for "several compatible" and "unconstrained" is
  documented user-facing behavior, not an implementation accident.
- **M5 — No dependence on scheduler internals.** `Create` derives its launch candidates from the
  NodeClaim's requirements, not from an assumption about which constraints the scheduler writes or
  how it narrowed them.
- **M6 — Round-trip.** The launched value is set as a NodeClaim label, and `Create`, `Get`, and
  `List` agree on it.
- **M7 — Infeasibility surfaces in scheduling.** A configuration that can't be launched leaves pods
  unschedulable, with no compatible instance type or offering. *Fails if a NodeClaim is created and
  then errors at launch.*

### Lifecycle and cost

- **L1 — Cache key completeness.** Any NodeClass field that affects offerings is part of the offering
  cache key.
- **L2 — Drift decided.** The design states whether changing the option drifts existing nodes, and
  the NodeClass hash reflects that decision.
- **L3 — Offering growth bounded.** If the change multiplies offering count, it applies only when
  configured, and the expected magnitude is stated.
- **L4 — Compatibility.** Newly added fields default to today's behavior. Narrowing an
  already-advertised set of label values is called out as a breaking change.

### Discouraged patterns

- **D1 — No new resource-request signalling.** New dynamic configuration is not inferred from
  extended resources in `nodeClaim.Spec.Resources` (§3.6), unless the design explains why offerings
  can't work. Exceptions may be permitted while offering refactor is pending.

### Documentation and tests

- **T1 — Discoverability.** New NodeClass fields have godoc/CRD descriptions; new labels are added to
  the website's label reference and, where the value set is closed, to
  `WellKnownValuesForRequirements`.
- **T2 — Negative coverage.** Tests cover the unavailable/incompatible path and the unconstrained
  launch-time default, not just the happy path.
