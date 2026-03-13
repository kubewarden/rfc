|              |                                                 |
| :----------- | :---------------------------------------------- |
| Feature Name | Policy Server permissions with mapping policy   |
| Start Date   | March 11th, 2026                                |
| Category     | Context aware policies                          |
| RFC PR       | [PR](TODO)                                      |
| State        | **ACCEPTED**                                    |

# Summary

[summary]: #summary

This RFC aims to implement the following promise: "If you can deploy namespaced
policies, you can do so without obtaining raised privileges". This allows users
to self-service namespaced policies securely.

This RFC proposes an implementation and UX to also gate context-aware
capabilities in the PolicyServer side for namespaced policies. This gating will
be enabled by default, and host-capabilities for namespaced policies will be
opt-in from then on.

No proposed change for cluster-wide policies (`ClusterAdmissionPolicies` and
`ClusterAdmissionPolicyGroups`).

# Motivation

[motivation]: #motivation

By design, namespaced policies (`AdmissionPolicies` and
`AdmissionPolicyGroups`) cannot fetch information from the Kubernetes cluster.
Therefore, they don't have a `spec.contextAwareResources` field.

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

A Cluster Operator wants to deploy Kubewarden in a safe manner, so teams using
and self-servicing their own Namespaces can do so securely. The Cluster
Operator achieves this by deploying secure PolicyServers for the needs of the
teams. The policies from the teams will only be deployable against the allowed
PolicyServer(s). Namespaced policies will only access the allowed context-aware
capabilities offered by their PolicyServer(s).

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

### User story \#5

A Cluster Operator wants to configure PolicyServers used for cluster-wide
policies, so they are limited in which host-capabilities they can use.

# Detailed design

[design]: #detailed-design

## List of allowed context-aware capabilities

