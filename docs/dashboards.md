# Observability

In this lab we explore one of the main strengths of Istio: observability.

The services in our mesh are automatically observable, without adding any burden on developers.

## Deploy the Addons

The istio distribution provides addons for a number of systems that together provide observability for the service mesh:

- [Zipkin](https://zipkin.io/) or [Jaeger](https://www.jaegertracing.io/) for distributed tracing
- [Prometheus](https://prometheus.io/) for metrics collection
- [Grafana](https://grafana.com/) provides dashboards for monitoring
- [Kiali](https://kiali.io/) allows us to visualize the mesh

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

### Install a load generator

1. Install a simple load generating tool named `siege`.

    [tbd]

1. Familiarize yourself with the command and its options.

    ```shell
    siege --help
    ```

Run the following command to generate a mild load against the application.

```shell
siege --delay=3 --concurrent=3 --time=20M http://$GATEWAY_IP/
```

## Kiali

[tbd]

- start with kiali, demo it, consider perhaps showing how mtls is turned on inside the mesh.  mention that will come back to kiali when we begin doing traffic shifting.

## Zipkin

[tbd]

## Prometheus

[tbd]

## Grafana

[tbd]

