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

The AllowancePolicy specifies which fields to capture via `capture` in each rule:

```yaml
spec:
  rules:
  - trigger: "spec.replicas"
    conditions:
    - "has(object.metadata.annotations['jira'])"
    capture:                                         # fields to store in trace
    - "metadata.annotations[jira]"
    - "metadata.annotations[approved-by]"
    policies:
    - ...
```

- `conditions` validates the field exists or matches a pattern
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
  name: deployment-policy
spec:
  for:
    apiGroup: apps
    apiVersion: v1
    kind: Deployment

  subjects:
  - kind: Group
    name: platform-team
    mayInitiate: true
  - kind: ServiceAccount
    name: deployment-controller
    namespace: kube-system

  # Policies during initialization (omit or empty = unrestricted)
  initializing:
    when: "!has(object.status.observedGeneration)"
    policies:
    - target:
        apiGroup: apps
        apiVersion: v1
        resource: replicasets
      relation: ControllerChild
      verbs: [Create]
      mutations:
      - jsonPath: "*"
        verbs: [Insert, Mutate]

  # Policies during deletion (omit or empty = unrestricted)
  deleting:
    policies:
    - target:
        apiGroup: apps
        apiVersion: v1
        resource: replicasets
      relation: ControllerChild
      verbs: ["*"]

  # Steady-state rules (trigger → policies mappings)
  rules:
  - trigger: "spec.replicas"
    policies:
    - target:
        apiGroup: apps
        apiVersion: v1
        resource: replicasets
      relation: ControllerChild
      verbs: [Update, Delete]
      mutations:
      - jsonPath: "spec.replicas"
        verbs: [Mutate]

  - trigger: "spec.template.spec.containers[*].image"
    conditions:
    - "object.metadata.labels['env'] != 'prod'"
    - "has(object.metadata.annotations['jira'])"
    capture:
    - "metadata.annotations[jira]"
    - "metadata.annotations[approved-by]"
    policies:
    - target:
        apiGroup: apps
        apiVersion: v1
        resource: replicasets
      relation: ControllerChild
      verbs: [Update]
      mutations:
      - jsonPath: "spec.template.spec.containers[*]"
        verbs: [Insert, Delete, Mutate]
    - target:
        external:
          provider: aws
          service: rds
      relation: External
      verbs: [Update, Delete, Create]
```

### Simplified Example

A schema-less policy that bounds children on any spec change, without enumerating field-level mutations:

```yaml
kind: AllowancePolicy
apiVersion: kausality.io/v1alpha1
metadata:
  name: deployment-policy-simple
spec:
  for:
    apiGroup: apps
    apiVersion: v1
    kind: Deployment

  subjects:
  - kind: ServiceAccount
    name: deployment-controller
    namespace: kube-system
    mayInitiate: true

  initializing:
    policies:
    - target:
        apiGroup: apps
        apiVersion: v1
        resource: replicasets
      relation: ControllerChild
      verbs: [Create]

  deleting:
    policies:
    - target:
        apiGroup: apps
        apiVersion: v1
        resource: replicasets
      relation: ControllerChild
      verbs: ["*"]

  rules:
  - trigger: "spec"
    policies:
    - target:
        apiGroup: apps
        apiVersion: v1
        resource: replicasets
      relation: ControllerChild
      verbs: ["*"]
