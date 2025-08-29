|              |                                  |
| :----------- | :------------------------------- |
| Feature Name | Kubewarden environment telemetry |
| Start Date   | 2025-08-28                       |
| Category     | telemetry                        |
| RFC PR       | [fill this in after opening PR]  |
| State        | **TO-BE-DEFINED**                |

# Summary

This RFC describes how the Kubewarden stack will send telemetry data about the
environment in which it is running. The goal is not to collect information
about user applications, but rather to learn more about the typical
environments where Kubewarden is deployed. This data will help guide the
project's future development.

# Motivation

The Kubewarden team currently has limited visibility into the environments
where users deploy the project. This information is invaluable for making
data-driven decisions about the project's future. For this reason, we would
like to add a feature to the Kubewarden stack that allows sending
non-sensitive, anonymous environment information to a central service for
aggregation and analysis

## Examples / User Stories

As the Kubewarden team, I want to know how many PolicyServers and policies are
typically deployed in an instance.

As the Kubewarden team, I want to know how many policies a PolicyServer usually
serves.

As the Kubewarden team, I want to know which Kubewarden versions are in use.

As the Kubewarden team, I want to know the most common Kubernetes versions used
with Kubewarden.

As a Kubewarden user, I want to help improve the project by providing anonymous
usage data without compromising my privacy or violating data protection
regulations.

As a Kubewarden user, I want to ensure that sending telemetry data does not
introduce a significant security vulnerability into my environment.

As a Kubewarden user, I want to ensure that the process of sending telemetry
data does not consume excessive resources in my environment.

# Detailed design

