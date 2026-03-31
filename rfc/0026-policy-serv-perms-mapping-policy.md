|              |                                                 |
| :----------- | :---------------------------------------------- |
| Feature Name | Policy Server permissions with mapping policy   |
| Start Date   | March 11th, 2026                                |
| Category     | Context aware policies                          |
| RFC PR       | [PR](TODO)                                      |
| State        | **ACCEPTED**                                    |

# Summary

[summary]: #summary

This RFC fulfills the following promise: "If you can deploy namespaced
policies, you can do so without obtaining raised privileges". This allows
low-privileged users to self-service namespaced policies securely.

This RFC proposes a UX to gate host capabilities on the PolicyServer
side for namespaced policies. This gating will be enabled by default, and
host capabilities for namespaced policies will then be opt-in.

# Motivation

[motivation]: #motivation

Namespaced policies (`AdmissionPolicies` and `AdmissionPolicyGroups`) cannot
fetch information from the Kubernetes cluster. Therefore, they don't have a
`spec.contextAwareResources` field.

Still, currently namespaced policies can exercise *some* of the host
capabilities, those that aren't gated by the `spec.contextAwareResources`
field: `can_i` capability, sigstore capabilities, container registry
capabilities, DNS lookup, cryptographic capabilities, and others in the future.
Some of these could be abused to perform reconnaissance or information
disclosure attacks of some type: `can_i`, sigstore, container registry, DNS
lookup, and possible future ones.

## User Stories

[userstories]: #userstories

### User story \#1

A Cluster Operator wants to deploy Kubewarden so teams using and self-servicing
their own Namespaces can do so securely. The Cluster Operator achieves this by
deploying dedicated PolicyServers for their needs. Policies from
teams will only be deployable against the allowed PolicyServer(s).
Namespaced policies will only access the allowed host capabilities
offered by their PolicyServer(s).

### User story \#2

A Policy User with low-permissions to deploy namespaced policies
(`AdmissionPolicies` and `AdmissionPolicyGroups`) wants to self-service deploy
policies in their own Namespace.

### User story \#3

An Integrator, building on top of Kubewarden, wants to use a UI or
programmatically generate namespaced policies. These namespaced policies can be
deployed securely against correctly configured PolicyServers.

### User story \#4

A Policy Developer wants to create policies that specify the needed
context-aware resources and host capabilities, or gracefully degrade if
possible, when they don't have permissions for specific host capability calls.

# Detailed design

[design]: #detailed-design

## Policies

Namespaced policies (AdmissionPolicies & AdmissionPolicyGroups) must now be
scheduled to run in a secure-by-default PolicyServer.

Documentation of policies that depend on host capability calls must be updated to
mention those calls. 

Namespaced policies don't have a `spec.contextAwareResources`, therefore no
access to Kubernetes resources. This stays the same.

## Allowed host capability calls

