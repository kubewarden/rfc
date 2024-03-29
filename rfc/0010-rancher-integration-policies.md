|              |                                  |
| :----------- | :------------------------------- |
| Feature Name | Rancher integration of Kubewarden Policies  |
| Start Date   | 2022-08-22                       |
| Category     | Development                      |
| RFC PR       | https://github.com/kubewarden/rfc/pull/11  |
| State        | **ACCEPTED**                     |


# Summary
[summary]: #summary

This RFC explains how the Kubewarden policies, either owned by the Kubewarden
team or 3rd party ones, can be integrated in Rancher Explorer.

For policies owned by the Kubewarden team it proposes building Rancher Helm charts
that will be published under https://github.com/rancher/charts.

For 3rd party policies not owned by the Kubewarden team,
it proposes using the Artifact Hub API to poll for the metadata stored in
`policy/artifacthub-pkg.yml`.


# Motivation
[motivation]: #motivation

Problems to solve:

- Obtain the list of available policies regardless of where they come from, and
  their metadata to show in Rancher Explorer UI.
- Allow for groups of policies to be installed.
- Implement it efficiently and with low development maintenance costs.
- Allow for airgap installations.

## Examples / User Stories
[examples]: #examples

1. As a user, I want to install one Kubewarden policy owned by the
   Kubewarden/Rancher team via Rancher Explorer UI.
2. As a Kubewarden developer, I want to allow users to deploy a collection of related policies
   in a simple way. For example: "PSP best of", "Security best practices",...
3. As a user, I want to install 3rd party Kubewarden policies via Rancher
   Explorer UI.
4. As a Kubewarden developer, I want to release a new Kubewarden chart/policy so
   it can be installed via Rancher Explorer UI.
5. As a 3rd party developer, I want to release a policy on Artifact Hub so it can
   be discovered and installed via Rancher Explorer UI.

# Detailed design
[design]: #detailed-design

## For policies owned by the Kubewarden team

Follow the process listed in [RFC-9, Rancher integration of Kubewarden
charts](./0009-rancher-integration-charts.md).

Submit all Helm charts of the policies to
https://github.com/kubewarden/helm-charts.

Even if the charts will not be released to users, the repository can be consumed
in Rancher UI (using git URL and branch) for testing the chart itself.

For policy bundles, they can be in a new chart (e.g: see `kubewarden-defaults`
chart).

The resulting policy charts will not be served and released by us, but used to
build Rancher charts from source by using the
[Package](https://github.com/rancher/charts/blob/dev-v2.6/docs/packages.md)
format in https://github.com/rancher/charts.
Policy charts will be submitted via a
`packages/kubewarden/<policy>.package.yaml`, with `package.yaml::commit`
pointing to the relevant `rancher-X` branch. Rancher charts are build from those
`package.yaml` files.

A pull request bot needs to be created to make the submissions to either
https://github.com/rancher/charts or our own staging fork, by bumping the
`package.yaml::commit`.

## For 3rd party artifacts

Use the provided [artifacthub.io public API](https://artifacthub.io/docs/api/).
Currently, the Artifact Hub API provides [`GET
/packages/search`](https://artifacthub.io/docs/api/#/Packages/searchPackages),
where one can set the `repository kind` to `Kubewarden policies`. With this call
one can obtain the list of Kubewarden policies. One would then query for each of
the policies with [GET
/packages/kubewarden/{repoName}/{packageName}](https://artifacthub.io/docs/api/#/Packages/getKubewardenPoliciesDetails),
and obtain its `name`, `version`, `signed` status, and `data/kubewarden-*`
information.

Artifacts owned by the Kubewarden team are also present in Artifact Hub. Still,
they have different provenance, therefore, different support assurances. Hence,
they should be able to coexist.

# Drawbacks
[drawbacks]: #drawbacks

## For policies owned by the Kubewarden team

Needing new Helm charts to ship policies owned by the Kubewarden team.

## For 3rd party policies

Using the provided Artifact Hub API, given that it doesn't have pagination and
one needs to query for each package independently, will be inefficient. A cache
could be implemented.

The following metadata is not obtainable without reading the Wasm binary right
now, therefore should be added to `policy/artifacthub-pkg.yml`:
- Metadata not present in `policy/artifacthub-pkg.yml` yet present in
  `policy/metadata.yml`, namely the array of `rules`.
- `questions.yml` information. Not a problem per se, as 3rd party policies would
  not need Rancher integration.

Airgap installations need to catch the Artifact Hub metadata somehow.

# Alternatives
[alternatives]: #alternatives

## A. Using the metadata stored in the custom section of Wasm modules

It could be possible to save the `questions.yml` metadata in the metadata
annotations of `annotated-policy.wasm` themselves (the custom section of the
Wasm module containing raw data, for us with the name
`KUBEWARDEN_CUSTOM_SECTION_METADATA`). `rules` metadata is present there
already.

With this, annotated policies will contain almost all metadata needed.
Information about if the policy is signed couldn't be included there by
definition.

Then, Rancher Explorer could make use of a client for OCI registries
(such as [oci-registry-js](https://www.npmjs.com/package/oci-registry-js) or a
wasm-compiled Rust library for example), to pull and cache each Wasm module.
With [`WebAssembly.Module.customSections()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Module/customSections)
it would be possible to extract all the metadata from the annotations of the
policy.

This would mean to pull hundreds of megabytes, as it will download the Wasm
module of the policies, regardless of pagination and caching.

Airgap installations would consist on mirroring the OCI registry (since metadata
in stored in the Wasm modules).

## B. Storing policy metadata in the OCI registry

Same as alternative (A), but storing the policies' metadata in the OCI registry
either under:
- A new layer, with `config.mediaType` [creating an artifact type
`application/vnd.oci.yaml-sample.config.v3+yaml`](https://github.com/opencontainers/artifacts/blob/main/artifact-authors.md#defining-a-unique-artifact-type).
  This means pulling layers, and possibly uncompressing them.
- Or in `manifest.config`
  (https://github.com/opencontainers/artifacts/blob/main/artifact-authors.md#optional-defining-config-schema).
  Which is not compressed, and whose checksum matches the checksum of the
  uncompressed contents of the `manifest.config`.

It would then be retrieved with an OCI registry client. This would be more
efficient than pulling full Wasm modules for all policies as in (A).

This means policies would have the metadata saved in both:
- Wasm module, as of today. Used by `kwctl`, `policy-server`.
- OCI registry `manifest.config`.

This would mean that policy-related metadata can be dropped from
`policy/artifacthub-pkg.yml`, which should only contain things related to
Artifact Hub.

Airgap installations would consist on mirroring the OCI registry.
Rancher airgap works by providing a list of images to be mirrored, and
instructions for using a private registry. We would need to instruct Kubewarden
users to use an additional OCI private registry, and an OCI-capable client for
performing the mirroring.

This alternative provides the same implementation for both kubewarden-owner
policies, Rancher-mirrored policies, and 3rd party policies. Just serve each of
them on their respective OCI registries.

A drawback could be finding frontend-capable OCI registry clients that can deal
with these formats, and can authenticate against private registries too.


# Unresolved questions
[unresolved]: #unresolved-questions

As in RFC-9, all needed images for the charts need to be mirrored in the Rancher
DockerHub repository, instead of being consumed from upstream. Since policies
are Wasm modules, we need to raise the request to provide a Rancher OCI
registry, use Kubewarden's upstream one, or else.
