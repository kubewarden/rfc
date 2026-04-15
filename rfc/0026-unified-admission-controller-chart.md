|              |                                                    |
| :----------- | :------------------------------------------------- |
| Feature Name | Unified Kubewarden Admission Controller Helm Chart |
| Start Date   | 2026-04-15                                         |
| Category     | Helm Charts                                        |
| RFC PR       | https://github.com/kubewarden/rfc/pull/54          |
| State        | **ACCEPTED**                                       |

# Summary

[summary]: #summary

This RFC proposes merging the three existing Kubewarden Helm charts
(`kubewarden-crds`, `kubewarden-controller`, and `kubewarden-defaults`) into a
single admission-controller chart. CRDs are moved under `templates/crds` so
they participate in the normal upgrade lifecycle. The `default` PolicyServer
and the recommended policies are defined as full Kubernetes YAML in a ConfigMap
that the controller reads periodically. A `DefaultsApplier` runs on a
configurable ticker (like the existing `CertReconciler`); on each tick it
decodes each resource into the corresponding typed CRD struct, injects
ownership labels, and creates or updates it via the API server. Stale managed
resources are cleaned up automatically. This simplifies the installation
process from a multi-chart, order-dependent procedure to a single `helm
install` command.

# Motivation

[motivation]: #motivation

Kubewarden currently ships three separate Helm charts:

- **kubewarden-crds** (mandatory): provides the CRDs.
- **kubewarden-controller** (mandatory): installs the controller.
- **kubewarden-defaults** (optional): defines the `default` PolicyServer and
  recommended policies.

This split exists because Helm does not guarantee resource ordering inside the
`templates/` directory, and CRDs must exist before any Custom Resources that
depend on them. However, managing three charts creates several problems:

1. **Installation complexity**: users must install three charts in a specific
   order and keep versions coordinated across all of them. This is an unusual
   and error-prone workflow.
2. **Upgrade friction**: upgrading the stack requires upgrading each chart
   separately in the correct sequence. A mistake in ordering can leave the
   cluster in a broken state.
3. **Helm limitations with the `/crds` directory**: resources placed in the
   special `/crds` folder are installed first but are **never upgraded** by
   Helm, which is a maintenance hazard as CRDs evolve.

The goal is to converge on a single chart that provides a straightforward
install-and-upgrade experience for users while keeping the project easier to
maintain.

## Examples / User Stories

[examples]: #examples

> As a cluster administrator, I want to install the entire Kubewarden stack
> (CRDs, controller, default PolicyServer and recommended policies) with a
> single `helm install` command.

> As a cluster administrator, I want `helm upgrade` to correctly update CRDs,
> the controller, and the default resources without manual ordering.

> As a Kubewarden maintainer, I want a single installation flow to test and
> maintain, reducing the surface area for upgrade-related bugs.

> As a cluster administrator, I want my own policies deployed on the `default`
> PolicyServer to be preserved across Helm upgrades.

# Detailed design

[design]: #detailed-design

## Chart restructuring

The three existing charts are merged into a single chart
(`kubewarden-admission-controller` or equivalent). CRDs are placed under
`templates/crds/` (not the special `/crds` directory) so they are rendered and
upgraded like any other template resource.

## Defining default resources via the controller

The `default` PolicyServer and the recommended admission policies are no longer
rendered directly as Helm templates. Instead, the controller itself is
responsible for creating and maintaining them:

1. **ConfigMap**: A ConfigMap is rendered by Helm containing the **full
   Kubernetes YAML definitions** of the desired default resources — one
   ConfigMap data key per resource (e.g., `policyserver-default.yaml`,
   `cap-no-privilege-escalation.yaml`). The Helm template resolves all values
   at render time, producing complete resource manifests. Disabled resources
   are simply omitted from the ConfigMap. When all defaults are disabled, the
   ConfigMap is not rendered at all.

   Helm values control which resources are included, their configuration (labels,
   annotations, `namespaceSelector`, default registry, failure policy, etc.).
   The ConfigMap is a regular Helm-managed resource, so `helm upgrade`
   naturally updates it.