Each host capability calls a host-call API function. These functions
are defined via a string that usually has a version and a "namespace". For
example, `kubernetes/can_i`. These strings match the host-call API call
listed in the [Host capabilities
specification](https://docs.kubewarden.io/reference/spec/host-capabilities/intro-host-capabilities).

We will use these function paths to uniquely refer to capabilities.

### List of host capability calls

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

The following are not already available for namespaced policies, hence they
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

## PolicyServer Custom Resource

Add a new specification field to the PolicyServer CRD,
`spec.namespacedPoliciesCapabilities`. Optional. Array of strings. Contains a
list of host capability API calls allowed in the policy-server Deployment for
namespaced policies. Gets validated against the known list of host capability
calls.

Its value mimics the format of Dynamic Admission Controller [match
rules](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#matching-requests-rules),
that is:

- `*`: Allow all calls, backwards-compatible.
- `oci/*`: allows all the OCI host capabilities, regardless of their version.
- `oci/v2/*`: allows all the OCI capabilities of v2.
- `oci*`: syntax not valid.
- `oci/v1/oci_*`: syntax not valid.

An exception is the `kubernetes/*` calls that are already gated by expecting a
list of context-aware resources, passed via a `spec.contextAwareResources`,
since namespaced policies' CRDs don't include that specification field.

## Adm Controller

The Adm Controller uses the new PolicyServer spec field
`spec.namespacedPoliciesCapabilities` when reconciling the `policies.yaml` ConfigMap
for the policy-server.

The format of the `policies.yaml` configuration file is described in the next
section. Suffice to say, it will be extended to mention, on a per-policy basis,
which host capabilities are granted to the policy.

When reconciling a namespaced policy, the controller will:
- determine over which Policy Server the policy is scheduled
- obtain the list of host capabilities that this instance exposes to namespaced
  policies (as stated on its CRD by the cluster admin)
- copy the list of capabilities into the policy configuration entry

That implies that all the namespaced policies running on a Policy Server will
be granted access to the same set of host capabilities; even if they don't
actually make use of them.

When reconciling a cluster wide policy, the controller will grant access
to all the host capabilities. It will do that by putting a `*` value.

In the future we can revisit this approach by adding a `hostCapabilities`
attribute to the CRDs of the cluster wide policies. This would allow the
cluster admins to give policies fine grained access to the host capabilities.

## policy-server binary

Currently, the policy-server binary reads a `policies.yml` file with the
`--policies policies.yml` flag. The configuration file looks like that:

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

We're going to extend the configuration entries of the file by adding
a new key named `host_capabilities`. This contains the list of host
capabilities that are granted to the policy.

The elements of the `host_capabilities` list follow the same conventions
used when specifying the allowed host capabilities inside of the PolicyServer spec:

- `[]`: no host capability is allowed. That's the default value
- `[ '*' ]`: all host capabilities are allowed
- `[ 'foo', 'bar' ]: only the `foo` and `bar` capabilities are allowed
- `['oci/v2/*', 'bar']: all the `ovi/v2` capabilities, plus the `bar` one are allowed

Example:

```yaml
# policies.yaml
# a cluster policy that has access to k8s resources
prod-unique-service-selector:
  module: registry://ghcr.io/kubewarden/policies/unique-service-selector-policy:v1.0.10
  host_capabilities:
  - *
namespaced-image-signatures:
  policies:
    sigstore_gh_action:
      module: ghcr.io/kubewarden/policies/verify-image-signatures:v0.2.8
      host_capabilities:
      - v2/verify
      - v1/oci_manifes_digest
      - v1/oci_manifest
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


## Helm Charts

By default, our PolicyServer `default`, will allow no host capability calls for
namespaced policies.
This is secure by default, backwards-incompatible and breaks upgrades.

Cluster Operators wishing to keep the old behavior must explicitly allow
a list of host capabilities (or `*`) using a new Value of the 
`kubewarden-defaults` Helm chart,
`.Values.policyServer.namespacedPoliciesCapabilities`, which sets the PolicyServer
`spec.namespacedPoliciesCapabilities` for the `default` PolicyServer.

## Upgrade scenario

By default, existing namespaced policies will be allowed all host
capability calls, which is backwards-compatible.

Cluster Operators must configure their own PolicyServers and a mapping policy,
or set `.Values.policyServer.namespacedPoliciesCapabilities` to `[]` to not
allow any host capability calls.

We must note that if done so, existing namespaced policies must be mutated by
the mapping policy. Cluster Operators must trigger an update for their changed
namespaced policies.

## Control scheduling of namespaced policies

A cluster administrator might want to retain full control over the
scheduling of namespaced policies to prevent users from scheduling
their policies over Policy Server instances exposing a broad list of
host capabilities.

To achieve that, we will provide a reference policy. It will be a mutating
cluster-wide policy that processes `AdmissionPolicy` and `AdmissionPolicyGroup`
resources and mutates the `spec.policyServer` field of those namespaced
policies.

The policy will determine the namespace where the
`AdmissionPolicy`/`AdmissionPolicyGroup` is defined and then it will read the
`admission.kubewarden.io/policy-server` label from the `Namespace` resource.

The value of this label will be used to enforce the Policy Server to be used by
the policy.

## Kwctl

In the future, expand `kwctl scaffold` and `kwctl run` so they take into
account the new metadata annotation to scaffold CRs and run the policy,
respectively.

## General documentation

Add a how-to page for Cluster Operators explaining the new PolicyServer feature.
Include a list of host-call API function paths that can be used in their
configuration.

Ensure the reference documentation for the host capabilities lists its API calls.

## Threat Model expansion

Expand our [Threat Model](https://docs.kubewarden.io/reference/threat-model)
docs to include the promise: "If you can deploy namespaced policies, you can do
so without obtaining raised privileges".

Add the following threats & mitigations.

### Threat: Attacker uses namespaced policy to extract information

Threat:

An attacker uses a namespaced policy and a PolicyServer will full
permissions to extract information from the cluster.

Mitigation:

Cluster Operator should either:  
* not allow low-privileged users to deploy namespaced policies
* map their namespaced policies to the default `for-namespaced-policies` PolicyServer
* configure themselves a PolicyServer with a list of desired host capabilities to allow.

### Threat: PolicyServer Compromise

Threat:

If a PolicyServer is compromised, all Namespaces mapped to it are at risk,
especially if the PolicyServer has broad host capabilities.

Mitigation:

- Apply principle of least privilege to host capabilities on PolicyServer.
- Isolate PolicyServers for sensitive Namespaces.
- Regularly audit and update PolicyServer configurations.

### Threat: Stale or Outdated custom Namespace-to-PolicyServer Mapping

Threat:  

A custom Namespace-to-PolicyServer mapping is not updated when PolicyServers or
Namespaces are added or removed, leading to policies being scheduled on unintended
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
opt-out by setting PolicyServers' `spec.namespacedPoliciesCapabilities` to `*`.

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
`--allowed-host-capability` would only apply to that endpoint, and namespaced
policies would always be bound to that endpoint.

## Reduce PolicyServers RBAC

Reduce or remove the RBAC provided in each PolicyServers, including the
PolicyServer `default`. Cluster Operators must then configure each PolicyServer
(including the `default`) with their desired RBAC. This configuration is less
explicit than adding the feature provided in this RFC, and only blocks
Kubernetes host capabilities, but not those for OCI registries or DNS
lookups, for example.

## Check explicit permissions per host capability

Implement explicit allow-list permissions, per host capability call that may
leak information. For example, check for RBAC permissions when doing
SubjectAccessReview, check for login into the OCI registries, check for general
network permissions when making networking queries, etc.

## Allow host capabilities on clusterwide policies only

This is a backwards-incompatible change. There's better approaches, like this
RFC.

## Policy metadata

Each policy is annotated with metadata. Add a new policy metadata annotation
that contains a list of allowed host capabilities. For example:

```yaml
# metadata.yml
annotations:
  io.kubewarden.policy.namespacedPoliciesCapabilities:
    - kubernetes/can_i
```

The semantics of this new annotation are the same as for the
`namespacedPoliciesCapabilities` field in `policies.yaml`.

Expand `kwctl scaffold` to print a big warning (on stderr) that Cluster
Operators should configure their PolicyServer to expose the listed host
capabilities.

Expand `kwctl run`, `kwctl bench` to accept a custom list of host capabilities
when running the policy, just as we do for contextAwareResources.

Problem: 
This creates contention when reconciling the `policies.yaml`: Should the Adm Controller
use the Policy metadata, or the PolicyServer `spec.namespacedPoliciesCapabilities`?

# Unresolved questions

[unresolved]: #unresolved-questions

- Compatibility with RFC-22 (policy lifecyle) for future proofing.  
