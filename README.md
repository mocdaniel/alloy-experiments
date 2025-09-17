# README

This repository contains various experiments regarding [OpenTelemetry](https://opentelemetry.io) (OTel) and the OTel collector distribution [Grafana Alloy](https://github.com/grafana/alloy).

More information can be found in the respective scenario's README.

---

## Scenarios

### Using Grafana Alloy instead of the OTel Collector in the OTel Demo

This scenario tries to use Grafana Alloy as drop-in replacement for the [official OTel Collector](https://opentelemetry.io/docs/collector/) in the context of the [OTel Demo Project](https://github.com/open-telemetry/opentelemetry-demo).

[More information can be found in the scenario's README](./otel-demo/README.md).

### Exploring the Grafana Alloy Web UI

One of Grafana Alloy's features that set it apart from the [official OTel Collector](https://opentelemetry.io/docs/collector/) is its Web UI that can be used for debugging, visualizing, and inspecting configured OTel pipelines.

In this scenario, a minimal setup provides a functional demo supporting multiple types of OTel signals, and guides you through the different parts of Grafana Alloy's Web UI.

[More information can be found in the scenario's README](./alloy-ui/README.md).

### Importing Remote Configuration with Grafana Alloy

Another feature of Alloy is its ability to import configuration snippets and secrets from HTTP endpoints, Hashicorp Vault, Kubernetes, or S3 object storage.

In this scenario, the same minimal setup as in the [Web UI scenario](#exploring-the-grafana-alloy-web-ui) is first converted to a standalone [custom component](https://grafana.com/docs/alloy/latest/get-started/custom_components/), then uploaded to S3 object storage and used remotely.

[More information can be found in the scenario's README](./remote-config/README.md).
