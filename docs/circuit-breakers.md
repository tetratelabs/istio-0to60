# Circuit breakers

This lab demonstrate how to configure [circuit breaking](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking){target=_blank} both with and without outlier detection in Istio.

## Prerequisites and setup

- Kubernetes with Istio and other tools (Prometheus, Zipkin, Grafana) installed
- `web-frontend` and `customers` workloads already deployed and running.

## Revise the Istio installation configuration

Modify the installation of Istio to use the [demo profile](https://istio.io/latest/docs/setup/additional-setup/config-profiles/){target=_blank} which enables high levels of tracing, which is convenient for this lab.

```shell
istioctl install --set profile=demo
```

## Install Fortio

Let us try and generate some load to the `web-frontend` workload and see the distribution of responses.

We'll use [Fortio](https://fortio.org/){target=_blank} to generate load on the `web-frontend` service.

1. Deploy Fortio

    ```yaml linenums="1" title="fortio.yaml" hl_lines="27-28"
    --8<-- "circuit-breakers/fortio.yaml"
    ```

    Above, notice the annotation, which configures the inclusion of additional Envoy metrics (aka statistics) including circuit breaking.

    Save the above file to `fortio.yaml` and deploy it:

    ```shell
    kubectl apply -f fortio.yaml
    ```

1. Make a single request to make sure everything is working:

    ```shell
    kubectl exec deploy/fortio -c fortio -- fortio curl web-frontend
    ```

    The above command should result in an HTTP 200 "OK" response from the `web-frontend` app.

1. With fortio, we can generate a load of 50 requests with two concurrent connections like this:

    ```shell
    kubectl exec deploy/fortio -c fortio -- \
      fortio load -c 2 -qps 0 -n 50 -quiet web-frontend
    ```

    All 50 requests should succeed.  That is the meaning of `Code 200 : 50` in the output.


!!! info

    Fortio also has a GUI, to access it:

    1. Port-forward the deployment's port

        ```shell
        kubectl port-forward deploy/fortio 8080
        ```
    
    1. In a browser, visit [http://localhost:8080/fortio](http://localhost:8080/fortio){target=_blank}


## Circuit breaker - connection pool settings

Study the following DestionationRule:

```yaml linenums="1" title="cb-web-frontend.yaml"
--8<-- "circuit-breakers/cb-web-frontend.yaml"
```

1. The maximum number of pending HTTP requests to a destination.
1. The maximum number of concurrent requests to a destination.
1. The maximum number of requests per connection.

It configures the connection pool for `web-frontend` with very low thresholds, to easily trigger the circuit breaker.

Save the above YAML to `cb-web-frontend.yaml` and apply the changes:

```shell
kubectl apply -f cb-web-frontend.yaml
```

Since all values are set to 1, we won't trigger the circuit breaker if we send the request using one connection and one request per second.

If we increase the number of connections and send more requests (i.e. 2 workers sending requests concurrently, and sending 50 requests), we'll start getting errors.

The errors happen because the `http2MaxRequests` is set to 1 and we have more than 1 concurrent request being sent. Additionally, we're exceeding the `maxRequestsPerConnection` limit.

```shell
kubectl exec deploy/fortio -c fortio -- \
  fortio load -c 2 -qps 0 -n 50 -quiet web-frontend
```

```console
...
Code 200 : 24 (48.0 %)
Code 503 : 26 (52.0 %)
```

!!! Tip

    To reset the metric counters, run:

    ```shell
    kubectl exec deploy/fortio -c istio-proxy -- curl -sX POST localhost:15000/reset_counters
    ```

## The `x-envoy-overloaded` header

When a request is dropped due to circuit breaking, the response will contain a response header `x-envoy-overloaded` with value "true".

One way to see this header is to run a fortio load with two concurrent connections, uninterrupted, in one terminal:

```shell
kubectl exec deploy/fortio -c fortio -- \
  fortio load -c 2 -qps 0 -t 0 --allow-initial-errors http://web-frontend
```

In a separate terminal, invoke a single request:

```shell
kubectl exec deploy/fortio -c fortio -- fortio curl http://web-frontend
```

Here is an example response to a dropped request:

```console hl_lines="2"
> HTTP/1.1 503 Service Unavailable
> x-envoy-overloaded: true
> content-length: 81
> content-type: text/plain
> date: Thu, 10 Aug 2023 18:25:37 GMT
> server: envoy
>
> upstream connect error or disconnect/reset before headers. reset reason: overflowcommand terminated with exit code 1
```

Then press ++ctrl+c++ to interrupt the load generation.

## Observe failures in Zipkin

Open the Zipkin dashboard:

```shell
istioctl dash zipkin
```

In the Zipkin UI, list failing traces by clicking the "+" button in the search field and specifying the query: `tagQuery=error`.  Then click the **Run Query** button.

Pick a failing trace to view the details.

The requests are failing because the circuit breaker is tripped.  [Response flags](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags){target=_blank} are set to `UO` (Upstream Overflow) and the status code is 503 (service unavailable).

## Prometheus metrics

Another option is looking at the Prometheus metrics directly.

Open the Prometheus dashboard:

```shell
istioctl dash prometheus
```

Apply the following PromQL query:

```promql
envoy_cluster_upstream_rq_pending_overflow{app="fortio", cluster_name="outbound|80||web-frontend.default.svc.cluster.local"}
```

The query shows the metrics for requests originating from the `fortio` app and going to the `web-frontend` service.

The `upstream_rq_pending_overflow` and other metrics are described [in the Envoy documentation](https://www.envoyproxy.io/docs/envoy/latest/configuration/upstream/cluster_manager/cluster_stats#general){target=_blank}.

Noteworthy are [circuit-breaking specific metrics](https://www.envoyproxy.io/docs/envoy/latest/configuration/upstream/cluster_manager/cluster_stats#circuit-breakers-statistics){target=_blank} showing the state of various circuit breakers.  For example `rq_open` indicates whether the "requests" circuit breaker is open, and its companion `remaining_rq` indicates how many requests remain to trip the corresponding circuit breaker.

We can also look at the metrics directly from the `istio-proxy` container in the Fortio Pod:

```shell
kubectl exec deploy/fortio -c istio-proxy -- \
  pilot-agent request GET stats | grep web-frontend | grep pending
```

```console
cluster.outbound|80||web-frontend.default.svc.cluster.local.circuit_breakers.default.remaining_pending: 1
cluster.outbound|80||web-frontend.default.svc.cluster.local.circuit_breakers.default.rq_pending_open: 0
cluster.outbound|80||web-frontend.default.svc.cluster.local.circuit_breakers.high.rq_pending_open: 0
cluster.outbound|80||web-frontend.default.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|80||web-frontend.default.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|80||web-frontend.default.svc.cluster.local.upstream_rq_pending_overflow: 26
cluster.outbound|80||web-frontend.default.svc.cluster.local.upstream_rq_pending_total: 24
```

!!! info

    Yet another convenient way to look at the stats emitted by an Envoy sidecar is via the Envoy dashboard:

    ```shell
    istioctl dashboard envoy deploy/fortio
    ```

    In the web ui, click on the "stats" endpoint, and filter by target outbound cluster "web-frontend".

## Resolving the errors

To resolve these errors, we can adjust the circuit breaker settings.

Increase the maximum number of concurrent requests to 2 (`http2MaxRequests`), as shown below:

```yaml linenums="1" hl_lines="11"
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: web-frontend
spec:
  host: web-frontend.default.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        http2MaxRequests: 2
        maxRequestsPerConnection: 1
```

Save the above YAML to `cb-web-frontend.yaml` and apply the changes:

```shell
kubectl apply -f cb-web-frontend.yaml
```

If we re-run Fortio with the same parameters, we'll notice less failures this time:

```shell
kubectl exec deploy/fortio -c fortio -- \
  fortio load -c 2 -qps 0 -n 50 -quiet web-frontend
```

```console
...
Code 200 : 39 (78.0 %)
Code 503 : 11 (22.0 %)
```

Since we're sending more than 1 request per connection, we can increase the `maxRequestsPerConnection` to 2:

```yaml linenums="1" hl_lines="12"
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: web-frontend
spec:
  host: web-frontend.default.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        http2MaxRequests: 2
        maxRequestsPerConnection: 2
```

Save the above YAML to `cb-web-frontend.yaml` and apply the changes:

```shell
kubectl apply -f cb-web-frontend.yaml
```

If we re-run Fortio this time, we'll get zero or close to zero HTTP 503 responses. Even if we increase the number of requests per second, we should only get a small number of 503 responses. To get rid of the remaining failing requests, we can increase the `http1MaxPendingRequests` to 2:

```yaml linenums="1" hl_lines="10"
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: web-frontend
spec:
  host: web-frontend.default.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 2
        http2MaxRequests: 2
        maxRequestsPerConnection: 2
```

With these settings (assuming 2 concurrent connections), we can easily handle a higher number of requests.

To be clear, the numbers we used in settings are just examples and are not realistic - we set them intentionally low to make the circuit breaker easier to trip.

Before continuing, delete the DestinationRule:

```shell
kubectl delete destinationrule web-frontend
```

Reset the metric counters:

```shell
kubectl exec deploy/fortio -c istio-proxy -- \
  curl -X POST localhost:15000/reset_counters
```

## Outlier detection

The circuit breaker is great when we want to protect the services from a sudden burst of requests. However, how can we protect the services in case of failures?

For example, if we have a service that is still failing after multiple requests, it doesn't make sense to send even more requests to it. Instead, we can remove the instance of the failing service from the load balancing pool for a certain period of time. That way, we know that the requests will go to other instances of the service. After a pre-defined period of time, we can bring the failing service back into the load balancing pool.

This process is called outlier detection. Just like in the connection pool settings, we can configure outlier detection in the DestinationRule.

To see the outlier detection in action we need a service that is failing. We'll create a `web-frontend-failing` deployment and configure it to return HTTP 503 responses:

??? note "Click for web-frontend-failing.yaml"

    ```yaml linenums="1" title="web-frontend-failing.yaml"
    --8<-- "circuit-breakers/web-frontend-failing.yaml"
    ```

Save the above YAML to `web-frontend-failing.yaml` and apply it to the cluster:

```shell
kubectl apply -f web-frontend-failing.yaml
```

If we run Fortio we'll see that majority (roughly, 80%) of the requests will be failing. That's because the `web-frontend-failing` deployment has more replicas than the "good" deployment.

```shell
kubectl exec deploy/fortio -c fortio -- \
  fortio load -c 2 -qps 0 -n 50 -quiet web-frontend
```

```console
...
Code 200 : 9 (18.0 %)
Code 500 : 41 (82.0 %)
```

Let's look at an example of outlier detection configuration:

```yaml linenums="1" title="outlier-web-frontend.yaml"
--8<-- "circuit-breakers/outlier-web-frontend.yaml"
```

1. Number of 5xx errors in a row that will trigger the outlier detection. 
2. The interval at which the hosts are checked whether they need to be ejected.
3. The duration of time an outlier is ejected from the load balancing pool. If the same host is ejected multiple times, the ejection time increases by multiplying the base ejection time by the number of times the host is ejected.
4. The maximum percentage of hosts that can be ejected.

Save the YAML to `outlier-web-frontend.yaml` and apply it:

```shell
kubectl apply -f outlier-web-frontend.yaml
```

If we repeat the test, we might get a similar distribution of responses the first time.  However, if we repeat the command (once the outliers were ejected), we'll get a much better distribution:

```shell
kubectl exec deploy/fortio -c fortio -- \
  fortio load -c 2 -qps 0 -n 50 -quiet web-frontend
```

```console
...
Code 200 : 50 (100.0 %)
```

The reason for more HTTP 200 responses is because as soon as the failing hosts were ejected (failing Pods from the `web-frontend-failing` deployment), the requests were sent to the other host that doesn't fail. If we waited until after the 60 second `baseEjectionTime` expired, the failing hosts would be brought back into the load balancing pool and we'd get a similar distribution of responses as before (majority of them failing).

We can also look at the metrics from the outlier detection in the same way we did for the circuit breakers:

```shell
kubectl exec deploy/fortio -c istio-proxy -- \
  pilot-agent request GET stats | grep web-frontend | grep ejections_total
```

Produces output similar to this:

```console
cluster.outbound|80||web-frontend.default.svc.cluster.local.outlier_detection.ejections_total: 4
```

!!! Note
    Other metrics that we can look at are `ejections_consecutive_5xx`, `ejections_enforced_total` or any other metric with `outlier_detection` in its name. The full list of metric names and their descriptions can be found in the [Envoy documentation](https://www.envoyproxy.io/docs/envoy/latest/configuration/upstream/cluster_manager/cluster_stats#config-cluster-manager-cluster-stats-outlier-detection){target=_blank}.

## Cleanup

To clean up resources created in this lab, run:

```shell
kubectl delete destinationrule web-frontend
kubectl delete -f web-frontend-failing.yaml
kubectl delete -f fortio.yaml
```