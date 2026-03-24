|              |                                                 |
| :----------- | :---------------------------------------------- |
| Feature Name | Policy Server permissions with mapping policy   |
| Start Date   | March 11th, 2026                                |
| Category     | Context aware policies                          |
| RFC PR       | [PR](TODO)                                      |
| State        | **ACCEPTED**                                    |

# Summary

[summary]: #summary

This RFC ensures the following promise: "If you can deploy namespaced
policies, you can do so without obtaining raised privileges". This allows
low-privileged users to self-service namespaced policies securely.

This RFC proposes a UX to gate context-aware capabilities in the PolicyServer
side for namespaced policies. This gating will be enabled by default, and
host-capabilities for namespaced policies will be opt-in from then on.

This PolicyServer gating will extend to cluster-wide policies
(`ClusterAdmissionPolicies` and `ClusterAdmissionPolicyGroups`).

# Motivation

[motivation]: #motivation

Namespaced policies (`AdmissionPolicies` and `AdmissionPolicyGroups`) cannot
fetch information from the Kubernetes cluster. Therefore, they don't have a
`spec.contextAwareResources` field.

Still, currently namespaced policies can exercise *some* of the host
capabilities, those that aren't gated by the `spec.contextAwareResources`
field: `can_i` capability, sigstore capabilities, container registry
capabilities, dns lookup, cryptographic capabilities, and others in the future.
Some of these could be abused to perform reconnaissance or information
disclosure attacks of some type: `can_i`, sigstore, container registry, dns
lookup, and possible future ones.

## User Stories

[userstories]: #userstories

### User story \#1

A Cluster Operator wants to deploy Kubewarden so teams using and self-servicing
their own Namespaces can do so securely. The Cluster Operator achieves this by
deploying dedicated PolicyServers for the needs of the teams. The policies from
the teams will only be deployable against the allowed PolicyServer(s).
Namespaced policies will only access the allowed context-aware capabilities
offered by their PolicyServer(s).

### User story \#2

A Policy User with low-permissions to deploy namespaced policies
(`AdmissionPolicies` and `AdmissionPolicyGroups`) wants to fearlessly deploy
policies in their own Namespace, in a self-service manner.

### User story \#3

An Integrator, building on top of Kubewarden, wants to use a UI or
programmatically generate namespaced policies. These namespaced policies can be
deployed securely against correctly configured PolicyServers.

### User story \#4

A Policy Developer wants to create policies that specify the needed
context-aware resources, or gracefully degrade if possible, when they don't
have permissions for specific context-aware calls.

# Detailed design

[design]: #detailed-design

## Policies

Namespaced policies (AdmissionPolicies & AdmissionPolicyGroups) must now be
scheduled to run in a secure-by-default PolicyServer.

Documentation of policies that depend on context aware calls must be updated to
mention those calls. 

Namespaced policies don't have a `spec.contextAwareResources`, therefore no
access to Kubernetes resources. This stays the same for now, see
[future work].

## Allowed context-aware calls

