# README

This scenario tries to use Grafana Alloy as drop-in replacement for the [official OTel Collector](https://opentelemetry.io/docs/collector/) in the context of the [OTel Demo Project](https://github.com/open-telemetry/opentelemetry-demo).

The goal is a reproduction of the observability capabilities demonstrated in the OTel demo, leveraging Alloy and
its available features.

In order to follow along, **multiple steps** are provided (see below). Follow along by going through the commands listed in each step.

---

## Prerequisites

Clone the OTel demo repository to this subdirectory, start the demo once to pull/build all needed images, and start at [Step One](#step-1) below.

> [!NOTE]
> Building some of the images might take some time. This step needs to be done only once.

```bash
git clone --depth=1 --single-branch https://github.com/open-telemetry/opentelemetry-demo.git
cd opentelemetry-demo
make start
cd ..
```

---

## Step 1

First, let's try running Grafana Alloy in `otelcol` mode, directly ingesting the existing OTel collector config in [`./opentelemetry-demo/src/otel-collector/otelcol-config.yml`](./opentelemetry-demo/src/otel-collector/otelcol-config.yml).

You can inspect the adjustments to the `docker-compose.yml` with below command:

```bash
diff opentelemetry-demo/docker-compose.yml step-1/compose.yml
```

Then, copy the provided `compose.yml` over to the demo directory, and restart the `otel-collector` service **in the foreground** to see Alloy's output:

```bash
cp step-1/compose.yml opentelemetry-demo/docker-compose.yml
docker compose -f opentelemetry-demo/docker-compose.yml up otel-collector
```

You will see output like this:

```
otel-collector  | Error: reading config path "/etc/otelcol-config.yml": (Critical) failed to get otelcol config: cannot unmarshal the configuration: decoding failed due to the following error(s):
otel-collector  | 
otel-collector  | 'receivers' unknown type: "redis" for id: "redis" (valid values: [jaeger otlp datadog opencensus solace tcplog awscloudwatch kafka splunk_hec vcenter zipkin influxdb syslog faro filelog filestats])
otel-collector  | 'processors' unknown type: "resourcedetection" for id: "resourcedetection" (valid values: [transform batch cumulativetodelta deltatocumulative filter groupbyattrs interval k8sattributes memory_limiter attributes probabilistic_sampler span tail_sampling])
otel-collector  | 'exporters' unknown type: "opensearch" for id: "opensearch" (valid values: [debug faro loadbalancing otlp otlphttp splunk_hec syslog datadog googlecloud kafka])
otel-collector exited with code 1
```

Alloy complains about unknown `receiver`/`processor`/`exporter` types. This happens because Alloy can only process `otelcol`-formatted configuration with definitions for pipeline objects for which an [Alloy component](https://grafana.com/docs/alloy/latest/reference/components/) exists.

You will tweak the `otelcol`-formatted configuration in [Step 2](#step-2).

## Step 2

In order to get rid of the errors in Alloy, you need to **remove** all pipeline components defined in [`./opentelemetry-demo/src/otel-collector/otelcol-config.yml`](./opentelemetry-demo/src/otel-collector/otelcol-config.yml) for which no corresponding [**Alloy components**](https://grafana.com/docs/alloy/latest/reference/components/) exist. 

In the case of the OTel demo, those are:


- **Receivers**
  - `docker_stats`
  - `hostmetrics`
  - `httpcheck`
  - `nginx`
  - `postgresql`
  - `redis`
- **Exporters**
  - `opensearch`
- **Processors**
  - `resourcedetection`[^1]

[^1]: While there _is_ an Alloy component for the `resourcedetection` processor, it is lacking the wrapper code needed to translate it from `otelcol` format into Alloy's DSL. Thus it needs to be removed for now, as well.

An adjusted `otelcol-config.yml` has been provided in [`./step-2/otelcol-config.yml`](./step-2/otelcol-config.yml). You can inspect the adjustments with below command:

```bash
diff opentelemetry-demo/src/otel-collector/otelcol-config.yml step-2/otelcol-config.yml
```

Then, copy the adjusted version over to the demo directory:

```bash
cp step-2/otelcol-config.yml opentelemetry-demo/src/otel-collector/otelcol-config.yml
```

> [!NOTE]
> This step removes _a lot_ of the observability capabilities of the OTel demo - you are going to add them back in a later step.

Next, restart the `otel-collector` service to load the new configuration with Alloy:

```bash
docker compose -f opentelemetry-demo/docker-compose.yml up otel-collector
```

You will see output like this:

```
otel-collector  | Error: /etc/otelcol-config.yml:65:1: component "otelcol.exporter.debug" is at stability level "experimental", which is below the minimum allowed stability level "generally-available". Use --stability.level command-line flag to enable "experimental" features
otel-collector  | 
otel-collector  | 64 | 
otel-collector  | 65 | otelcol.exporter.debug "default" {
otel-collector  |    | ^^^^^^^^^^^^^^^^^^^^^^
otel-collector  | 66 |     verbosity = "Basic"
otel-collector  | 
```

Alloy now complains about the wrong _stability level_ in order to use the **experimental** `debug` component in Alloy.

You will fix this issue in [Step 3](#step-3).

## Step 3

Alloy manages components in different _stability levels_:

- **General Availability (GA)**
- **Public Preview**
- **Experimental**

The desired stability level for Alloy can be set with the `--stability.level` parameter.

Compare and copy the adjusted [`compose.yml`](./step-3/compose.yml) for step 3 to the OTel demo directory, then restart the `otel-collector` service once again:

```bash
diff opentelemetry-demo/docker-compose.yml step-3/compose.yml
cp step-3/compose.yml opentelemetry-demo/docker-compose.yml
docker compose -f opentelemetry-demo/docker-compose.yml up otel-collector
```

You will see yet another Alloy error in the output:

```
otel-collector  | Error: -: cycle: otelcol.processor.transform.default, otelcol.processor.batch.default, otelcol.processor.memory_limiter.default, otelcol.connector.spanmetrics.default
otel-collector  | Error: could not perform the initial load successfully
```

This is due to a difference in handling/building pipelines between the OTel Collector and Alloy.

Alloy - due to [its DSL](https://grafana.com/docs/alloy/latest/get-started/configuration-syntax/) - requires you to explicitly create **multiple** components if the same type of **processor/connector** should be used twice within the same pipeline.

You will fix this issue in [Step 4](#step-4).

## Step 4

It's time to move away from the `otelcol`-formatted config provided by the OTel demo, and create an equivalent configuration in [Alloy's DSL](https://grafana.com/docs/alloy/latest/get-started/configuration-syntax/).

The Alloy DSL is similar to Hashicorp's HCL (used e.g. in Terraform). A `config.alloy` file equivalent to the `otelcol-config.yml` used since [Step 2](#step-2) can be found in [./step-4/config.alloy`](./step-4/config.alloy).

> [!NOTE]
> Take your time to go through the provided `config.alloy` and spot similarities/differences to the `otelcol-config.yml` used until now.

In order to use the provided `config.alloy`, you will need to update the `otel-collector` service in the `docker-compose.yml` file to map and use this new config.

You can compare the changes and copy the needed files to the OTel demo directory like this:

```bash
diff opentelemetry-demo/docker-compose.yml step-4/compose.yml
cp step-4/compose.yml opentelemetry-demo/docker-compose.yml
cp step-4/config.alloy opentelemetry-demo
```

Afterwards, restart the `otel-collector` service again:

```bash
docker compose -f opentelemetry-demo/docker-compose.yml up otel-collector
```

For the first time, Alloy starts up and initializes the defined pipelines correctly.

You can take a look at Alloy's web interface by visiting [http://localhost:12345](http://localhost:12345).

You will see a list of the initialized components. If you click on **Graph** in the header and wait a few moments, you can validate the created pipelines and that OTel signals are being received, processed, and forwarded.

However, most of the pipelines initially configured for the OTel Demo have been lost by the 'fixes' you applied in [Step 2](#step-2).

You will add _some_ of those pipelines back in [Step 5](#step-5).

## Step 5

This step will be all about adding back some of the signals you lost by adjusting `otelcol-config.yml` in [Step 2](#step-2).

As a reminder, these pipeline components needed to be deleted because there was no equivalent component in Alloy's DSL:

- **Receivers**
  - `docker_stats`
  - `hostmetrics`
  - `httpcheck`
  - `nginx`
  - `postgresql`
  - `redis`
- **Exporters**
  - `opensearch`
- **Processors**
  - `resourcedetection`[^1]

However, for many of these receivers, exporters, and processors, there exists a similar [Alloy component](https://grafana.com/docs/alloy/latest/reference/components/):

| OTel Collector Component | Alloy Component |
|--------------------------|-----------------|
| `docker_stats`           | [`prometheus.exporter.cadvisor`](https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.exporter.cadvisor/) |
| `hostmetrics` | [`prometheus.exporter.unix`](https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.exporter.unix/) / [`prometheus.exporter.windows`](https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.exporter.windows/) |
| `httpcheck` | [`prometheus.exporter.blackbox`](https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.exporter.blackbox/) |
| `nginx` | ❌ |
| `postgresql` | [`prometheus.exporter.postgres`](https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.exporter.postgres/) |
| `redis` | [`prometheus.exporter.redis`](https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.exporter.redis/) |
| `opensearch` | ❌ |
| `resourcedetection` | [`otelcol.processor.resourcedetection`](https://grafana.com/docs/alloy/latest/reference/components/otelcol/otelcol.processor.resourcedetection/) |

Except for two components, there exists a workaround in Alloy DSL for the signals the OTel demo originally collected.

> [!NOTE]
> In many cases the exact metrics differ (in name, labels, or general nature), and thus the **Grafana dashboards** provided by the demo will _not_ work any longer without adjustments.

In [`./step-5/config.alloy`](./step-5/config.alloy) you will find an adjusted Alloy configuration that tries to approximate the original OTel demo setup.

It adds and configures the components mentioned in the table above to collect, process, and export signals for all components of the demo, _except_ NGINX. You can view the changes to `config.alloy` compared to [Step 4](#step-4) like this:

```bash
diff step-4/config.alloy step-5/config.alloy
```

If you want to deploy and experiment with this setup, copy the provided files to the OTel demo directory as you did throughout the previous steps, and browse the collected signals in Grafana.

```bash
cp step-5/config.alloy opentelemetry-demo/config.alloy
cp step-5/compose.yml opentelemetry-demo/docker-compose.yml
docker compose -f opentelemetry-demo/docker-compose.yml up -d
```

> [!NOTE]
> You will have to **add a new datasource** for Loki in Grafana. Loki is available at `http://loki:3100` from Grafana.

You can browse the collected signals in Grafana's [Explore](http://localhost:8080/grafana/explore) or [Drilldown](http://localhost:8080/grafana/drilldown) views.

## Cleanup

You can tear down the whole OTel demo (adjusted or not), by running the following command from the `otel-demo` scenario directory:

```bash
docker compose -f opentelemetry-demo/docker-compose.yml down
# Try deleting the opensearch container in case it's still running
docker stop opensearch && docker rm opensearch
```
