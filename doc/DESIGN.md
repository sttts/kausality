# Kausality

A system for tracing and gating spec changes through a hierarchy of KRM objects and downstream systems (e.g., Terraform). Controllers cannot mutate downstream unless explicitly allowed.

## Goals

**This is about safety, not security.**

We want a best-effort system that stays out of the way of the user, but enables:
- Traceability of destructive actions
- Avoidance of accidental damage where possible

The system assumes good intent. It protects against accidents, not malicious actors.

## Core Concepts

### Allowance

A permission for a controller to perform a specific mutation on a child object.

- Stored in parent object's annotations, protected by admission
- Additive: allowances accumulate without conflict
- Carries causation trace back to origin (the initiator)

### AllowancePolicy

A CRD defining rules that map parent field changes to permitted child mutations. Evaluated by the admission webhook.

## Allowance Storage

Allowances are stored in annotations to remain controller-agnostic:

```yaml
kind: Deployment
metadata:
  annotations:
    kausality.io/allowances: |
      - kind: ReplicaSet           # child kind the controller may mutate
        mutation: spec.replicas    # field the controller may change on child
        generation: 7              # generation of this object that caused it
        initiator: hans@example.com  # human/CI that started the chain
        trace:                     # causation path to this point
        - kind: Deployment
          name: foo
          generation: 7
          field: spec.replicas     # field change that triggered this
```

Propagated to child (ReplicaSet allows Pod mutations):

```yaml
kind: ReplicaSet
metadata:
  annotations:
    kausality.io/allowances: |
      - kind: Pod                  # child kind the controller may mutate
        mutation: delete           # operation permitted on child
        generation: 14             # generation of this object that caused it
        initiator: hans@example.com
        trace:
        - kind: Deployment
          name: foo
          generation: 7
          field: spec.replicas
        - kind: ReplicaSet         # this object, appended by admission
          name: foo-abc
          generation: 14
          field: spec.replicas
```

### Trace

The trace is a linear causation path from initiator to current object.

- **Linearity**: When multiple allowances could justify a mutation, one is chosen deterministically (e.g., alphabetic by `{kind}/{name}/{field}`)
- **Initiator**: The human or CI system that started the chain (only captured once, at the top level)
- **Hops**: Each intermediate object that propagated the allowance
- **Field**: Full JSON path including concrete indices (e.g., `spec.template.spec.containers[0].image`)
- **Attestations**: Optional captured values from the object (e.g., Jira ticket)

Namespace is omitted from trace hops — it's always the same as the object carrying the allowance (or cluster-scoped).

#### Attestations (Optional Extension)

Traces can capture external references as proof:

```yaml
kind: Deployment
metadata:
  annotations:
    kausality.io/allowances: |
      - kind: ReplicaSet
        mutation: spec.replicas
        generation: 7
        initiator: hans@example.com
        trace:
        - kind: Deployment
          name: foo
          generation: 7
          field: spec.replicas
          attestations:                              # captured values
            "metadata.annotations[jira]": "INFRA-23232"
            "metadata.annotations[approved-by]": "alice@example.com"
```

The AllowancePolicy specifies which fields to capture via `capture`:

```yaml
spec:
  match:
    kind: Deployment
    fields: ["spec.replicas"]
    cel:
    - "has(object.metadata.annotations['jira'])"
    capture:                                         # fields to store in trace
    - "metadata.annotations[jira]"
    - "metadata.annotations[approved-by]"
```

- `cel` validates the field exists or matches a pattern
- `capture` stores the actual value in the trace

The trace becomes self-documenting — auditable without looking up external systems.

### Consumption

Allowances are consumed based on `status.observedGeneration`:

- If `status.observedGeneration >= allowance.generation` → allowance is consumed
- Consumed allowances can be pruned by the next admission

This leverages existing controller behavior — no changes required to controllers.

## AllowancePolicy