Each context-aware capability calls a host-call API function. These functions
are defined via a string that usually has a version and a "namespace". For
example, `kubernetes/can_i`. These strings match the host-call API call
listed in the [Host capabilities
specification](https://docs.kubewarden.io/reference/spec/host-capabilities/intro-host-capabilities).

We will use these function paths to uniquely refer to capabilities.

We will accept partial paths to these functions calls: if `oci/v2` is
provided, that will refer to all API calls under that namespace, for example,
this includes `oci/v2/verify`  or
`oci/v2/oci_manifest_config`. This is safe to implement since we control
the list of function calls.

An exception is the `kubernetes/*` calls that are already gated by expecting a
list of context-aware resource, passed via a `spec.contextAwareResources`,
since namespaced policies' CRDs don't included that spec field for now,
see [future work].

### List of context-aware calls

#### OCI:
- `oci/v1/verify`
- `oci/v2/verify`
- `oci/v1/manifest_digest`
- `oci/v1/oci_manifest`
- `oci/v1/oci_manifest_config`

#### Kubernetes

All are unversioned.

The can_i is available:
- `kubernetes/can_i`

The following are not available already for namespaced policies, hence they
aren't taken into account nor publiciced. This allows to, in the case of
cluster-wide policies, defer to the policy `spec.contextAwareResources`.
- `kubernetes/list_resources_by_namespace`
- `kubernetes/list_resources_all`
- `kubernetes/get_resource`

#### Net
- `net/v1/dns_lookup_host`

#### Crypto
- `crypto/v1/is_certificate_trusted`

#### Tracing
- `tracing/log`: Always available.

## policy-server binary

Currently, the policy-server binary reads a `policies.yml` file with the
`--policies policies.yml` flag. The policies names contain information
if they are namespaced. Its format is as follows:

```yaml
# policies.yaml
namespaced-prod-unique-service-selector: # in prod ns
  module: registry://ghcr.io/kubewarden/policies/unique-service-selector-policy:v1.0.10
pod-image-signatures: # policy group
  policies:
    sigstore_gh_action:
      module: ghcr.io/kubewarden/policies/verify-image-signatures:v0.2.8
      settings:
        signatures:
          - image: "*"
            githubActions:
            owner: "kubewarden"
    reject_latest_tag:
      module: ghcr.io/kubewarden/policies/trusted-repos-policy:v0.1.12
      settings:
        tags:
          reject:
            - latest
  expression: "sigstore_gh_action() && reject_latest_tag()"
  message: "The group policy is rejected."
```

The policy-server binary will accept a new `allowedContextAware` key for each
policy listed there. This includes the policies in the policy groups.

This new `allowedContextAware` can be set to:
- An empty object (`{}`: the policy will be allowed all context aware calls.
- An empty list (`[]`): no allowed context aware calls.
- A list of elems, with some allowed context aware calls.

Example:

```yaml
# policies.yaml
namespaced-prod-unique-service-selector: # in prod ns
  module: registry://ghcr.io/kubewarden/policies/unique-service-selector-policy:v1.0.10
  allowedContextAware:
  - kubernetes/get_resource
  - kubernetes/list_resources_all
pod-image-signatures: # policy group
  policies:
    sigstore_gh_action:
      module: ghcr.io/kubewarden/policies/verify-image-signatures:v0.2.8
      allowedContextAware:
        - oci/v2/verify
      settings:
        signatures:
          - image: "*"
            githubActions:
            owner: "kubewarden"
    reject_latest_tag:
      module: ghcr.io/kubewarden/policies/trusted-repos-policy:v0.1.12
      settings:
        tags:
          reject:
            - latest
  expression: "sigstore_gh_action() && reject_latest_tag()"
  message: "The group policy is rejected."
```


## PolicyServer Custom Resource

Add a new spec field to the PolicyServer CRD, `spec.allowedContextAwareCalls`.
Optional. Array of strings. Contains a list of context-aware host-call API
calls allowed in the policy-server Deployment. Gets validated against a known
valid list of context aware calls.

The new `spec.allowedContextAwareCalls` accepts as element an entry `all`. If
that's the case, it should be the only element. If provided, all context-aware
API calls are allowed.

## Adm Controller

The Adm Controller uses the new PolicyServer spec field
`spec.allowedContextAwareCalls` when reconciling the `policies.yaml` ConfigMap
for the policy-server:

- If `spec.allowedContextAwareCalls` contains `all`, the policies listed in the
`policies.yaml` get their `allowedContextAware` set to `{}` (empty object),
allowing all context aware calls for each policy. This provides the
functionality up until this RFC.

- If `spec.allowedContextAwareCalls` is a list (empty, or with elements), the
Controller sets the policies `allowedContextAware` key with it, giving
them zero allowed calls (empty list), or those specified in the list, verified
against the know list of available calls.

The Controller doesn't diferentiate between namespaced or clusterwide policies
when populating the `policies.yaml` ConfigMap: the allowed context-aware calls
are applied to all policies. We expect PolicyServers to be dedicated to
namespaced policies, and configured as such, with general PolicyServers that are
for cluster-wide policies to lack their `spec.allowedContextAwareCalls`.

## Mapping ClusterAdmissionPolicy

To map the Namespaces of the namespaced policies to secure PolicyServers, we
provide a new ClusterAdmissionPolicy, named `map-ns-to-policyservers`, intalled
by default.

This is just one possible mapping policy, Cluster Operators could deploy their
own using other schemes (e.g: using LabelSelectors for the mapping, etc).

This policy applies to namespaced policies and mutates their
`spec.policyServer` to match the provided settings. These PolicyServers may or
not be already created or ready; in that case the policies will stay in
`scheduled` status with no harm done.

The new policy will look like this:

```yaml
# policy.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: map-ns-to-policyservers
spec:
  module: registry://ghcr.io/kubewarden/policies/map-ns-to-policyservers:v0.1.0
  mode: enforce
  mutating: true
  rules:
    - apiGroups: ["policies.kubewarden.io"]
      apiVersions: ["*"]
      resources: ["AdmissionPolicy", "AdmissionPolicyGroup"]
      operations:
        - CREATE
        - UPDATE
  settings:
    # see mapping RFC section
```

### Mapping Namespace-to-PolicyServer

The mapping is used by the policy settings:

```yaml
# ConfigMap policy-server-mapping
settings:
  namespaceToPolicyServers:
    "team-A": policyserver-team # ns name starting with `team-A` scheduled in `policyserver-team`
    "prod-*": policyserver-prod # we support globs for ns names
  defaultPolicyServer: default-namespaced-policies   # default catch-all
```

This format would allow us do 1:1, many:1, and catch-all mappings for future
proofing.

This provides Cluster Operators with a central place for observability. We
could expose this in the future via metrics and events.

#### Ambiguous patterns

Overlapping or ambiguous patterns (e.g. `prod-*` and `*`) can have unintended
PolicyServer assignments and escalate privileges.

Therefore, we must enforce deterministic pattern matching order, most specific
first.

Using only `*` is not allowed, and is a CRD error:

```yaml
settings:
  namespaceToPolicyServers:
    "prod-*": policyserver-prod 
    "*": policyserver-A  # Not allowed. what would happen if so? do we use the default PolicyServer?
  defaultPolicyServer: default-namespaced-policies
```

## Helm Charts

We deploy our PolicyServer `default` with full permissions for cluster-wide
policies.

We deploy a new PolicyServer `for-namespaced-policies` with as restrictive
permissions as possible: zero allowed host-capabilities.
Its permissions are optionally configurable via `kubewarden-defaults` Helm
chart values.

We deploy a mutating ClusterAdmissionPolicy `map-ns-to-policyservers` with the
following settings:

```yaml
# settings of mapping policy
settings:
  defaultPolicyServer: default-namespaced-policies # default catch-all
```

We allow users to disable this policy or edit the policy settings via
`kubewarden-defaults` Helm chart Values.

We expect Cluster Operators to configure their own PolicyServers in accordance
with their needs.

## Upgrade scenario

Existing namespaced policies must be mutated by the mapping policy. For that,
we must trigger updates of all existing namespaced policies. We can do this by
adding a post-install hook that waits for the mapping policy to be active, then
updates the generation of the existing namespaced policies.

## General documentation

Add a how-to page for Cluster Operators explaining the new PolicyServer feature.
Include a list of host-call API function paths that can be used in their
configure.

Ensure the reference docs for the host-capabilities list their API calls.

## Threat Model expansion

Expand our [Threat Model](https://docs.kubewarden.io/reference/threat-model)
docs to include the promise: "If you can deploy namespaced policies, you can do
so without obtaining raised privileges".

Add the following threats & mitigations.

### Threat: Attacker uses namespaced context-aware policy to extract information

Threat:

An attacker uses a namespaced policy and a PolicyServer will full
permissions to extract information from the cluster.

Mitigation:

Cluster Operator should either not allow low-privileged users to deploy
namespaced policies, map their namespaced policies to the default
`for-namespaced-policies` PolicyServer, or configure themselves a PolicyServer
with a list of desired host-capabilities to allow.

### Threat: PolicyServer Compromise

Threat:

If a PolicyServer is compromised, all Namespaces mapped to it are at risk,
especially if the PolicyServer has broad host-capabilities.

Mitigation:

- Apply principle of least privilege to host-capabilities on PolicyServer.
- Isolate PolicyServers for sensitive Namespaces.
- Regularly audit and update PolicyServer configurations.

### Threat: Stale or Outdated custom Namespace-to-PolicyServer Mapping

Threat:  

A custom Namespace-to-PolicyServer mapping is not updated when PolicyServers or
Namespaces are added/removed, leading to policies being scheduled on unintended
PolicyServers or becoming unschedulable.

Mitigation:

- Automate mapping policy updates as part of Namespace/PolicyServer lifecycle
  management, and wait for the policy to be active before scheduling follow-up
  namespaced policies in the Namespaces.
- Periodically reconcile actual assignments with intended mappings and alert on
  drift.

# Drawbacks

UX becomes a bit more complex. Cluster Operators that don't want low-privileged
or unprivileged users with permissions to deploy namespaced policies can
opt-out by setting PolicyServers' `spec.allowedContextAwareCalls` to `all`.

# Alternatives

## Opt-in feature to simplify UX

Not every Cluster Operator uses namespaced policies. And for those that do,
they don't always allow non-privileged users RBAC to namespaced policies.
Therefore, make this RFC feature opt-in, and by default stay with status-quo.
This simplifies UX, as users don't need to care about maintaining possible
mapping.

This would break the RFC promise  "If you can deploy namespaced policies, you
can do so without obtaining raised privileges".

## A new policy-server endpoint

Instead of separating PolicyServers and mapping Namespaces to PolicyServers, we
could use a new `/validate_gated` policy-server endpoint. The new
`--allowed-context-aware` would only apply to that endpoint, and namespaced
policies would always be bound to that endpoint.

## Reduce PolicyServers RBAC

Reduce or remove the RBAC provided in each PolicyServers, including the
PolicyServer `default`. Cluster Operators must then configure each PolicyServer
(including the `default`) with their desired RBAC. This configuration is less
explicit than adding the feature provided in this RFC, and only blocks
Kubernetes context-aware capabilities, but not those for OCI registries or dns
lookups, for example.

## Check explicit permissions per host-capability

Implement explicit allow-list permissions, per context-aware feature that may
leak information. For example, check for RBAC permissions when doing
SubjectAccessReview, check for login into the OCI registries, check for general
network permissions when making networking queries, etc.

## Allow Context-aware to clusterwide policies only

This is a backwards-incompatible change. There's better approaches, like this
RFC.

# Future work

[future work]: #future-work

## Add `spec.contextAwareResources` to namespaced policies


Add new `spec.contextAwareResources` field to namespaced policies, analogous to
the same field for cluster-wide policies.

This means that policies can query for non-namespaced resources. We should only
allow querying for namesapced resources in their own namespace. And even then,
care should be taken as users may not have RBAC for such operations, but the
PolicyServers used by the policy may have enough RBAC, and this would be a form
of data exfiltration.

Right now, we have the `policies.yml` convention of naming policies as:
```
{namespaced-<ns name>-}<policy name>`
```

We could use that to obtain the Namespace that the policy can read from,
or extend `policies.yml` to annotate the policy Namespace if needed.

This would allow a new family of policies. For example, the
`unique-service-selector-policy` works by checking all Services within the same
Namespace for uniqueness, with a `list_services(service.metadata.namespace)`
(Services with the same selectors are only considered duplicates within the
same Namespace).

# Unresolved questions

[unresolved]: #unresolved-questions

- Compatibility with RFC-22 (policy lifecyle) for future proofing.  