2. **DefaultsApplier (periodic, leader-only)**: A `DefaultsApplier` component
   is registered with the controller-runtime manager using the `Runnable` and
   `LeaderElectionRunnable` interfaces (the same pattern used by the existing
   `CertReconciler`). When the leader is elected, its `Start()` method begins a
   periodic loop controlled by a configurable ticker. On each tick (and
   immediately on startup) it runs the following phases:

   - **Phase 1 — Read ConfigMap**: Fetch the `kubewarden-defaults` ConfigMap
     from the deployment namespace. If the ConfigMap does not exist, clean up
     all resources carrying the managed ownership labels and wait for the next
     tick.
   - **Phase 2 — Apply desired resources**: For each `*.yaml` key in the
     ConfigMap data, decode the value into the appropriate typed CRD struct
     (e.g., `v1.PolicyServer`, `v1.ClusterAdmissionPolicy`) using the
     controller's registered scheme. Inject ownership labels, then create the
     resource if it does not exist or update it if it does. Because the resources
     are applied through the API server, they pass through the controller's own
     validating and mutating webhooks — ensuring defaults are validated exactly
     like user-created resources.
   - **Phase 3 — Clean up stale resources**: List all PolicyServers and
     ClusterAdmissionPolicies carrying the full set of ownership labels. Delete
     any that are absent from the desired set (the ConfigMap keys).
   - **Wait**: Sleep until the next tick. The ticker duration is configurable
     via a Helm value (e.g., `defaults.reconcileInterval`, default: `5m`). The
     loop continues until the context is cancelled (controller shutdown).

   This ensures:

   - CRDs are expected to be registered before the controller starts (they are
     rendered under `templates/crds/`). If a CRD is not yet available, the
     Create/Update call fails and the next tick retries automatically.
   - Recommended policies are cleanly removed when disabled via Helm values.
   - The managed `default` PolicyServer is removed when disabled, though this
     causes all policies bound to it to be deleted by the controller's existing
     reconciliation logic.
   - Idempotency is guaranteed by the Create-or-Update logic. Re-applying the
     same desired state is a no-op.
   - On failure, the next tick retries the operation automatically.
   - ConfigMap changes from `helm upgrade` are picked up on the next tick
     without requiring a pod restart.

### ConfigMap structure

The ConfigMap holds the **full Kubernetes YAML definition** of each default
resource. Each key is named with a `.yaml` extension so
the controller can distinguish resource entries from other data. The Helm chart
templates render the complete `PolicyServer` and `ClusterAdmissionPolicy`
objects directly into the ConfigMap data keys. The controller decodes each entry
into the corresponding typed CRD struct (e.g., `v1.PolicyServer`,
`v1.ClusterAdmissionPolicy`) using its registered scheme, injects ownership
labels, and creates or updates the resource via the API server.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubewarden-defaults
  namespace: kubewarden
data:
  policyserver-default.yaml: |
    apiVersion: policies.kubewarden.io/v1
    kind: PolicyServer
    metadata:
      name: default
      finalizers:
        - kubewarden.io/finalizer
    spec:
      image: ghcr.io/kubewarden/policy-server:v1.35.0
      serviceAccountName: policy-server
      replicas: 1
      env:
        - name: KUBEWARDEN_LOG_LEVEL
          value: info
      namespacedPoliciesCapabilities:
        - "*"
  cap-no-privilege-escalation.yaml: |
    apiVersion: policies.kubewarden.io/v1
    kind: ClusterAdmissionPolicy
    metadata:
      name: no-privilege-escalation
      annotations:
        io.kubewarden.policy.severity: medium
        io.kubewarden.policy.category: PSP
    spec:
      mode: monitor
      module: ghcr.io/kubewarden/policies/allow-privilege-escalation-psp:v1.0.10
      mutating: true
      rules:
        - apiGroups: [""]
          apiVersions: ["v1"]
          resources: ["pods"]
          operations: ["CREATE"]
      settings:
        allowPrivilegeEscalation: false
  # ... additional recommended policies, one key per resource