```

This says:
- **`trigger: "spec"`** — any change under `spec` triggers the rule
- **`verbs: ["*"]`** — Create, Update, Delete all permitted for replicasets
- **No `mutations`** — all field mutations allowed when `verbs` includes `Update`

The effect: replicasets are bounded (tracked) but can be freely mutated when the parent spec changes. Other child types (not mentioned) are unrestricted.

### for

The object kind this policy applies to, fully qualified:

| Field | Description |
|-------|-------------|
| `apiGroup` | API group (e.g., `apps`, `crossplane.io`) |
| `apiVersion` | API version (e.g., `v1`, `v1alpha1`) |
| `kind` | Object kind (singular, CamelCase, e.g., `Deployment`, `ReplicaSet`) |

### subjects

Who may trigger this policy. Follows RBAC subject conventions.

| Field | Description |
|-------|-------------|
| `kind` | `User`, `Group`, or `ServiceAccount` |
| `name` | Subject name |
| `namespace` | For ServiceAccount only |
| `mayInitiate` | If `true`, can start a new allowance chain (not just propagate) |

Subjects without `mayInitiate` can only propagate allowances that already exist on a parent object.

### initializing

Upper bounds during initialization (before first successful reconcile).

| Field | Description |
|-------|-------------|
| `when` | CEL expression, true = still initializing. Default: `"!has(object.status.observedGeneration)"` |
| `policies` | List of policy entries bounding operations during initialization |

If `policies` is omitted or empty, initialization is unrestricted.

### deleting

Upper bounds when parent has `deletionTimestamp`.

| Field | Description |
|-------|-------------|
| `policies` | List of policy entries bounding operations during deletion |

If `policies` is omitted or empty, deletion is unrestricted.

### rules

List of trigger → policies mappings for steady state. Each rule defines upper bounds for downstream operations when a specific field changes.

| Field | Description |
|-------|-------------|
| `trigger` | JSON path of the field that triggers this rule (supports `[*]` wildcards) |
| `conditions` | CEL expressions that must all pass (optional) |
| `capture` | Fields to capture in the trace as attestations (optional) |
| `policies` | List of policy entries bounding downstream operations |

If `policies` is omitted or empty for a rule, that trigger allows unrestricted downstream operations.

CEL expressions have access to `object` and `oldObject`. Use cases:
- Require external references: `has(object.metadata.annotations['jira'])`
- Pattern validation: `object.metadata.annotations['jira'].matches('^INFRA-[0-9]+$')`
- Conditional requirements: `object.metadata.labels['env'] != 'prod'`
- Change direction: `object.spec.replicas > oldObject.spec.replicas`

Wildcards in trigger paths (e.g., `[*]`) match any index; traces record the actual index.

### policies

Each entry in `policies` defines an upper bound for a specific target GVR:

| Field | Description |
|-------|-------------|
| `target` | Target (required) — GVR for `ControllerChild`, or `external` map for `External` |
| `relation` | `ControllerChild` or `External` |
| `verbs` | Object-level operations allowed |
| `mutations` | Field-level operations allowed (only for `ControllerChild`) |

**Upper-bound semantics:**
- For each GVR, collect all policy entries that match it
- The union of those entries is the upper bound for that GVR
- GVRs not mentioned in any policy entry are **unrestricted**
- `policies: []` (empty) means nothing is restricted → everything unrestricted

**`target` is required.** Without a target, a policy entry doesn't restrict anything. The `target` uses `resource` (plural, lowercase) because it describes admission on API resources:

| Field | Description |
|-------|-------------|
| `apiGroup` | API group (e.g., `apps`, `crossplane.io`) |
| `apiVersion` | API version (e.g., `v1`, `v1alpha1`) |
| `resource` | Resource name (plural, lowercase, e.g., `deployments`, `replicasets`) |

### verbs (object-level)

For `ControllerChild`:
- `Create` — create the KRM object
- `Update` — update the KRM object (requires `mutations` to specify which fields)
- `Delete` — delete the KRM object
- `"*"` — all of the above

For `External`:
- Provider-specific verbs (e.g., Crossplane: `Create`, `Update`, `Delete`; Terraform: `apply`, `destroy`)
- `"*"` — all operations

### mutations (field-level)

Only for `relation: ControllerChild`. Specifies which fields may be mutated when `verbs` includes `Update`.

| Field | Description |
|-------|-------------|
| `jsonPath` | Path to the field that may be mutated (use `"*"` for any field) |
| `verbs` | `Insert`, `Delete`, `Mutate` |

- `Insert` — add new items (arrays/maps)
- `Delete` — remove items from arrays/maps or remove the field
- `Mutate` — change existing values

If `mutations` is omitted and `verbs` includes `Update`, all field mutations are allowed.

### target.external

Only for `relation: External`. Free-form `map[string]string` to identify the external resource type. Provider-specific, Kausality doesn't interpret it.

```yaml
# Crossplane
target:
  external:
    provider: provider-aws
    kind: RDSInstance

# Terraform
target:
  external:
    resource: aws_rds_cluster
