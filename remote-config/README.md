# README

This scenario is meant to showcase [Grafana Alloy's](https://github.com/grafana/alloy) [remote configuration feature](https://grafana.com/docs/alloy/latest/reference/components/remote/).

Alloy can import and digest configuration from the following sources:

- [Kubernetes](https://kubernetes.io) secrets
- [Kubernetes](https://kubernetes.io) configmaps
- HTTP endpoints
- [Hashicorp Vault](https://developer.hashicorp.com/vault) (the docs don't state whether [OpenBao](https://openbao.org/) is supported as well)
- S3 object storage, e.g. on [NWS](https://nws.netways.de/en/cloud-services/) or [AWS](https://aws.amazon.com/s3/)


If you want to follow along, there are multiple steps provided that guide you through **converting** a local `config.alloy` file into an [Alloy custom component](https://grafana.com/docs/alloy/latest/get-started/custom_components/) that will be uploaded to and consumed from S3 object storage.

---

## Prerequisites

You will start off with the same setup as in the [Alloy UI scenario](../alloy-ui/README.md). In addition, you will need an **S3 bucket**, along with the bucket's **client and secret key** for access. You can create S3 buckets e.g. on [NWS](https://id.nws.netways.de/realms/nws/protocol/openid-connect/registrations?client_id=atlas&redirect_uri=https://my.nws.netways.de/home/index?locale=en&response_type=code&scope=email&kc_locale=en&_gl=1*1946p0s*_gcl_au*MTE4NTEyMDkxNi4xNzU0MjkxMTgw) or AWS.

Afterwards, bring up the demo stack by running the provided [`compose.yml`](./compose.yml) with Docker:

```bash
docker compose up -d
```

This starts four services:

- `alloy`: Grafana Alloy
- `otel-metrics-generator`: the [otelgen](https://github.com/krzko/otelgen) utility generating OTel metric signals
- `otel-logs-generator`: the [otelgen](https://github.com/krzko/otelgen) utility generating OTel log signals
- `otel-traces-generator`: the [otelgen](https://github.com/krzko/otelgen) utility generating OTel trace signals

Once the compose stack is running, you can visit [`http://localhost:12345`](http://localhost:12345) to access Alloy's Web UI and make sure everything is working.

---

## Step 1

Once the setup is running and you confirmed it's working as intended, you will have to convert the contents of [`config.alloy`](./config.alloy) into a [custom component](https://grafana.com/docs/alloy/latest/get-started/custom_components/) to be pushed to your S3 bucket.

This can be done by wrapping the existing config in a `declare` block and saving it to a new file:

```hcl
declare "my-remote-config" {
    <existing Alloy config>
}
```

> [!NOTE]
> The `livedebugging` component needs to be removed from the custom component, as so-called _service blocks_ are not allowed within custom components.

A resulting custom component definition can be found in [`./step-1/remote-config.alloy`](./step-1/remote-config.alloy).

This file needs to be **pushed** to your **S3 bucket**:

```console
s3cmd put step-1/remote-config.alloy s3://<bucket-name>/remote-config.alloy
```

Substitute `bucket-name` in the command above with your bucket name.

You will adjust the existing `config.alloy` to pull and instantiate the remote config in [Step 2](#step-2).

## Step 2

Once you converted the existing config into a custom component and uploaded it to your S3 bucket, you will have to tell Alloy to fetch and read it.

This can be done using Alloy's [`remote.s3`](https://grafana.com/docs/alloy/latest/reference/components/remote/remote.s3/) component definition that pulls contents from files in your buckets and exports them from the component as strings.

You will have to take care of the following steps:

1. Fetch the remote config using `remote.s3`.
2. Import the fetched remote config (as string) using [`import.string`](https://grafana.com/docs/alloy/latest/reference/config-blocks/import.string/).
3. Instantiate the imported custom component.

The resulting `config.alloy` looks like this:

A resulting, new `config.alloy` is provided in [`./step-2/adjusted-config.alloy`](./step-2/adjusted-config.alloy).

Copy the adjusted version over to `config.alloy` and fill in the needed information:

```console
cp ./step-2/adjusted-config.alloy config.alloy
```

Make sure to substitute the following placeholders:

- `bucket-name` - the name of your bucket
- `client-key` - your client key
- `secret-key` - your secret key
- `s3-endpoint` - the endpoint of your S3 storage provider
- `region` - the region your bucket is located in

## Step 3

Once all the needed adjustments are done, redeploy the Docker Compose stack:

```console
docker compose up -d --force-recreate
```

Check the Alloy Web UI at [`http://localhost:12345`](http://localhost:12345).

If you navigate to the **Graph View**, you will see that it now looks different and only displays the two components defined in your new iteration of `config.alloy`.

This is intended, as `imported_remote_config.my_custom_config.instantiated_remote_config` is its own component. However, if you switch to this component's **Details View** by clicking on it, a button in the top of the view labeled **Graph** lets you switch back to the view of your OTel pipelines you might have expected, as they are now contained by your new, custom component.

## Cleanup

You can tear down the whole scenario by running the following command from the `remote-config` scenario directory:

```bash
docker compose down
```
