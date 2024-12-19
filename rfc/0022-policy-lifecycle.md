|              |                                 |
| :----------- | :------------------------------ |
| Feature Name | [Name]                          |
| Start Date   | [Today]                         |
| Category     | [Category]                      |
| RFC PR       | [fill this in after opening PR] |
| State        | **ACCEPTED**                    |

# Summary

[summary]: #summary

This RFC introduces a revised policy lifecycle for Kubewarden, enabling users to monitor the status of a policy on a specific Policy Server replica.
The new lifecycle enhances the Policy Server's resilience to failures and facilitates seamless policy updates without requiring server restarts.
Additionally, the user will be able to rollback to a previous policy version in case of issues with the latest policy.

# Motivation

[motivation]: #motivation

This change addresses a critical issue with the Policy Server.
Currently, if a policy fails to load—due to reasons such as an inability to download it from the registry or incorrect settings—the Policy Server exits with an error,
causing the pod to enter a CrashLoopBackOff state.
Resolving this requires inspecting the error messages from the Policy Server pod and fixing the underlying issue.
However, a user might have the permissions to create a policy but lack the necessary permissions to check the logs of the Policy Server, making it challenging to diagnose and resolve the problem.
Additionally, when updating a policy, there is no status reported to indicate the failure. The policy remains in the Active state even while the Policy Server is stuck in a crash loop.
This creates a significant risk, as new Policy Server pods might fail to start, and the old ones running the previous functional configuration could be lost if their node becomes unavailable.
This situation can disrupt the cluster, as all incoming admission requests will be denied in the absence of operational Policy Server instances.

The proposed changes aim to address these issues by introducing a new policy lifecycle that includes the following features:

- Hot reload of policies: The Policy Server will be able to load a new policy without requiring a restart.
- Policy validation: The Policy Server will validate the policy before loading it, ensuring that it is correctly compiled and can be loaded.
- Policy status monitoring: The Policy Server will report the status of the policy, indicating whether it was loaded successfully or if an error occurred.
- Policy versioning: The Policy Server will keep running with a previous version of the policy in case of issues with the latest policy.
- Policy rollback: The user will be able to rollback to a previous policy version in case of issues with the latest policy.

## Examples / User Stories

[examples]: #examples

- As a user, I want to load a new policy/update an existing policy without requiring a restart of the policy server.
- As a user, I want to know if a policy was pulled or validated successfully.
- As a user, I want to monitor the status of a policy on specific policy server replica so that I can identify and resolve issues with the policy.
- As a user, I want that the policy server keeps running with a previous version of the policy in case of issues with the latest policy.
- As a user, I want to rollback to a previous policy version in case of issues with the latest policy.

# Detailed design

[design]: #detailed-design

## Concepts

To implemenet the new policy lifecycle, we will introduce the following concepts:

### The Policy Server acts as a controller

At the time of writing the Policy Server is configured with a ConfigMap that contains all the policies to be loaded.
When a new policy is created, the ConfigMap is updated, and the Policy Server is restarted to load the new policy.

We want to change this behavior to allow the Policy Server to act as a controller that can load new policies without requiring a restart.
The Policy Server will be watching the Policy CRD directly and it will change the internal state of the evaluation environment when a new policy is created or updated.

A leader election mechanism will be implemented to ensure that only one Policy Server instance is acting as a controller at a time.
The leader will be responsible to pull the policies from the registry, validate the policy settings, precompile the policy and store it in a shared cache.
After the policy is precompiled, the leader will change the status of the policy to Valid and signal the other replicas that a new policy is available to be loaded.
The replicas will be watching the Policy CRDs and load the new policy when the status is `Valid`.
In case of an error, the policy status will be updated, but the Policy Server will continue serving the previous valid policy until the issue is resolved.

### Shared Policy Cache

The Policy Server will have a shared cache where the precompiled policies will be stored.
We can use a RWX Persistent Volume to store the cache and share it among all the Policy Server replicas.

### Policy Versioning

We can use the `generation` field of the Policy CRD to version policies. When a new policy is created or updated, Kubernetes automatically increments the `generation` field.

