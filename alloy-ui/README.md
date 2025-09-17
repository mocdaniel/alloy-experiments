# README

This scenario is meant to showcase one of [Grafana Alloy's](https://github.com/grafana/alloy) unique features: its **Web UI**.

With Alloy's Web UI, you can **inspect and debug** your OTel pipelines at runtime, including information such as **healthiness, signal throughput**, and additional information.

The goal is to set up a minimal OTel setup generating OTel signals and have them processed by Alloy in order to take a look at its UI features.

If you want to follow along, a suitable [`compose.yml](./compose.yml) and [`config.alloy`](./config.alloy) are provided in this directory.

---

## Provisioning the Environment

Bring up the demo stack by running the provided [`compose.yml`](./compose.yml) with Docker:

```bash
docker compose up -d
```

This starts four services:

- `alloy`: Grafana Alloy
- `otel-metrics-generator`: the [otelgen](https://github.com/krzko/otelgen) utility generating OTel metric signals
- `otel-logs-generator`: the [otelgen](https://github.com/krzko/otelgen) utility generating OTel log signals
- `otel-traces-generator`: the [otelgen](https://github.com/krzko/otelgen) utility generating OTel trace signals

Once the compose stack is running, you can visit [`http://localhost:12345`](http://localhost:12345) to access Alloy's Web UI.

## Navigating the Alloy Web UI

### Component View

Opening the link to Alloy's Web UI above will display Alloy's **component overview**, which allows you to spot **healthy/unhealthy** components at a glance. If you want to view additional details, click on the **View** button of the respective component.

### Graph View

Clicking on **Graph** in the navbar will open a graph view of your configured OTel pipelines in Alloy.

After a few seconds, the graph's **edges** will be populated and color-coded with **live information** of your OTel platforms:

- **Colors**: Indication of the **OTel signal type** being processed.
- **Numbers**: Indication of **throughput**, stated as `signals/s`.

> [!NOTE]
> Only the edges between components that support **live debugging** in Alloy are enriched with live information of your pipelines.

You can adjust the inspection window with the slider above the graph; the minimum timeframe is `1s`, and the maximum is `60s`. The **live information** will update accordingly after a few seconds.

## Details View

If you click on one of the components in the [Component View](#component-view) or nodes in the [Graph View](#graph-view), you end up in the component's Detail View.

Here you can see **health-related messages**, **arguments** passed to the component, as well as configured *exports** and the component's **dependants/dependencies** (if any).

You can also find the entrypoint to the component's [Live-Debugging View](#live-debugging-view) here, if available.

> [!NOTE]
> Not all components support **live-debugging** (yet). In addition to the component supporting live-debugging, the feature itself must be enabled in [`config.alloy`](./config.alloy) first:
> ```hcl
> livedebugging {
>   enabled = true
> }
> ```    

## Live-Debugging View

In a component's live-debugging view, you can inspect incoming OTel signals in real-time.

The controls in the top of the view allow you to **filter, clean-up, copy**, or **stop** the collection of live-debugging information. Additionally, you can **tweak the sample-rate** of the captured OTel signals.

## Cleanup

You can tear down the whole scenario, by running the following command from the `alloy-ui` scenario directory:

```bash
docker compose down
```