> [!WARNING]
> As a CNCF project, it is necessary to get approval from the CNCF to implement either of the following options.
> [Telemetry Data Collection and Usage Policy | Linux Foundation](https://www.linuxfoundation.org/legal/telemetry-data-policy)

This feature involves creating a dedicated controller reconciler
that sends telemetry data to a remote server. It requires changes in some
Kubewarden components, as described below.

## Desired metrics

The changes described in this document aim to allow the Kubewarden team to
gather the following metrics:

- Number of active PolicyServers
- Number of active policies
- Average number of policies per PolicyServer
- Number of Kubewarden installations by version
- Number of Kubewarden installations by Kubernetes version

## Controller Changes

To send information about the environment, the controller would be responsible
for collecting and transmitting the data to a remote, central place. The
controller already has access to all Kubewarden resources and can be granted
permissions to access other relevant information.

The main changes to the Kubewarden controller would be:

- A configuration option to specify the endpoint that will receive the
  information.
- A periodic job within the controller to collect and send the information.

## Controller Configuration

The controller should have two new CLI flags to enable telemetry collection and
delivery: `--stack-telemetry-endpoint` and `--stack-telemetry-period`. The first
contains the full URL of the remote server. The latter defines the interval at
which the controller sends data, defaulting to `1h`. For example:

```
manager --leader-elect ... --stack-telemetry-endpoint="https://metrics.kubewarden.io/" --stack-telemetry-period=6h ...
```

When the `--stack-telemetry-endpoint` flag is not defined, no telemetry will be
sent. The feature will be disabled by default.

## Controller Information Collection and Delivery

When the telemetry endpoint is configured, a new periodic job should be created
in the controller to collect and send the data. This job can be implemented
using the Runnable interface from the controller-runtime package and integrated
into the manager alongside the existing reconcilers.

This new periodic reconciler will start a time.Ticker that will periodically:

- Collect information about how many PolicyServers are deployed, including
  their versions.
- Collect information about how many policies are deployed, including the
  policy modules used.
- Collect the Kubewarden version in use (this can be a constant updated on
  every release).
- Collect the Kubernetes version.
- Build a JSON payload with the collected data and send it to the remote server
  using secure communication.

## Telemetry Server

The telemetry server is a custom application that receives information from
controllers and store it into a database for further analysis. This server
will expose an endpoint to receive a JSON payload with the following structure:

```
{
  "instanceUid": "unique-id-for-the-cluster",
  "kubewardenVersion": "v1.28.0",
  "kubernetesVersion": "v1.33.0",
  "policyServers": [
    {
      "uid": "policy-server-uid",
      "imageDigest": "sha256:6c7d6433524e1dd7def937e123ee8f8f8da993fdc2297efaa6640b44f3285eee",
      "policies": [
        { "moduleDigest": "sha256:44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a" },
        { "moduleDigest": "sha256:eef38c001a43600b2fb7e7d0609c8f26fbd93d8142476406988d0374d36d0bfc" }
      ]
    }
  ]
}

```

In this payload, container images and policy modules are identified by their
manifest digest to avoid exposing metadata about internal environments (like
private registries) and to prevent confusion from different tags pointing to
the same OCI artifact. However, this implies that the telemetry reconciler must
fetch this information from the OCI registry at runtime.

When the telemetry server receives the request, it will:

1. Validate the payload.
2. Infer the origin location from the source IP address.
3. Look up policy and PolicyServer versions from the OCI artifact digest.
4. Store the enriched data in a time-series database.

The telemetry server should store data to the Prometheus time series database
to be used later as a source for a Grafana dashboard. The metric should something like
the following:

```
deployed-policies{	"policy_server_id"="uid",
	"policy_server_image"="sha256:6c7d6433524e1dd7def937e123ee8f8f8da993fdc2297efaa6640b44f3285eee",
	"module"="sha256:44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a",
	"kubewarden_version"="v1.28.0",
	"instance-uid"="uid"
	"country"="<some country>",
	"city"="<some city>"
```

## Helm Chart Changes

The Kubewarden Helm charts would require new values to configure the controller's CLI flags.

```yaml
stackTelemetry:
  enabled: true
  endpoint: "https://metrics.kubewarden.io"
  period: "1h"
```

When `stackTelemetry.enabled` is `true`, the feature is enabled in the
controller via the corresponding CLI flags.

# Drawbacks

- Requires building and maintaining a custom client, a custom server, and a
  custom data pipeline.
- When new metadata is required, it necessitates code changes and a new release
  for both the controller and the telemetry server. The JSON schema versions must
  be kept in sync.
- We are solely responsible for ensuring secure communication (TLS,
  authentication, etc.) with the remote telemetry server.

# Alternatives

If we proceed with the OpenTelemetry integration, we should investigate if the
existing metrics can be updated to achieve the same goal. It might be possible
to reuse current metrics and simply add a new Otel collector pipeline to send
them to the remote collector, minimizing controller changes.

## Not selected option: OpenTelemetry Integration

This implementation option aims for a less intrusive approach, decoupling the
controller from the telemetry backend. With this design, all telemetry will be
managed outside the Kubewarden controller's reconciliation loop. The controller
will be instrumented with telemetry collection points that send data to an
OpenTelemetry collector, which will then be configured to forward this data to
a remote location.

This option is inspired by previous observability work done in collaboration
with SUSE.

### Controller Changes

The controller's `kubewarden-controller/internal/metrics`
[package](https://github.com/kubewarden/kubewarden-controller/blob/main/internal/metrics/metrics.go)
can be extended by adding new metrics to track the number of PolicyServers and
policies deployed over time. These new metrics should also contain information
about the Kubewarden and Kubernetes versions in use.

The new metric can create time series for each policy like this, using the
time-series database to group similar series:

```go
policiesGauge, err := meter.Int64Gauge("deployed-policies", metric.WithDescription("The number of deployed policies"))
if err != nil {
    log.Fatal(err)
}
// Attributes represent additional key-value descriptors that can be bound
// to a metric observer or recorder.
commonAttrs := []attribute.KeyValue{
    attribute.String("policy_server_id", policy.GetPolicyServer().GetUID()),
    attribute.String("policy_server_image", policy.GetPolicyServer().GetImageDigest()),
    attribute.String("module", policy.GetModuleDigest()),
    attribute.String("kubewarden_version", KUBEWARDEN_VERSION),
    attribute.String("instance_uid", getInstanceUID()), // Corrected key name for consistency
}

policiesGauge.Record(context.Background(), 1, metric.WithAttributes(commonAttrs...)) // Corrected context.Background()
```

Once the metric is recorded, it is up to the OpenTelemetry collector to
manipulate the data and send it to a remote location, which can be another
OpenTelemetry collector or a time-series database (e.g., Prometheus).

### OpenTelemetry Configuration

In the OpenTelemetry collector configuration, a new pipeline should be added to
redirect the new controller metric to a remote location for further analysis.
This is an example of an initial configuration used to forward the metric to a
remote OpenTelemetry collector.

```yaml
receivers:
  otlp:
    protocols:
      grpc: {}
processors:
  # Control how much time we can wait before sending the metric to the remote Otel collector
  batch:
    timeout: 1h
  # Drop any metrics not called "deployed-policies"
  filter/dropInternalMetrics:
    error_mode: ignore
    metrics:
      metric:
        - 'name != "deployed-policies"' # metrics not called deployed-policies will be dropped
  attributes:
    - key: kubewarden_version
      action: upsert
      value: v1.28.0 # this value can be defined in the Helm chart installation
    # Note: The Kubernetes version can often be populated automatically by an Otel resource detector.
    - key: kubernetes_version
      action: upsert
      value: v1.33.0
exporters:
  otlphttp/remoteKubewardenOtel:
    auth:
      authenticator: bearertokenauth
    endpoint: https://metric.kubewarden.io
service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [attributes, filter/dropInternalMetrics, batch]
      exporters: [otlphttp/remoteKubewardenOtel] # Corrected to match exporter name
```

This OpenTelemetry configuration can be used to control the telemetry export
and can be expanded over time to include more metadata by leveraging the rich
ecosystem of OpenTelemetry
[processors](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor),
potentially with no changes to the controller code. This makes telemetry
customization more flexible. In the example above, processors are used to add
labels, filter unrelated metrics, and batch metrics before sending.

All OpenTelemetry collector configuration can be managed in the Helm chart
installation/update. The remote endpoint would be provided in the Helm Chart
values. If the `stackTelemetry.enabled` is disabled, the OpenTelemetry
configuration will not be applied:

```yaml
stackTelemetry:
  enabled: true
  remoteOtelEndpoint: "https://metrics.kubewarden.io"
  batchTimeout: "1h"
```

The remote OpenTelemetry collector's configuration is abstracted from the user.
For instance, it could use the [GeoIP
Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/geoipprocessor)
to add location information to the received metrics.

# Unresolved Questions

# Reference

[metric package - go.opentelemetry.io/otel/metric - Go Packages](https://pkg.go.dev/go.opentelemetry.io/otel/metric#Int64Gauge)

[go.opentelemetry.io/otel/attribute](https://pkg.go.dev/go.opentelemetry.io/otel/attribute)

[opentelemetry-collector-contrib/processor/filterprocessor/README.md at main · open-telemetry/opentelemetry-collector-contrib · GitHub](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/filterprocessor/README.md)

[opentelemetry-collector-contrib/processor/geoipprocessor at main · open-telemetry/opentelemetry-collector-contrib · GitHub](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/geoipprocessor)

[opentelemetry-collector-contrib/processor at main · open-telemetry/opentelemetry-collector-contrib · GitHub](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor)

[opentelemetry-collector/processor at main · open-telemetry/opentelemetry-collector · GitHub](https://github.com/open-telemetry/opentelemetry-collector/tree/main/processor)
