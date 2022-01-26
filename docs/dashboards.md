# Observability

In this lab we explore one of the main strengths of Istio: observability.

The services in our mesh are automatically observable, without adding any burden on developers.

## Deploy the Addons

The istio distribution provides addons for a number of systems that together provide observability for the service mesh:

- Zipkin or Jaeger distributed tracing
- Prometheus for metrics collection
- Grafana provides dashboards for monitoring
- Kiali for visualizing the mesh

These addons are located in the `samples/addons/` folder of the distribution.

1. Navigate to the addons directory

    ```shell
    cd ~/istio-1.12.2/samples/addons
    ```

1. Deploy each addon:

    ```shell
    kubectl apply -f extras/zipkin.yaml
    ```

    ```shell
    kubectl apply -f prometheus.yaml
    ```

    ```shell
    kubectl apply -f grafana.yaml
    ```

    ```shell
    kubectl apply -f kiali.yaml
    ```

The `istioctl` CLI provides convenience commands for accessing the web UIs for each dashboard.

Take a moment to review the help output for the `istioctl dashboard` command:

```shell
istioctl dashboard --help
```

## Generate a load

In order to have something to observe, we need to generate a load on our system.

- start with kiali, demo it, consider perhaps showing how mtls is turned on inside the mesh.  mention that will come back to kiali when we begin doing traffic shifting.
- show zipkin and distributed traces
- show prometheus and grafana