```

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

## Object Lifecycle Phases

An object goes through three phases, each with different allowance rules:

### Initializing

During initialization, the object graph is being built. Defined by `initializing.when` CEL expression in the policy (default: `"!has(object.status.observedGeneration)"`).

Typical permissions:
- `Create` child objects: allowed
- `Insert`, `Mutate` fields: allowed (for late initialization)
- `Delete` children: usually not allowed

### Steady State

After initialization completes (CEL expression becomes false), explicit allowances are required for all changes. The `rules` section defines trigger → policies mappings.

### Deleting

When the parent has `deletionTimestamp`, the `deleting` section applies.

Default behavior:
- Child KRM objects: can be deleted (`verbs: ["*"]`)
- External resources: preserved (must be explicitly allowed)

## Controller, Composition, and Function Upgrades

When controllers, Crossplane Compositions, Functions, or other "logic" changes, reconciliation may produce different outputs for all affected objects. This requires a mechanism to grant upgrade allowances.

### User-Agent Based Detection

Kubernetes API requests include a User-Agent header with the format:
```
command/version (os/arch) kubernetes/commit
```

Example: `crossplane/v1.15.0 (linux/amd64) kubernetes/abc1234`

Admission webhooks receive this header. By storing a hash of the user-agent on each object, we can detect when a new controller version is making requests.

**Annotation on objects:**
```yaml
metadata:
  annotations:
    kausality.io/controller-ua-hash: "a1ef23b4"  # short hash of user-agent
```

**Detection flow:**
1. Controller mutates object
2. Admission computes hash of request's User-Agent
3. Compares with `kausality.io/controller-ua-hash` annotation
4. If different → upgrade detected → check UpgradeAllowance
5. If same → normal allowance rules apply

### UpgradeAllowance

Defines what mutations a ServiceAccount may perform when an upgrade is detected:

```yaml
kind: UpgradeAllowance
apiVersion: kausality.io/v1alpha1
metadata:
  name: crossplane-upgrade
spec:
  serviceAccount: system:serviceaccount:crossplane-system:crossplane
  policies:
  - target:
      apiGroup: rds.aws.crossplane.io
      apiVersion: v1alpha1
      resource: rdsinstances
    relation: ControllerChild
    mutations: ["spec.forProvider"]
  - target:
      external:
        provider: provider-aws
    relation: External
    verbs: [Update]
```

No `validFrom`/`validUntil` needed — activation is automatic based on user-agent change.

**Admission flow:**
1. Controller (new version) mutates object
2. Admission sees user-agent hash differs from annotation
3. Finds UpgradeAllowance for this ServiceAccount
4. Checks if mutation matches an entry in `policies`
5. If match: grant allowance, update `kausality.io/controller-ua-hash`
6. If no match or no UpgradeAllowance: block or escalate

**Benefits:**
- No timing coordination — detection is based on actual requests
- Old controller continues normally (same hash, no upgrade triggered)
- New controller automatically detected on first request
- Per-object: each object upgraded once per new binary
- UpgradeAllowance can be created before OR after deployment

### Workflow

1. Create UpgradeAllowance for the ServiceAccount (can be done anytime)
2. Deploy new controller version
3. New controller reconciles objects
4. Admission detects user-agent change, grants upgrade allowance
5. Annotation updated, subsequent reconciles use normal rules

### Open Questions

#### Composition and Function Changes

User-agent detection works for controller binary changes. But Crossplane Compositions and Functions can change independently of the controller binary.

Options:
- Treat Composition/Function changes as requiring explicit UpgradeAllowance with time window
- Hash Composition content and store alongside controller hash
- Accept that Composition changes are lower risk (they don't change external API calls directly)

#### Hash Stability

The user-agent includes:
- Binary name (from `os.Args[0]`)
- Version (build-time)
- OS/arch
- Git commit

Some of these may change without semantic controller changes (e.g., rebuilding same version). Consider:
- Hashing only the version portion
- Allowing wildcard patterns in UpgradeAllowance to ignore patch versions

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
policies:
# KRM mutations (what the controller may change on child MRs)
- target:
    apiGroup: rds.aws.crossplane.io
    apiVersion: v1alpha1
    resource: rdsinstances
  relation: ControllerChild
  mutations:
  - jsonPath: "spec.forProvider.instanceClass"
    verbs: [Mutate]

# External mutations (what the provider may do to cloud resources)
- target:
    external:
      provider: provider-aws
      kind: RDSInstance
  relation: External
  verbs: [Update]
```