```yaml
kind: AllowancePolicy
apiVersion: kausality.io/v1alpha1
metadata:
  name: deployment-to-replicaset
spec:
  match:
    kind: Deployment
    fields:
    - "spec.replicas"
    - "spec.template.spec.containers[*].image"    # wildcards supported
    cel:
    - "has(object.spec.jiraTicket)"
    - "object.metadata.labels['env'] != 'prod' || has(object.spec.jiraTicket)"

  subjects:
  - kind: Group
    name: platform-team
    mayInitiate: true
  - kind: ServiceAccount
    name: deployment-controller
    namespace: kube-system

  allow:
  - kind: ReplicaSet
    relation: Child
    mutations: ["spec.replicas", "spec.template"]
```

### match

Defines when this policy applies.

| Field | Description |
|-------|-------------|
| `kind` | Parent object kind |
| `fields` | Spec fields that trigger this rule (supports `[*]` wildcards) |
| `cel` | CEL expressions for additional conditions (optional) |
| `capture` | Fields to capture in the trace as attestations (optional) |

CEL expressions have access to `object` and `oldObject`. Use cases:
- Require external references: `has(object.spec.jiraTicket)`
- Pattern validation: `object.spec.jiraTicket.matches('^INFRA-[0-9]+$')`
- Conditional requirements: `object.metadata.labels['env'] != 'prod' || has(object.spec.jiraTicket)`
- Change direction: `object.spec.replicas > oldObject.spec.replicas`

Wildcards in policy fields (e.g., `[*]`) match any index; traces record the actual index.

### subjects

Who may trigger this rule. Follows RBAC subject conventions.

| Field | Description |
|-------|-------------|
| `kind` | `User`, `Group`, or `ServiceAccount` |
| `name` | Subject name |
| `namespace` | For ServiceAccount only |
| `mayInitiate` | If `true`, can start a new allowance chain (not just propagate) |

Subjects without `mayInitiate` can only propagate allowances that already exist on a parent object.

### allow

List of permitted downstream mutations when the policy matches.

| Field | Description |
|-------|-------------|
| `kind` | Target child object kind |
| `relation` | `Child` (via ownerRef) |
| `mutations` | Permitted operations: field paths or `delete`, `create` |

## Admission Flow

```
Human changes Deployment.spec.replicas
         |
         v
+---------------------------------------------+
| Admission Webhook (on Deployment)           |
|                                             |
| 1. Evaluate AllowancePolicies               |
| 2. Check subjects + mayInitiate             |
| 3. Check CEL conditions                     |
| 4. Inject allowances into annotations       |
+---------------------------------------------+
         |
         v
Controller mutates ReplicaSet
         |
         v
+---------------------------------------------+
| Admission Webhook (on ReplicaSet)           |
|                                             |
| 1. Find parent via ownerRef                 |
| 2. Check parent has matching allowance      |
| 3. Check subject (SA) is permitted          |
| 4. Inject downstream allowances             |
| 5. Prune consumed allowances on parent      |
+---------------------------------------------+
         |
         v
        ...
```

## Primitives

- Allowances stored on objects are protected by admission
- A mutation can only add allowances if:
  - Parent object has a matching allowance AND policy permits propagation, OR
  - Policy matches with a `mayInitiate` subject
- User identity comes from Kubernetes authentication (not self-declared)
- Default-deny: without a matching allowance, controllers cannot mutate downstream

## Creation vs Steady State

On initial creation, the entire object graph needs to be built. Every child is "new", not a "mutation". We need broader permissions during initialization.

**Implicit creation allowance**: If the parent object has never been reconciled (`status.observedGeneration` is unset or zero), child creation is implicitly allowed. Once the parent reaches steady state (has been successfully reconciled at least once), explicit allowances are required for changes.

This means:
- First reconciliation of a new Deployment → may freely create ReplicaSets
- Subsequent changes to the Deployment → require explicit allowances

The signal for "ready at least once" could be:
- `status.observedGeneration >= 1`
- A condition like `Ready=True` having been observed
- A dedicated annotation `kausality.io/initialized: "true"` set by the Kausality controller

## Controller and API Upgrades

When controllers or APIs are upgraded, reconciliation may need to touch all objects (new defaults, schema migrations, etc.). This requires a mechanism to grant temporary broad allowances.

### UpgradeAllowance

```yaml
kind: UpgradeAllowance
apiVersion: kausality.io/v1alpha1
metadata:
  name: deployment-controller-v1.2.3
spec:
  serviceAccount: system:serviceaccount:kube-system:deployment-controller
  validUntil: "2024-01-10T00:00:00Z"  # safety net
```

