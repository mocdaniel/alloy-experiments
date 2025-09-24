# README

This scenario is meant to showcase [Grafana Alloy's](https://github.com/grafana/alloy) [clustering feature](https://grafana.com/docs/alloy/latest/get-started/clustering/).

To quote the docs, _clustering allows a fleet of Alloy deployments to work together for workload distribution and high availability. It enables horizontally scalable deployments with minimal resource and operation overhead._

This scenario comes preconfigured with 10 services and working OTel pipelines processing signals:

- `alloy-1`: The first member of your Alloy cluster.
- `alloy-2`: The second member of your Alloy cluster.
- `node-exporter-[1-8]`: Eight Prometheus node exporters, each providing a `/metrics` endpoint to scrape Prometheus metrics from

---

## Prerequisites

Bring up the Docker Compose stack containing the services listed above like this:

```console
docker compose up -d
```

You can validate the setup by navigating to the Grafana Alloy Web UIs of the two clustered instances:

- [`http://localhost:12345`](http://localhost:12345) for `alloy-1`
- [`http://localhost:12346`](http://localhost:12346) for `alloy-2`

In the [`compose.yml`](./compose.yml) file, you can inspect the **needed flags** passed to `alloy run` in order to configure clustering:

- `--cluster.enabled` - enables cluster mode
- `--cluster.join-addresses` - the addresses of desired cluster nodes, needed for initial discovery of members
- `--cluster.name` - the cluster name, preventing Alloy deployments from accidentally joining wrong clusters (_optional_)

---

Once the setup has been provisioned, you can inspect the clustering aspect from the Alloy instances' Web UIs: Navigate to the **Clustering view**, and you should see two entries, each listing the **node name**, as well as its **advertized address** and **current state** for the cluster members. The last column of the view indicates which member is the **local node** who's Web UI is currently being inspected.

If you switch to the **Graph view**, you can observe that the reported **throughput metrics** for both cluster members differ from each other. That's because each cluster member independently calculates the subset of discovered targets to scrape.

For this to work correctly, the following **config block** needs to be set in [supported Alloy components](https://grafana.com/docs/alloy/latest/get-started/clustering/#target-auto-distribution):

```hcl
clustering {
    enabled = true
}
```

> [!NOTE]
> If clustering is not enabled for Alloy instances, this block is a no-op.

You can check out this demo's configuration in the `prometheus.scrape` component configured in [`config.alloy`](./config.alloy).

Now that the cluster has been established, try shutting down one of its members, e.g. `alloy-1`:

```console
docker stop alloy-1
```

The **Clustering view** of `alloy-2` will update instantaneously, listing only `alloy-2` as remaining cluster member.

If you switch back to the **Graph view** of `alloy-2`, the througput indicators should have changed to the aggregated amount of metrics reported for both Alloy instances before - there's no other cluster member left to share the load with.

You can reverse the scenario by restarting `alloy-1` and stopping `alloy-2` - the result looks the same.

## Caveat

The Grafana Alloy documentation lists [best practices and caveats](https://grafana.com/docs/alloy/latest/get-started/clustering/#best-practices) for running Alloy in cluster mode. Make sure to check them out and learn more about the implications of cluster mode.

## Cleanup

You can tear down the whole scenario by running the following command from the `remote-config` scenario directory:

```bash
docker compose down
```