The `verbs` vocabulary for External is system-specific:

| System | Verbs |
|--------|-------|
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
  name: database-claim-policy
spec:
  for:
    apiGroup: example.com
    apiVersion: v1alpha1
    kind: DatabaseClaim

  subjects:
  - kind: Group
    name: platform-team
    mayInitiate: true

  initializing:
    when: "!has(object.status.observedGeneration)"
    policies:
    - target:
        apiGroup: example.com
        apiVersion: v1alpha1
        resource: xdatabases
      relation: ControllerChild
      verbs: [Create]
      mutations:
      - jsonPath: "*"
        verbs: [Insert, Mutate]

  deleting:
    policies:
    - target:
        apiGroup: example.com
        apiVersion: v1alpha1
        resource: xdatabases
      relation: ControllerChild
      verbs: ["*"]
    - target:
        external:
          provider: provider-aws
      relation: External
      verbs: [Delete]

  rules:
  - trigger: "spec.size"
    conditions:
    - "has(object.metadata.annotations['jira'])"
    capture:
    - "metadata.annotations[jira]"
    policies:
    - target:
        apiGroup: example.com
        apiVersion: v1alpha1
        resource: xdatabases
      relation: ControllerChild
      verbs: [Update]
      mutations:
      - jsonPath: "spec.size"
        verbs: [Mutate]

  - trigger: "spec.engine"
    conditions:
    - "has(object.metadata.annotations['jira'])"
    capture:
    - "metadata.annotations[jira]"
    policies:
    - target:
        apiGroup: example.com
        apiVersion: v1alpha1
        resource: xdatabases
      relation: ControllerChild
      verbs: [Update]
      mutations:
      - jsonPath: "spec.engine"
        verbs: [Mutate]
---
kind: AllowancePolicy
apiVersion: kausality.io/v1alpha1
metadata:
  name: xdatabase-policy
spec:
  for:
    apiGroup: example.com
    apiVersion: v1alpha1
    kind: XDatabase

  subjects:
  - kind: ServiceAccount
    name: crossplane
    namespace: crossplane-system

  initializing:
    when: "!has(object.status.observedGeneration)"
    policies:
    - target:
        apiGroup: rds.aws.crossplane.io
        apiVersion: v1alpha1
        resource: rdsinstances
      relation: ControllerChild
      verbs: [Create]
      mutations:
      - jsonPath: "*"
        verbs: [Insert, Mutate]
    - target:
        external:
          provider: provider-aws
      relation: External
      verbs: [Create]

  deleting:
    policies:
    - target:
        apiGroup: rds.aws.crossplane.io
        apiVersion: v1alpha1
        resource: rdsinstances
      relation: ControllerChild
      verbs: ["*"]
    - target:
        external:
          provider: provider-aws
      relation: External
      verbs: [Delete]

  rules:
  - trigger: "spec.size"
    policies:
    - target:
        apiGroup: rds.aws.crossplane.io
        apiVersion: v1alpha1
        resource: rdsinstances
      relation: ControllerChild
      verbs: [Update]
      mutations:
      - jsonPath: "spec.forProvider.instanceClass"
        verbs: [Mutate]
    - target:
        external:
          provider: provider-aws
          kind: RDSInstance
      relation: External
      verbs: [Update]

  - trigger: "spec.engine"
    conditions:
    - "object.metadata.annotations['allow-recreate'] == 'true'"
    policies:
    - target:
        apiGroup: rds.aws.crossplane.io
        apiVersion: v1alpha1
        resource: rdsinstances
      relation: ControllerChild
      verbs: [Update]
      mutations:
      - jsonPath: "spec.forProvider.engine"
        verbs: [Mutate]
    - target:
        external:
          provider: provider-aws
          kind: RDSInstance
      relation: External
      verbs: [Delete, Create]  # destructive: recreate