When admission processes a mutation:
1. Finds UpgradeAllowance(s) for the requesting ServiceAccount
2. Checks object annotation: `kausality.io/last-upgrade`
3. If annotation missing or references older UpgradeAllowance → grants broad allowance, sets annotation
4. If annotation matches current UpgradeAllowance → normal rules apply (already upgraded)

```yaml
# On object after upgrade
metadata:
  annotations:
    kausality.io/last-upgrade: "deployment-controller-v1.2.3"
```

Each object gets upgraded exactly once per UpgradeAllowance. The annotation tracks which upgrade has been applied.

### Open Question: Activation Timing

**Problem**: The UpgradeAllowance must become active only when the new controller is actually running. Otherwise:
- If activated before deployment: old controller might consume the upgrade allowance
- If activated after deployment: new controller is blocked until activation

Both old and new controller use the same ServiceAccount, so admission cannot distinguish them.

**Options considered:**

**A. Explicit activation**
```yaml
spec:
  active: false  # flip manually after deployment verified
```
- Pro: Simple, explicit
- Con: Window where new controller is blocked waiting for activation

**B. Time window**
```yaml
spec:
  validFrom: "2024-01-06T15:05:00Z"
  validUntil: "2024-01-10T00:00:00Z"
```
- Pro: Can be created ahead of time
- Con: Requires estimating deployment time; old controller might be running at validFrom

**C. Reference controller Deployment generation**
```yaml
spec:
  controllerRef:
    kind: Deployment
    name: deployment-controller
    namespace: kube-system
    minGeneration: 15
```
- Pro: Tied to actual deployment
- Con: Deployment generation doesn't indicate pods are actually running

**D. Pod-based activation**
```yaml
spec:
  activateWhen:
    podSelector:
      namespace: kube-system
      labels:
        app: deployment-controller
    image: "registry.io/deployment-controller:v1.2.3"
status:
  active: false  # set by Kausality controller when pod is Ready
```
- Pro: Activates exactly when new pod is ready
- Con: Brief overlap during rollout where both old and new pods run; complexity

**E. Different ServiceAccount per version**
- Pro: Clean separation
- Con: Operationally complex, requires SA rotation on every upgrade

**No perfect solution exists.** The fundamental problem is that admission cannot distinguish which version of a controller is making a request when they share a ServiceAccount.

Possible mitigations for the overlap window:
- Accept that old controller might reconcile during brief overlap (changes should be idempotent)
- Use rollout strategies that minimize overlap (Recreate instead of RollingUpdate)
- Controller-specific ServiceAccounts (option E) for critical controllers

## Crossplane Integration

Crossplane manages external resources (cloud infrastructure) through a hierarchy:

```
Claim (namespaced)
    → Composite Resource (XR, cluster-scoped)
        → Managed Resources (MRs)
            → External API (AWS, GCP, Terraform)
```

Each level connected by ownerRefs. Kausality traces and gates mutations through this chain.

### External Relation

For non-KRM resources, use `relation: External`:

```yaml
allow:
# KRM mutations (what the controller may change on child MRs)
- kind: RDSInstance
  relation: Child
  mutations: ["spec.forProvider.instanceClass"]

# External mutations (what the provider may do to cloud resources)
- relation: External
  mutations: ["Update"]
```

The `mutations` vocabulary for External is system-specific:

| System | Mutations |
|--------|-----------|
| Crossplane | `Create`, `Update`, `Delete` (managementPolicies verbs) |
| Terraform | `apply`, `destroy` |
| ArgoCD | `sync`, `delete` |

### Gating via managementPolicies

Crossplane providers respect `spec.managementPolicies` on Managed Resources. Kausality uses this to gate external mutations without modifying providers.

**Default state** — MRs are read-only:
```yaml
kind: RDSInstance
metadata:
  annotations:
    kausality.io/allowances: |
      []  # no allowances
spec:
  managementPolicies: ["Observe", "LateInitialize"]  # provider won't mutate
  forProvider:
    instanceClass: db.t3.medium
```