The current policy will be stored in another CRD, similar to the `Policy` CRD, but with a different name, such as `PolicyReplica`.
Only one `PolicyReplica` will be kept at any time. When a Policy Server replica restarts or a new replica is created, it will load the policy from the `PolicyReplica` CRD.

The `PolicyReplica` is always assumed to be valid, as it has already been validated by the leader.
If a Policy Server replica fails to load the policy from the `PolicyReplica` CRD, it will exit with an error.
Since Kubernetes removes the pod from the service endpoints during a restart or a termination, the Policy Server service will remain consistent, ensuring that no server operates without a valid policy.

Additionally, the policy validation endpoint will include the `generation` number in its URL.
When all Policy Server replicas are ready with the new `PolicyReplica`, the validation endpoint will switch to the new `generation` number.
Once this transition is complete, the `PolicyReplica` (current state) and the `Policy` (desired state) will be identical, ensuring consistency across all Policy Server replicas.
This approach guarantees that the Policy Server will always have a valid policy to load and avoids any service interruptions during updates.

### The Policy Server is a StatefulSet

The Policy Server will be deployed as a StatefulSet to ensure that each replica has a unique identity.
This will allow us to identify the status of a policy on a specific Policy Server replica.

### Policy Status

The Policy CRD status will be updated to support the following Status Conditions:

From the leader perspective:

- `Pending`: The leader has received the event and is starting the policy loading process.
- `Valid`: The policy was successfully loaded and is ready to be used.
- `Invalid`: The policy failed to load due to an error with the policy settings.
- `PullError`: The leader failed to pull the policy from the registry.
- `Pending`: The policy is being loaded.
- `Ready` (replica number): The policy was successfully loaded and is ready to be used.
- `Active` (generation number): The webhook is configured to the desidered policy generation.

We can use the [status condition](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties) convention to report the status of the policy.
We will extend the status condition struct to include the replica number.
The approach of extending the status condition struct is already used in the Kubernetes codebase,
for example in the [Pod conditions](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-conditions).

### Rollback

If a policy fails to load, the Policy Server will keep running with the previous valid policy.
The user will be able to rollback to a previous policy version by updating the `Policy` CRD (via a spec field or an annotation, TBD).
The controller will detect the change and use the `PolicyReplica` CRD to update the Policy CRD with the previous configuration.

## Policy Lifecycle

Given the concepts described above, the policy lifecycle will be as follows:

| Description                                                                                                                                                                 | Condition                    |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| The user creates or updates a policy by creating or modifying the `Policy` CRD.                                                                                             | `Created/Updated`            |
| The Policy Server leader receives the event and starts the policy loading process. The policy status is set to `Pending`.                                                   | `Pending`                    |
| The leader successfully pulls the policy from the registry, validates the settings, precompiles the policy, and stores it in the shared cache.                              | `Valid`                      |
| The leader encounters an error while loading the policy. The status is set to `Invalid`, and the webhook will not be configured.                                            | `Invalid`                    |
| The leader fails to retrieve the policy from the registry. The status is set to `PullError`, and the webhook will not be configured.                                        | `PullError`                  |
| The Policy Server replica successfully loads the policy and updates the status to `Ready` with the replica number. The webhook can then be configured.                      | `Ready (replica number)`     |
| The controller configures the webhook to validate the policy. The status is updated to indicate the policy generation being served, which includes the `generation` number. | `Active (generation number)` |

NOTE: If there is a failure during an update, the Policy Server will keep running with the previous valid policy.

# Drawbacks

[drawbacks]: #drawbacks

<!---
Why should we **not** do this?

  * obscure corner cases
  * will it impact performance?
  * what other parts of the product will be affected?
  * will the solution be hard to maintain in the future?
--->

# Alternatives

[alternatives]: #alternatives

<!---
- What other designs/options have been considered?
- What is the impact of not doing this?
--->

# Unresolved questions

[unresolved]: #unresolved-questions

<!---
- What are the unknowns?
- What can happen if Murphy's law holds true?
--->