Each context-aware capability calls a host-call API function. These functions
are defined via a string that usually has a version and a "namespace". For
example, `v1/kubernetes/can_i`. These strings match the host-call API call
listed in the [Host capabilities
specification](https://docs.kubewarden.io/reference/spec/host-capabilities/intro-host-capabilities).

We will use these function paths to uniquely refer to capabilities.

We will accept partial paths to these functions calls: if `v1/kubernetes` is
provided, that will refer to all API calls under that namespace, for example,
this includes `v1/kubernetes/get_resource`  or
`v1/kubernetes/list_resources_all`. This is safe to implement since we control
the list of function calls.

## policy-server binary

By default, for security reasons, the policy-server will not allow context
aware calls (this is backwards-incompatible).

The policy-server binary will accept a new optional `--allow-all-context-aware`
flag that allows all host-capability calls.

The policy-server binary will accept a new optional `--allowed-context-aware`
flag to provide it with a list of host-capabilities calls. If provided, this
flag has precedence over `--allow-all-context-aware` flag. The policy-server
will only execute these host-capabilities.

This new `--allowed-context-aware` flag expects 1 host-capability call
identifier.

For example:

```console
--allowed-context-aware v1/manifest_digest
```

Several apparitions of this flag are accepted and aggregated together:

```console
--allowed-context-aware v1/manifest_digest \
--allowed-context-aware v2/verify \
--allowed-context-aware v1/kubernetes/can_i
```

If a policy is active, and makes use of a capability that it is not
allowed to, the policy-server will error on the policy execution and log the
attempted unprivileged use, just as it does now. By default, webhooks fail
closed, so the policy will reject on error.

## PolicyServer Custom Resource

Expose the new feature to the PolicyServers Custom Resources via new fields.
They are mutually exclusive which is enforced by our validating webhook:

- `spec.allowAllContextAware`. Boolean. Defaults to `false`. Setting it to
  `true` would provide the functionality up until this RFC.  
- `spec.allowedContextAware`. Optional. Array of strings.Contains a list of
  context-aware host-call API calls allowed in the policy-server Deployment. For
  example:

```yaml
spec:
  allowedContextAware:
    - v1/manifest_digest 
    - v2/verify 
    - v1/kubernetes/can_i
```

This new feature is designed for PolicyServers used for namespaced policies,
but it is optionally available for all PolicyServers.

## Mapping ClusterAdmissionPolicy

To map the Namesapces to PolicyServers of all the namespaced policies, we
provide a new default mutating ClusterAdmissionPolicy, named
`map-ns-to-policyservers`. The policy is installed by default.

This policy checks for all namespaced policies and mutates their
`spec.policyServer` to match the policy settings. These PolicyServers may or
not be already created or ready; in that case the policies will stay in
`scheduled` status.


```yaml
spec:
  mutating: true
  rules:
    - apiGroups: ["policies.kubewarden.io"]
      apiVersions: ["*"]
      resources: ["ClusterAdmissionPolicy", "ClusterAdmissionPolicyGroup"]
      operations:
        - CREATE
        - UPDATE
  settings:
    # see mapping RFC section
```


## Mapping Namespace-to-PolicyServer

The mapping is used by the policy settings:
the sort:

```yaml
# ConfigMap policy-server-mapping
settings:
  namespaceToPolicyServers:
    "team-A": policyserver-team # ns name starting with `team-A` scheduled in `policyserver-team`
    "prod-*": policyserver-prod # we support globs for ns names
  defaultPolicyServer: policyserver-namespaced-policies   # default catch-all
```

This format would allow us do 1:1, many:1,, and catch-all mappings for future
proofing.

This provides Cluster Operators with a central place for observability. It
would be good to expose this via metrics and events.

### Ambiguous patterns

Overlapping or ambiguous patterns (e.g. `prod-*` and `*`) can have unintended
PolicyServer assignments and escalate privileges.

Therefore, we must enforce deterministic pattern matching order, most specific
first.

Using only `*` is not allowed:

```yaml
settings:
  namespaceToPolicyServers:
    "prod-*": policyserver-prod 
    "*": policyserver-A  # what happens now? do we use the default PolicyServer?
  defaultPolicyServer: policyserver-namespaced-policies
```

### Race conditions during mapping updates

Simultaneous updates to the mapping could result in inconsistent assignments.
We must reconcile webhooks after each mapping state change.


## Helm Charts

We deploy our PolicyServer `default` with full permissions for cluster-wide
policies.

We deploy a new PolicyServer `namespaced-policies` with as restrictive
permissions as possible, by default. This means zero allowed host-capabilities.
Its permissions are optionally configurable via `kubewarden-defaults` Helm
chart values.

By default, we deploy a mutating ClusterAdmissionPolicy
`map-ns-to-policyservers` with the following settings:

```yaml
# settings of mapping policy
settings:
  defaultPolicyServer: policyserver-namespaced-policies # default catch-all
```

We allow users to disable this policy or edit the policy settings via
`kubewarden-defaults` Helm chart Values.

We expect Cluster Operators to configure their own PolicyServers in accordance
with their needs.

## Upgrade scenario

With this change, namespaced policies may become unschedulable because their
desired `spec.policyServer` is not a valid one.

Existing namespaced policies need to mutated by the mapping policy. We can
add a post-install hook that waits for the mapping policy to be active,
and then updates all namespaced policies.

## General documentation

Add a how-to page for Cluster Operators explaining how to configure
PolicyServers to expose capabilities. Include a list of host-call API function
paths that can be used in their configure.

Ensure the reference docs for the current host-capabilities calls list their
host-call API function calls.

## Threat Model expansion

Expand documentation of our [Threat
Model](https://docs.kubewarden.io/reference/threat-model) to include this new
promise. Add possible threats, such as users deploying vulnerable namespaced
policies, and mitigations, such as cluster operators configuring PolicyServers
correctly.

Add the following threats & mitigations.

### Threat: Attacker uses namespaced context-aware policy to extract information

Threat:

An attacker uses a namespaced policy and a PolicyServer will full
permissions to extract information from the cluster.

Mitigation:

Cluster Operator should either not allow low-privileged users to
deploy namespaced policies, or configure the PolicyServer with a list of
desired host-capabilities to allow.

### Threat: Scheduling policy in incorrect PolicyServer

Threat:

A low-privileged user tries to schedule a privileged policy against a
PolicyServer they should not be allowed to.

Mitigation:

The Cluster operator ensures the mutating ClusterAdmissionPolicy that we
install by default is installed and active.

### Threat: PolicyServer Compromise

Threat:

If a PolicyServer is compromised, all Namespaces mapped to it are at risk,
especially if the PolicyServer has broad host-capabilities.

Mitigation:

- Minimize host-capabilities per PolicyServer.  
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
- Periodically reconcile actual assignments with intended mappings and alert on drift.

### Threat: Misconfiguration of Namespace-to-PolicyServer Mapping

Threat:  

Operator error in the mapping could assign sensitive Namespaces to
less-restricted PolicyServers.

Mitigation:  

Validate mapping changes, provide tooling for safe updated, audit mapping
changes.

## Examples

The examples are listed under our Threat Model expansion.

# Drawbacks

UX becomes a bit more complex. Cluster Operators that don't want low-privileged
or unprivileged users can still opt-out by setting PolicyServers'
`spec.allowAllContextAware` to `true`.

Adding a new `/validate_gated` complicates our Admission Controller and overall
architecture.

# Alternatives

## Opt-in to simplify UX

Not every Cluster Operator uses namespaced policies. And for those that do,
they don't always allow non-privileged users RBAC to namespaced policies.
Therefore, make the RFC feature opt-in, and by default stay with status-quo.
This simplifies UX, as users don't need to care about maintaining possible
mapping.

This would break the RFC promise  "If you can deploy namespaced policies, you
can do so without obtaining raised privileges".

## Map with ConfigMap

To map Namespaces to PolicyServers, the Controller reads a ConfigMap and
reconciles policies as needed.

Advantages:  

We can always reconcile a namespaced policy if the mappings change. We cannot
do that with the mapping policy, if the namespaced policy is not updated.

Disadvantages:  

Less flexible. Still, users can always not use the ConfigMap and use their
own mapping policy if desired.

## Support Label selectors for mapping

The Controller matches Namespace labels to assigned PolicyServers, by the
provided mapping configuration:

```yaml
data:
  labelSelectorToPolicyServers:
    "env=prod": [policyserver-prod]
    "team=foo": [policyserver-team-foo]
```

## Map with Namespace labels

To map Namespaces to PolicyServers, use Labels in Namespaces. This is simpler
than other mappings. However:

- Mapping would be 1:1 or 1:many-PolicyServers, but no many-Namespaces:1.
  Unless we duplicate label values and add complexity.  
- Changing labels may trigger other controllers and policies.  
- Threat of escalation if user uses a label that matches a pattern mapped to a
  more privileged PolicyServer.

## Map with new PolicyServer `spec.namespaceMappings`

Each PolicyServer has their mappings saved in a `spec.namespaceMappings`. We
allow globs. The Controller validates that the several mappings between all
PolicyServers don't have conflicts, and if so, it errors on create/update (this
may introduce racy errors when creating several PolicyServers at once).

## Map with new PolicyServerMapping Custom Resource

Same as the ConfigMap approach, but using a Custom Resource. Allows us to
validate the format easier on its creation/update.

## A new policy-server endpoint

Instead of separating gated-only PolicyServers and mapping Namespaces to
PolicyServers, we could use a new `/validate_gated` policy-server endpoint. The
new `--allowed-context-aware` would only apply to that endpoint, and namespaced
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

## Reduce Context-aware to clusterwide policies only

This is a backwards-incompatible change. There's better approaches, like this
RFC.

# Unresolved questions

[unresolved]: #unresolved-questions

- There's no way to now at policy instantiation time of what capabilities a
  policy makes use of, and we cannot trust self-reporting. This means that policy
  status will never show a `lacks-permissions` to the user or similar. Such
  policies will fail on execution time instead.  
- Compatibility with RFC-22 (policy lifecyle) for future proofing.  
- Compatibility with [Raw policies](https://docs.kubewarden.io/howtos/raw-policies).

