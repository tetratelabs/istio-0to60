# Observability

In this lab we explore one of the main strengths of Istio: observability.

The services in our mesh are automatically observable, without adding any burden on developers.

## Deploy the Addons

The istio distribution provides addons for a number of systems that together provide observability for the service mesh:

- [Zipkin](https://zipkin.io/) or [Jaeger](https://www.jaegertracing.io/) for distributed tracing
- [Prometheus](https://prometheus.io/) for metrics collection
- [Grafana](https://grafana.com/) provides dashboards for monitoring, using Prometheus as the data source
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

!!! note

    The `siege` command stays in the foreground while it runs.
    It may be simplest to leave it running, and open a separate terminal in your cloud shell environment.

## Kiali

Launch the Kiali dashboard:

```shell
istioctl dashboard kiali
```

!!! note

    The `istioctl dashboard` command also blocks.
    Leave it running until you're finished using the dashboard, at which time 
    pressing ++ctrl+c++ can interrupt the process and put you back at the terminal prompt.

The Kiali dashboard displays.

Customize the view as follows:

1. Select the `Graph` section from the sidebar.
1. From the `Namespace` "pulldown" menu at the top of the screen, select the `default` namespace, the location where the application's pods are running.
1. From the third _pulldown_ menu, select _App graph_.
1. From the "Display" _pulldown_, toggle on _Traffic Animation_ and _Security_.
1. From the footer, toggle the legend so that it is visible.  Take a moment to familiarize yourself with the legend.

Observe the visualization and note the following:

- We can see traffic coming in through the ingress gateway to the `web-frontend`, and the subsequent calls from the `web-frontend` to the `customers` service
- The lines connecting the services are green, indicating healthy requests
- The small lock icon on each edge in the graph indicates that the traffic is secured with mutual TLS

Such visualizations are helpful in understanding the flow of requests in the mesh and in diagnosis.

Feel free to spend more time exploring Kiali.

We will revisit Kiali in a later lab to visualize traffic shifting such as when performing a blue-green or canary deployment.

### Kiali Cleanup

Close the Kiali dashboard.  Interrupt the `istioctl dashboard kiali` command by pressing ++ctrl+c++.


## Zipkin

Launch the Zipkin dashboard:

```shell
istioctl dashboard zipkin
```

The zipkin dashboard displays.

- Click on the red '+' button and select "serviceName".
- Select the service named `web-frontend.default` and click on the _Run Query_ button (lightblue) to the right.

A number of query results will display.  Each row is expandable and will display more detail in terms of the services participating in that particular trace.

Click the _Show_ button to the right of one of the traces.  The resulting view shows spans that are part of the trace, and more importantly how much time was spent within each span.  Such information can help diagnose slow requests and pin-point where the latency lies.

Such traces also help us make sense of the flow of requests in a microservice architecture.

### Zipkin Cleanup

Close the Zipking dashboard.  Interrupt the `istioctl dashboard zipkin` command with ++ctrl+c++.


## Prometheus

[tbd]

## Grafana

[tbd]

