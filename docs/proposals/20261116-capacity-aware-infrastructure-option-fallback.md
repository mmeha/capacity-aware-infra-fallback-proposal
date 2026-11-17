---
title: Capacity-aware infrastructure option fallback for Machines
authors:
  - "@TODO"
reviewers:
  - "@TODO"
creation-date: 2026-11-16
last-updated: 2026-11-16
status: provisional
see-also:
  - "/docs/proposals/20240916-improve-status-in-CAPI-resources.md"
  - "/docs/proposals/20221003-In-place-propagation-of-Kubernetes-objects-only-changes.md"
---

# Capacity-aware infrastructure option fallback for Machines

## Table of Contents

- [Glossary](#glossary)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals/Future Work](#non-goalsfuture-work)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Requirements](#requirements)
    - [Functional Requirements](#functional-requirements)
    - [Non-Functional Requirements](#non-functional-requirements)
  - [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Alternatives](#alternatives)
- [Upgrade Strategy](#upgrade-strategy)

## Glossary

This proposal introduces:

- **Infrastructure preference:** the ordered set of infrastructure options for a `Machine` — the existing `infrastructureRef` as the most-preferred option, followed by `infrastructureFallbackPreference` in order. Each option is an ordinary infrastructure template reference (as `infrastructureRef` is today), considered as one ranked candidate.
- **Capacity-available condition:** a standardized, pre-provision condition an infrastructure provider sets to report that it cannot satisfy the currently selected option.
- **Advance:** moving a `Machine` from its current infrastructure option to the next one in the preference, either by mutating the infrastructure object in place or by replacing it.

## Summary

Today a `Machine` binds to a single `infrastructureRef`, so if the infrastructure provider has no capacity for that configuration (for example, a specific instance type), the `Machine` is stuck until capacity returns or an operator intervenes.

This proposal lets a `Machine` carry an **ordered list of fallback infrastructure options** alongside its existing `infrastructureRef` and fall back to the next option when the provider reports — **before provisioning anything** — that the current one has no capacity, provisioning on the first option that has capacity. It standardizes three small, opt-in additions: an ordered `infrastructureFallbackPreference` on the owning resource, a pre-provision capacity condition in the infrastructure contract, and a provider-agnostic advance (in-place mutation or replacement).

The initial target is **control-plane providers**, where `Machine`s are provisioned one at a time and capacity stalls are most painful. The contract is defined at the `Machine` level so it can extend to `MachineDeployment`/`MachineSet` and `MachinePool` later; that is future work.

## Motivation

Operators of scarce, heterogeneous capacity want to express a ranked set of acceptable configurations and have the platform provision the best option currently available, without manual intervention and without churning real infrastructure to discover shortages.

Cluster API has no contract for any part of this: no way to express an ordered preference, no standard pre-allocation "no capacity" signal, and no standard way to move a `Machine` to the next option. Each consumer therefore builds it provider-specific. NVIDIA Kubernetes Engine (NVKE) did so for the NICo provider — preference on a provider-specific payload, fallback keyed off provider-specific condition reasons, mutating a provider-specific field — which works, but only for that provider.

### Goals

- Define a standard, ordered **infrastructure fallback preference** for a `Machine` (primary `infrastructureRef` plus ordered fallbacks), targeting **control-plane providers** first.
- Define a standard, **pre-provision capacity condition** in the infrastructure-machine contract, distinguishing "no capacity" from a transient check failure.
- Define a provider-agnostic **advance** behavior that works via in-place mutation **or** replacement, chosen by a declared provider capability.
- Guarantee **churn-free discovery**: no backing infrastructure is created or destroyed to learn that an option has no capacity.
- Guarantee **forward progress**: a still-pending `Machine` cycles past unavailable options and provisions the first option with capacity; while all options are unavailable it keeps re-probing the preferred one, so a freed-up preferred option is taken as soon as nothing else is available.
- Preserve full backward compatibility: a `Machine` with a single `infrastructureRef` behaves exactly as today.

### Non-Goals/Future Work

- A general scheduler, bin-packer, or cost optimizer across options. This is strictly ordered fallback evaluated per `Machine`.
- Wiring the preference into `MachineDeployment`/`MachineSet` and `MachinePool` owners. The contract is designed to extend to them, but the initial scope is control-plane providers; worker-side adoption is future work.

## Proposal

### User Stories

#### Story 1

As an operator of a cluster with scarce capacity, I declare an ordered set of acceptable instance types for my **control plane**. When the most-preferred type is exhausted, a new control-plane `Machine` is provisioned on the first acceptable type that has capacity, automatically and without leaving failed instances behind.

#### Story 2

As an infrastructure-provider author, I opt my provider into capacity-aware fallback by surfacing one standard condition before I provision, and (optionally) by accepting an in-place spec update. My provider learns nothing about preference policy; the core/owning controller drives the fallback.

### Requirements

#### Functional Requirements

##### FR1 (MUST)
A **control-plane provider** may declare an ordered `infrastructureFallbackPreference` on its machine template, alongside the existing `infrastructureRef` (the most-preferred option). When set, the owning controller materializes each control-plane `Machine`'s infrastructure object from the selected option and records the selection on the `Machine`'s status.

##### FR2 (MUST)
An infrastructure provider that supports this feature MUST evaluate capacity for the selected option **before allocating any backing resource** and, when it cannot satisfy it, set a standard `CapacityAvailable=False` condition with a standard reason, having allocated nothing.

##### FR3 (MUST)
The owning controller MUST advance to the next option on an "advance" reason, MUST retry the same option on a "retry-same" reason, and MUST provision the first option that reports capacity as it cycles. While every option is unavailable and the `Machine` is still pending, it MUST keep re-probing cyclically so a freed-up more-preferred option is picked up; it does not move an already-provisioned `Machine`, nor a pending one off an available less-preferred option, onto a more-preferred option that frees up later.

##### FR4 (SHOULD)
A provider may declare `inPlaceCapacityRotation`; the owning controller always performs the advance, and this flag only selects the mechanism. When true, the controller mutates the infrastructure object's spec in place; when false/unset, the controller deletes the (unprovisioned) object and recreates it from the next option.

#### Non-Functional Requirements

##### NFR1 (MUST — churn-free)
No backing infrastructure resource is created or destroyed to discover that an option lacks capacity. The only cost of a rejected option is an unfulfilled spec and a condition.

##### NFR2 (SHOULD — observability)
The capacity state should be expressible as a standard condition/metric so operators can distinguish, fleet-wide: still cycling through options; every option temporarily out of capacity (held and retried automatically); and every option invalid (`ConfigurationNotFound`), which cannot recover on its own and needs operator action. The controller keeps retrying in all but the last case, so this is a severity signal, not a "given up" state.

### Implementation Details/Notes/Constraints

Three additions; all opt-in; absence preserves today's behavior.

**1. Ordered fallback options (owner level).** Placement policy belongs to the owning resource, not the provider — initially a **control-plane provider's machine template**, with the field defined generically so other owners can honor it later. The design is strictly additive: `infrastructureRef` keeps its current meaning as the most-preferred option (index 0); a new optional, ordered `infrastructureFallbackPreference` lists the rest:

```yaml
# control-plane provider machine template (initial scope)
spec:
  machineTemplate:
    spec:
      infrastructureRef: { ... }            # existing; unchanged; the most-preferred option (index 0)
      infrastructureFallbackPreference:     # new, optional, ordered; index 1..n, most-preferred first
        - { apiGroup: infrastructure.cluster.x-k8s.io, kind: FooMachineTemplate, name: option-1 }
        - { apiGroup: infrastructure.cluster.x-k8s.io, kind: FooMachineTemplate, name: option-2 }
```

Absent or empty, the `Machine` behaves exactly as today (single `infrastructureRef`, no fallback). Entries are templates of the same infrastructure kind as `infrastructureRef` in the first iteration. The owning controller materializes the selected option, keeps the `Machine`'s own `infrastructureRef` populated with it for downstream compatibility, and records the selection:

```yaml
# Machine
status:
  placement:
    selectedIndex: 0
    totalOptions: 3
    selectedRef: { apiGroup: infrastructure.cluster.x-k8s.io, kind: FooMachineTemplate, name: option-0 }
```

`selectedRef` names the option directly, so operators and tooling read the selection without cross-referencing the preference list, and the record stays meaningful if the list is later reordered (the index alone would not).

Each option is an opaque template, so fallback is over whole infrastructure *configurations*, not over any named field. Whatever differs between options — instance type, subnet, VPC prefix, availability zone, or a combination — is the provider's business; the owning controller never reads a provider-typed field. Multi-dimensional preference (e.g. "try each subnet for an instance type, then the next instance type") is expressed as the flattened cross-product in list order, keeping combinatorial placement policy out of the core controller.

The list stays inside `machineTemplate` next to `infrastructureRef` because a fallback option is not a standalone machine spec: it overrides only `infrastructureRef` and inherits the rest of `machineTemplate` — bootstrap config, Kubernetes version, failure domain, metadata, node timeouts — unchanged. All options therefore produce the same node, differing only in infrastructure shape. Entries MUST vary only the infrastructure reference, which keeps options interchangeable and leaves failure-domain placement undisturbed.

Editing the preference MUST NOT recycle running nodes: changing a running node's desired shape is a rollout as today, whereas adding or reordering a fallback only affects still-pending placement. Because the field lives inside the hashed template, replica-managing owners (`MachineDeployment`/`MachineSet`, KCP) MUST exclude `infrastructureFallbackPreference` from the template hash / clone-from diff and MUST detect rollouts from the *resolved* per-`Machine` infrastructure reference. This reuses the "mutate a template field without a rollout" mechanism the [in-place-propagation proposal](./20221003-In-place-propagation-of-Kubernetes-objects-only-changes.md) established for metadata and node timeouts, so no new machinery is required; it matters most for control-plane providers, where an accidental rollout rolls etcd and the API server.

**2. Pre-provision capacity condition (infrastructure-machine contract).**

```
Condition type: CapacityAvailable
Status:         False
Reason:
  InsufficientCapacity    # valid option, no capacity now  -> advance; may retry later
  ConfigurationNotFound   # option does not exist          -> advance; never retry this option
  CapacityCheckDeferred   # provider could not check        -> retry the SAME option (transient)
```

The provider MUST set this with nothing allocated, and the condition's `observedGeneration` MUST match the infrastructure object's generation before the owning controller acts on it. Absent condition is treated as "capacity available" for backward compatibility.

**3. Advance behavior and provider capability.**

```yaml
# declared on the InfraMachineTemplate, or a well-known annotation on the InfraMachine CRD
spec:
  capabilities:
    inPlaceCapacityRotation: true   # provider accepts in-place spec mutation (see below)
```

- `true` → owning controller **patches** the infrastructure object's spec to the next option (`infrastructureRef` for index 0, else `infrastructureFallbackPreference[index-1]`); the provider re-evaluates and re-provisions. No object churn.
- false/unset → owning controller **replaces**: deletes the (unprovisioned) infrastructure object and materializes a fresh one from the next option, repointing `Machine.spec.infrastructureRef`. Churns the CR but not a real resource.

The owning controller always drives the rotation; the provider capability only selects the mechanism. A provider that declares `inPlaceCapacityRotation` owes three things, all only while the object is unprovisioned: allow its spec to be mutated (rather than rejecting edits as immutable), re-run its capacity preflight and emit a fresh condition on the new generation, and re-provision from the mutated spec. A provider that declares none of this still works via the replace path, which is why in-place is opt-in.

Controller flow:

```
read CapacityAvailable (matching observedGeneration) on the current infra object:
  True / absent              -> proceed; provisioning continues
  False, CapacityCheckDeferred
                             -> requeue with backoff; same option
  False, InsufficientCapacity | ConfigurationNotFound
                             -> next = (selectedIndex + 1) mod totalOptions   (cyclic)
                                advance via patch (inPlaceCapacityRotation) or delete+recreate
                                record status.placement.selectedIndex = next
  every option walked, none available
                             -> any option InsufficientCapacity: hold; back off; keep re-probing index 0
                                (recovers automatically when capacity returns)
                             -> every option ConfigurationNotFound: no option can ever satisfy;
                                surface for operator action (re-probe only resumes if the spec changes)
```

Forward progress: while the `Machine` is pending (nothing provisioned), each probe is only a spec the provider has not fulfilled, so cycling is churn-free. The controller provisions the first option that reports capacity as it advances; while every option is unavailable it keeps re-probing index 0, so a freed-up preferred option is taken as soon as nothing else is available. It does not move off an available less-preferred option to a more-preferred one that frees up afterward.

### Risks and Mitigations

- **Provider reports capacity failure only after allocating,** defeating NFR1. Mitigation: make "pre-provision" an explicit MUST and a conformance test; post-provision failures stay ordinary provisioning errors, not fallback triggers.
- **Preference edits trigger rollouts.** The field lives inside the hashed template. Mitigation: exclude it from the template hash / clone-from diff and detect rollouts from the resolved per-`Machine` reference (as the in-place-propagation proposal does for metadata and timeouts); restricting options to vary only `infrastructureRef` keeps the exclusion narrow.
- **Stale condition double-advance.** After an in-place patch the provider has not re-checked, so the condition still reflects the previous option; acting on it would race through every option in one pass. Mitigation: the `observedGeneration` match requirement.
- **In-place providers cache the old config** and skip re-provisioning on a spec change. Mitigation: the `inPlaceCapacityRotation` contract requires re-evaluation while `CapacityAvailable=False`; covered by conformance tests.
## Alternatives

- **In-place mutation only.** Rejected as the sole mechanism: many providers mark infrastructure specs immutable and will not re-provision on edits, so in-place cannot be the only path. The replace path makes the feature work for any conforming provider.
- **Post-provision failure detection instead of a pre-provision probe.** Rejected: loses the churn-free guarantee; every fallback would create and tear down a real resource.

## Upgrade Strategy

All additions are optional and additive. On upgrade:

- To keep previous behavior: no action. A `Machine` with only `infrastructureRef` is unchanged; a provider that never sets `CapacityAvailable` is treated as always having capacity.
- To adopt the feature: set `infrastructureFallbackPreference` on the owning resource, and use a provider that implements the capacity condition. `inPlaceCapacityRotation` defaults to false, so the safe replace path is used unless a provider opts in.