**When allowance arrives** — admission opens the gate:
```yaml
kind: RDSInstance
metadata:
  annotations:
    kausality.io/allowances: |
      - relation: External
        mutations: ["Update"]
        generation: 8
        initiator: hans@example.com
        trace:
        - kind: DatabaseClaim
          name: my-database
          generation: 5
          field: spec.size
        - kind: XDatabase
          name: my-database-xyz
          generation: 12
          field: spec.instanceClass
        - kind: RDSInstance
          name: my-database-xyz-rds
          generation: 8
          field: spec.forProvider.instanceClass
spec:
  managementPolicies: ["Observe", "LateInitialize", "Update"]  # gate open
  forProvider:
    instanceClass: db.t3.large  # changed
```

Provider sees `Update` in managementPolicies, performs the AWS API call.

### Kausality Controller

A controller watches MRs and closes the gate as soon as the mutation is applied:

```
+--------------------------------------------------+
| Kausality Controller (watches MRs)                  |
|                                                  |
| On reconcile:                                    |
| - If status.observedGeneration >= allowance.gen  |
|   AND managementPolicies contains mutation verbs |
|   → Patch spec.managementPolicies to read-only   |
|   → Prune consumed allowances                    |
+--------------------------------------------------+
```

This ensures the gate is open only for the duration of the mutation, not longer.

### Crossplane Admission Flow

```
User changes Claim.spec.size: "large"
         |
         v
+-----------------------------------------------+
| Admission (on Claim)                          |
| - Evaluates AllowancePolicies                 |
| - Injects allowances for XR mutations         |
+-----------------------------------------------+
         |
         v
Crossplane claim-controller updates XR.spec
         |
         v
+-----------------------------------------------+
| Admission (on XR)                             |
| - Checks Claim has allowance                  |
| - Injects allowances for MR mutations         |
| - Includes relation: External allowances      |
+-----------------------------------------------+
         |
         v
Composition controller updates MR.spec
         |
         v
+-----------------------------------------------+
| Admission (on MR)                             |
| - Checks XR has allowance                     |
| - Sets managementPolicies based on External   |
|   allowance mutations (e.g., adds "Update")   |
+-----------------------------------------------+
         |
         v
Provider sees managementPolicies: [..., "Update"]
         |
         v
Provider calls AWS API
         |
         v
Provider updates status.observedGeneration
         |
         v
+-----------------------------------------------+
| Kausality Controller                             |
| - Sees observedGeneration caught up           |
| - Reverts managementPolicies to read-only     |
| - Prunes consumed allowances                  |
+-----------------------------------------------+
```

### Example AllowancePolicy for Crossplane

```yaml
kind: AllowancePolicy
apiVersion: kausality.io/v1alpha1
metadata:
  name: database-claim-to-xr
spec:
  match:
    kind: DatabaseClaim
    fields: ["spec.size", "spec.engine"]
    cel:
    - "has(object.metadata.annotations['jira'])"
    capture:
    - "metadata.annotations[jira]"
  subjects:
  - kind: Group
    name: platform-team
    mayInitiate: true
  allow:
  - kind: XDatabase
    relation: Child
    mutations: ["spec.size", "spec.engine"]
---
kind: AllowancePolicy
apiVersion: kausality.io/v1alpha1
metadata:
  name: xdatabase-to-rds
spec:
  match:
    kind: XDatabase
    fields: ["spec.size"]
  subjects:
  - kind: ServiceAccount
    name: crossplane
    namespace: crossplane-system
  allow:
  - kind: RDSInstance
    relation: Child
    mutations: ["spec.forProvider.instanceClass"]
  - relation: External
    mutations: ["Update"]
---
kind: AllowancePolicy
apiVersion: kausality.io/v1alpha1
metadata:
  name: xdatabase-to-rds-destructive
spec:
  match:
    kind: XDatabase
    fields: ["spec.engine"]  # changing engine requires recreate
    cel:
    - "object.metadata.annotations['allow-recreate'] == 'true'"
  subjects:
  - kind: ServiceAccount
    name: crossplane
    namespace: crossplane-system
  allow:
  - kind: RDSInstance
    relation: Child
    mutations: ["spec.forProvider.engine"]
  - relation: External
    mutations: ["Delete", "Create"]  # destructive
```