```

**One ConfigMap key = one desired resource.** The set of `*.yaml` keys defines
the desired state. The controller iterates over all keys ending in `.yaml`,
decodes each into the appropriate typed CRD struct using the scheme, and
creates or updates the resource via the API server.

**Helm templates do the rendering.** All values (`{{ .Values.* }}`) are
resolved at Helm template time. The controller receives fully-formed YAML and
does not need to understand Helm values or custom configuration schemas.

**Disabled resources are omitted.** When a resource is disabled via Helm
values (e.g., `defaults.recommendedPolicies.enabled: false`), the Helm
template simply does not render that key into the ConfigMap data. The
controller sees a smaller set of desired resources and cleans up the stale
ones.

**Absent ConfigMap = clean up everything.** When all defaults are disabled,
the Helm chart does **not** render the ConfigMap at all. The controller
detects the missing ConfigMap and cleans up all resources carrying the
managed ownership labels. This is the "full disable" path.

**Ownership label**: The controller injects a single ownership label onto each
resource **at apply time** (it is not present in the Helm-rendered YAML):

- `kubewarden.io/managed-by: kubewarden-controller-defaults` — identifies the
  resource as created and managed by the defaults applier; uses a
  Kubewarden-specific key to avoid collision with the generic
  `app.kubernetes.io/managed-by` label

This label enables scoped cleanup: the applier only deletes stale resources
that carry it. User-managed policies remain untouched unless the managed
`default` PolicyServer itself is deleted. Since only one Kubewarden instance
can run per cluster, no additional release-scoping label is needed.

> [!NOTE]
> Kubewarden does not support multiple instances on the same cluster. The
> controller uses cluster-scoped CRDs (PolicyServer, ClusterAdmissionPolicy,
> etc.) and a single leader election identity. Installing a second instance
> would cause conflicts in leader election, webhook registrations, and CRD
> ownership. This is an existing limitation, not introduced by this RFC.

Users configure the default PolicyServer and recommended policies entirely
through Helm values; they should **not** edit the ConfigMap directly.
Documenting this clearly is important, though we do not mark the ConfigMap as
`immutable` because Helm must be able to update it on `helm upgrade`.

### DefaultsApplier implementation outline

The `DefaultsApplier` follows the same pattern as the existing
`CertReconciler`. It implements the `Runnable` and `LeaderElectionRunnable`
interfaces from controller-runtime. The manager starts it only on the leader
instance. It runs a periodic loop controlled by a configurable ticker,
re-reading the ConfigMap and reconciling default resources on each tick.

Because the ConfigMap holds full Kubernetes resource YAML and the controller
already registers the Kubewarden CRD types in its scheme, each entry is decoded
into the appropriate typed Go struct (e.g., `v1.PolicyServer`,
`v1.ClusterAdmissionPolicy`). The controller injects ownership labels, then
creates the resource if it does not exist or updates it if it does. This
approach gives compile-time type safety and ensures the resources pass through
the controller's own validating and mutating webhooks.

**Startup and periodic loop**: When the leader is elected, the applier runs an
initial reconciliation immediately, then enters a ticker loop. On each tick it
performs the same reconciliation. If a reconciliation fails, the error is logged
and the next tick retries. The loop exits when the context is cancelled
(controller shutdown). The ticker duration is exposed as a CLI flag on the
controller binary (e.g., `--defaults-reconcile-interval`) and configured via a
Helm value (e.g., `defaults.reconcileInterval: 5m`). The Helm chart passes this
value to the controller Deployment's container args.

**Reconciliation logic**:

1. Fetch the `kubewarden-defaults` ConfigMap from the deployment namespace. If
   not found, clean up all resources carrying the managed ownership labels and
   return.
2. Iterate over all ConfigMap data keys ending in `.yaml`. For each key, decode
   the YAML into a typed CRD object using the controller's scheme. Inject the
   the ownership label onto the decoded object.
3. For each decoded object, check whether it already exists in the cluster. If
   it does not exist, create it. If it does, set the resource version from the
   existing object and update it. Both operations go through the API server and
   therefore through the controller's validating and mutating webhooks.
4. Track the set of desired resource names (kind + name) from the ConfigMap.
5. List all PolicyServers and ClusterAdmissionPolicies carrying the full set of
   ownership labels. Delete any that are present in the cluster but absent from
   the desired set — these are stale managed resources.

The applier is instantiated and added to the controller-runtime
manager alongside existing reconcilers, using the same `mgr.Add()` call pattern
as the `CertReconciler`.

**Safety guarantees**:

- Only resources with the ownership label are considered for deletion.
- User-managed policies (lacking the label) are never deleted by the applier.
- Resources are applied through the API server and pass through the controller's
  validating and mutating webhooks, ensuring defaults are validated exactly like
  user-created resources.
- The controller overwrites managed resources entirely on each apply. User
  customization of managed defaults must go through Helm values, not manual
  edits.
- Disabling the managed `default` PolicyServer is destructive because the
  controller will delete all policies bound to it.
- **Upgrade safety**: the ConfigMap YAML and the controller binary ship in the
  same Helm chart, so the typed Go structs always match the ConfigMap schema.
  During a rolling upgrade there is a brief window where the old binary reads
  the new ConfigMap; Go's `json.Unmarshal` silently ignores unknown fields, so
  the worst case is a no-op apply of slightly stale data. The new pod corrects
  it on restart.

### User policy impact and default resource lifecycle

Because label-scoped deletion is used (not blanket
deletion and recreation), user-managed policies are preserved across upgrades
when only recommended policies are being reconfigured. However, if the managed
`default` PolicyServer is disabled, the applier deletes it and the controller
then deletes all policies bound to that PolicyServer, including user-managed
ones.

**Managed PolicyServer enable/disable transitions**: When the managed `default`
PolicyServer is disabled by toggling `values.yaml` (for example,
`defaults.policyServer.enabled: false`):

1. Helm updates (or removes) the ConfigMap.
2. On the next tick, `DefaultsApplier.reconcile()` reads the updated ConfigMap.
   If the PolicyServer's YAML key is absent (or the ConfigMap itself is absent),
   the PolicyServer is no longer in the desired set.
3. The stale `default` PolicyServer resource is deleted because it carries the
   ownership labels but is absent from the desired set.
4. The controller observes the PolicyServer deletion and deletes all policies
   bound to it, including user-managed ones.
5. When the `default` PolicyServer is re-enabled, it is recreated on the next
   upgrade, but previously deleted policies are not automatically restored.

### Helm values surface

The existing values from `kubewarden-controller` and `kubewarden-defaults` are
merged into a single `values.yaml` in the unified chart. Where field names do
not conflict, they are kept at the same level (i.e., no nesting under a new
parent key). If any field names conflict between the two charts, they will be
moved under a separate key (to be defined during implementation). The
`questions.yaml` for the Rancher UI is updated accordingly. The fact that a
ConfigMap and controller-managed applier are involved is hidden from the user.

# Migration path

[migration]: #migration

This is the recommended approach for its simplicity. There is a window between
uninstalling the old charts and completing the new installation where **no
admission control is active** — any workload can be deployed unchecked during
this period.

1. **Backup all policies and PolicyServers**: uninstalling `kubewarden-crds`
   cascade-deletes **all** custom resources, so every policy and PolicyServer
   must be backed up — not just user-managed ones:
   ```sh
   kubectl get clusteradmissionpolicies -A -o yaml > clusteradmissionpolicies-backup.yaml
   kubectl get admissionpolicies -A -o yaml > admissionpolicies-backup.yaml
   kubectl get clusteradmissionpolicygroups -A -o yaml > clusteradmissionpolicygroups-backup.yaml
   kubectl get admissionpolicygroups -A -o yaml > admissionpolicygroups-backup.yaml
   kubectl get policyservers -o yaml > policyservers-backup.yaml
   ```
2. **Uninstall old charts** in reverse order. Uninstalling `kubewarden-crds`
   cascade-deletes all CRs (PolicyServers and policies), producing a clean
   slate:
   ```sh
   helm uninstall kubewarden-defaults -n <namespace>
   helm uninstall kubewarden-controller -n <namespace>
   helm uninstall kubewarden-crds -n <namespace>
   ```
3. **Install the unified chart**: this creates CRDs, the controller, and the
   controller's defaults applier provisions the default PolicyServer and
   recommended policies on leader election:
   ```sh
   helm install <release-name> kubewarden/kubewarden-admission-controller -n <namespace>
   ```
4. **Restore user policies**: once the default PolicyServer is ready, re-apply
   all backed-up resources. The default PolicyServer and recommended policies
   will be recreated by the controller's applier, so it is safe to restore
   everything — the applier will overwrite them on the next tick with the
   correct ownership labels:
   ```sh
   kubectl apply -f policyservers-backup.yaml
   kubectl apply -f clusteradmissionpolicies-backup.yaml
   kubectl apply -f admissionpolicies-backup.yaml
   kubectl apply -f clusteradmissionpolicygroups-backup.yaml
   kubectl apply -f admissionpolicygroups-backup.yaml
   ```

# Drawbacks

[drawbacks]: #drawbacks

- **Breaking change**: users must migrate from three charts to one (backup,
  uninstall, reinstall). This incurs a window with no admission control active.
  A migration guide and/or automation script must be provided.
- **Controller code changes**: the controller gains new responsibility for
  creating default resources. This adds complexity to the codebase and requires
  thorough testing of the periodic applier logic.
- **Cleanup scope complexity**: the applier uses label selectors to identify
  stale managed default resources for deletion. A misconfigured or stale label
  can lead to unintended deletions. Mitigation: ownership labels are
  Helm-templated and immutable once set; safety is enforced by requiring **all
  the ownership label before deletion.
- **Destructive PolicyServer disable**: disabling the managed `default`
  PolicyServer is a destructive operation. Once the applier deletes the
  PolicyServer, the controller deletes all policies bound to it, including
  user-managed ones. This must be clearly communicated in chart notes and
  documentation.
- **Orphaned resources**: if a user manually removes ownership labels from a
  controller-managed default resource to prevent deletion, it becomes
  orphaned — no longer managed by the applier but not user-managed either. The
  cleanup logic will skip it. Documentation must clearly discourage this.
- **Ticker delay**: changes to the defaults ConfigMap are not picked up
  immediately — there is a delay of up to one ticker interval (default: 5
  minutes) before the applier reconciles. This is acceptable for default
  resource management, which is not latency-sensitive.
- **Downgrade leaves orphaned resources**: if a cluster is downgraded from a
  chart version with the defaults feature to an older version without it, the
  managed resources (PolicyServer, recommended policies) are left behind with
  ownership labels that no controller code understands. They continue to
  function but are no longer managed. Manual cleanup is required.
- **Ordering within `templates/`**: Helm does not guarantee strict ordering of
  resources under `templates/`. In practice Helm applies resources by
  kind in a well-known order (Namespaces → RBAC → Deployments → …), and CRDs
  are applied early. If CRDs are not yet available when the applier first runs,
  the Create/Update call fails and the next tick retries automatically.
- **CRD deletion on uninstall**: Moving CRDs from the `/crds`
  directory to `templates/crds/` causes `helm uninstall` to delete the CRDs,
  triggering cascading deletion of ALL custom resources cluster-wide — every
  PolicyServer and policy in the entire cluster, not just chart-managed ones.
  This is catastrophic data loss with no recovery mechanism. Mitigation: add
  annotation `"helm.sh/resource-policy": "keep"` to CRD templates so Helm
  never deletes them, or keep CRDs in `/crds` directory (trade-off: manual
  CRD upgrades required).
- **Helm rollback limitations**: `helm rollback` only reverts Helm-managed
  templates (ConfigMap, Deployment) but does NOT restore cluster resources
  created or deleted by the applier. Deleted policies are permanently lost and
  won't be restored on rollback. If the PolicyServer was disabled,
  controller-cascaded deletion of user policies cannot be recovered.
  Workarounds: use `helm upgrade --dry-run` to preview changes and maintain
  external backups.
- **Dual active policies during name changes**: because Phase 3a (apply) runs
  before Phase 3b (cleanup), renaming a recommended policy between chart
  versions creates a window where both the old-named and new-named policies are
  simultaneously active on the cluster. Mitigation: avoid renaming policies
  between versions; treat renames as a removal plus an addition and document the
  brief dual-enforcement window in release notes.

# Alternatives

[alternatives]: #alternatives

## ConfigMap + post-install/post-upgrade Job

An earlier version of this RFC proposed having the ConfigMap hold full YAML
resource definitions (one key per resource) and a text inventory file. A
Kubernetes Job annotated with `helm.sh/hook: post-install,post-upgrade` would
mount the ConfigMap, run `kubectl apply` for each resource, and diff the
inventory to clean up stale defaults.

This was discarded because:

- The Job requires its own ServiceAccount with cluster-wide write access to
  PolicyServer and policy CRDs, creating a significant RBAC attack surface.
- Job reliability depends on the controller being fully ready within a timeout,
  which can fail in clusters with slow image pulls.
- Debugging failures inside a Job is less visible than controller logs.
- The complexity of managing the Job lifecycle (retries, cleanup, RBAC) across
  install/uninstall/upgrade scenarios was deemed too high relative to the
  simpler controller-based approach.

## Post-install hook with inline bash and hard-coded YAML

The first alternative considered was a post-install Job with the PolicyServer
and policy definitions embedded directly in a bash script using heredocs and
`kubectl apply`. While functional and used by other projects, this approach was
discarded because maintaining complex YAML inside a bash heredoc is fragile,
hard to review, and error-prone. The ConfigMap approach separates data
(resource definitions) from logic (the apply script), making it cleaner and
easier to maintain.

## Helm post-install hooks on the resources themselves

Another approach explored was adding `helm.sh/hook: post-install` annotations
directly onto the PolicyServer and recommended policy templates, with
`helm.sh/hook-weight` to control ordering (creating resources before the
validating webhook is registered).

This was discarded for a critical reason: Helm hooks with
`hook-delete-policy: before-hook-creation` **delete the `default` PolicyServer
on every `helm upgrade`**. The Kubewarden controller's reconciliation logic then
cascades this deletion to all policies targeting the default PolicyServer.
Including user-deployed policies not managed by the chart. This means every
`helm upgrade` (even a config-only one like `--set logLevel=debug`) permanently
destroys user policies. This is an unacceptable data-loss scenario.

Without `hook-delete-policy: before-hook-creation`, uninstalling the chart
leaves the webhook configuration behind, which breaks subsequent installs.

## Subcharts approach

A superchart containing three subcharts (`crds`, `controller`, `defaults`) was
considered. Helm installs dependencies in the order declared in `Chart.yaml`,
which would provide CRDs → controller → defaults ordering.

This was discarded because:

- Helm does **not** wait for readiness between subcharts. The `defaults`
  subchart's resources may be applied before the controller is ready to serve
  the validating webhook, causing validation failures.
- Helm renders all templates across subcharts and applies them grouped by
  resource kind, not by subchart, undermining the assumed sequential order.
- Nested subcharts add complexity when this chart is itself consumed as a
  dependency inside a larger superchart.

## Do nothing

Keeping three separate charts is always an option but leaves the installation
complexity and upgrade ordering problems unsolved.

# Unresolved questions

[unresolved]: #unresolved-questions

- **PolicyServer deletion cascading to policies**: the current controller
  behavior of deleting all policies when their PolicyServer is removed is
  orthogonal to this RFC but amplifies the risk of any approach that deletes
  the PolicyServer during upgrades. A separate investigation should evaluate
  changing this to leave policies in a "pending" state instead.
- **Orphaned managed resources handling**: should the cleanup logic detect orphaned
  resources (those that lost ownership labels after chart creation) and warn the
  user, or simply skip them? This affects discoverability of accidental manual
  modifications.
- **`questions.yaml` design**: the exact layout of the Rancher UI questions for
  the unified chart values needs to be defined in collaboration with the UI
  team. This includes toggles for enabling/disabling individual recommended
  policies, toggles for enabling/disabling the managed `default` PolicyServer,
  and clear warnings about what happens when default resources are disabled.
- **`helm test` validation hook**: the chart should ship a `helm test` hook
  that verifies the expected PolicyServer and recommended policies exist and
  are in a healthy state after installation or upgrade. The applier's successful
  return only confirms the resources were applied without errors, not that they
  are functioning correctly. The test hook design (which resources to check, what
  "healthy" means for each, and how to report failures) needs to be defined.