```

## Kro Integration

[Kro](https://github.com/kubernetes-sigs/kro) (Kube Resource Orchestrator) defines resource DAGs via ResourceGraphDefinitions. The dependency graph can be used to automatically derive AllowancePolicies.

### ResourceGraphDefinition Example

```yaml
apiVersion: kro.run/v1alpha1
kind: ResourceGraphDefinition
metadata:
  name: webapp
spec:
  schema:
    apiVersion: example.com/v1alpha1
    kind: WebApp
    spec:
      name: string
      image: string
      replicas: integer

  resources:
  - id: deployment
    template:
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: ${schema.spec.name}
      spec:
        replicas: ${schema.spec.replicas}
        template:
          spec:
            containers:
            - name: app
              image: ${schema.spec.image}

  - id: service
    template:
      apiVersion: v1
      kind: Service
      metadata:
        name: ${schema.spec.name}
      spec:
        selector:
          app: ${deployment.metadata.labels.app}  # depends on deployment
```

### Deriving AllowancePolicy from DAG

The `${...}` references create explicit edges in the dependency graph:

| Schema Field | References | Resource Field |
|--------------|------------|----------------|
| `schema.spec.name` | → | `deployment.metadata.name` |
| `schema.spec.replicas` | → | `deployment.spec.replicas` |
| `schema.spec.image` | → | `deployment.spec.template.spec.containers[0].image` |
| `deployment.metadata.labels.app` | → | `service.spec.selector.app` |

From this, we can derive:

```yaml
kind: AllowancePolicy
apiVersion: kausality.io/v1alpha1
metadata:
  name: webapp-policy
  annotations:
    kausality.io/derived-from: webapp  # source RGD
spec:
  for:
    apiGroup: example.com
    apiVersion: v1alpha1
    kind: WebApp

  subjects:
  - kind: ServiceAccount
    name: kro-controller
    namespace: kro-system

  initializing:
    when: "!has(object.status.observedGeneration)"
    policies:
    - target:
        apiGroup: apps
        apiVersion: v1
        resource: deployments
      relation: ControllerChild
      verbs: [Create]
      mutations:
      - jsonPath: "*"
        verbs: [Insert, Mutate]
    - target:
        apiGroup: ""
        apiVersion: v1
        resource: services
      relation: ControllerChild
      verbs: [Create]
      mutations:
      - jsonPath: "*"
        verbs: [Insert, Mutate]

  rules:
  - trigger: "spec.name"
    policies:
    - target:
        apiGroup: apps
        apiVersion: v1
        resource: deployments
      relation: ControllerChild
      verbs: [Update]
      mutations:
      - jsonPath: "metadata.name"
        verbs: [Mutate]
    - target:
        apiGroup: ""
        apiVersion: v1
        resource: services
      relation: ControllerChild
      verbs: [Update]
      mutations:
      - jsonPath: "metadata.name"
        verbs: [Mutate]

  - trigger: "spec.replicas"
    policies:
    - target:
        apiGroup: apps
        apiVersion: v1
        resource: deployments
      relation: ControllerChild
      verbs: [Update]
      mutations:
      - jsonPath: "spec.replicas"
        verbs: [Mutate]

  - trigger: "spec.image"
    policies:
    - target:
        apiGroup: apps
        apiVersion: v1
        resource: deployments
      relation: ControllerChild
      verbs: [Update]
      mutations:
      - jsonPath: "spec.template.spec.containers[*].image"
        verbs: [Mutate]
```

### Derivation Algorithm

1. Parse ResourceGraphDefinition
2. For each `${schema.spec.X}` reference in a resource template:
   - Extract the schema field path (`spec.X`)
   - Extract the target resource and field path
   - Create a rule: `trigger: spec.X` → `policies: [{target, jsonPath}]`
3. For inter-resource references (`${resourceId.field}`):
   - Track transitive dependencies
   - Include in parent's policies list
4. Set `subjects` to the Kro controller ServiceAccount
5. Annotate with source RGD for traceability

### Automation

A Kro webhook or controller could:
1. Watch ResourceGraphDefinitions
2. Generate corresponding AllowancePolicies automatically
3. Keep them in sync when RGDs change

This eliminates manual policy authoring for Kro-managed resources.
