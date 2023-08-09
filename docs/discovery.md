# Service discovery and load balancing

This lab explores service discovery and load balancing in Istio.

## Clusters and endpoints

The `istioctl` CLI's diagnostic command `proxy-status` provides a simple way to list all proxies that Istio knows about.

Confirm that `istiod` knows about the workloads running on Kubernetes:

```shell
istioctl proxy-status
```

## Deploy the `helloworld` sample

The Istio distribution comes with a sample application "helloworld".

```shell
cd ~/istio-{{istio.version}}
```

Deploy `helloworld` to the default namespace.

```shell
kubectl apply -f samples/helloworld/helloworld.yaml
```

Check the output of `proxy-status` again, and confirm that `helloworld` is listed:

```shell
istioctl proxy-status
```

## The service registry

Istio maintains an internal service registry which can be observed through a debug endpoint `/debug/registryz` exposed by `istiod`:

1. `curl` the registry endpoint:

    ```shell
    kubectl exec -n istio-system deploy/istiod -- \
      curl localhost:15014/debug/registryz
    ```

    The output can be prettified with a tool such as [`jq`](https://stedolan.github.io/jq/){target=_blank}.

    ```shell
    kubectl exec -n istio-system deploy/istiod -- \
      curl localhost:15014/debug/registryz | jq .[].hostname
    ```

## The sidecar configuration

Review the deployments in the `default` namespace:

```shell
kubectl get deploy
```

The `istioctl` CLI's diagnostic command `proxy-config` will help us inspect the configuration of proxies.

Envoy's term for a service is "cluster".

Confirm that `sleep` knows about other services (`helloworld`, `customers`, etc..):

```shell
istioctl proxy-config clusters deploy/sleep
```

List the endpoints backing each "cluster":

```shell
istioctl proxy-config endpoints deploy/sleep
```

Zero in on the endpoints for the `helloworld` service:

```shell
istioctl proxy-config endpoints deploy/sleep \
  --cluster "outbound|5000||helloworld.default.svc.cluster.local"
```

## Load balancing

The `sleep` pod's container image has `curl` pre-installed.

Make repeated calls to the `helloworld` service from the `sleep` pod:

```shell
kubectl exec deploy/sleep -- curl -s helloworld:5000/hello
```

Some responses will be from `helloworld-v1` while others from `helloworld-v2`, an indication that Envoy is load-balancing requests between these two endpoints.

Envoy does not use the ClusterIP service.  It performs client-side load-balancing using the endpoints you resolved above.

To influence the load balancing algorithm Envoy uses when calling `helloworld`, we can define a traffic policy.

```yaml linenums="1" title="helloworld-lb.yaml"
--8<-- "discovery/helloworld-lb.yaml"
```

Apply the above traffic policy to the cluster:

```shell
kubectl apply -f helloworld-lb.yaml
```

!!! Tip

    We highly recommend the follow blog entry on the Envoy Proxy blog about [Examining Load Balancing Algorithms with Envoy](https://blog.envoyproxy.io/examining-load-balancing-algorithms-with-envoy-1be643ea121c){target=_blank}


We can examine the `helloworld` "cluster" definition in a sample client and note that the specified load balancing policy has been applied:

```shell
istioctl proxy-config cluster deploy/sleep \
  --fqdn helloworld.default.svc.cluster.local -o yaml | grep lbPolicy
```

## Traffic distribution

We can go a step further and control how much traffic to send to version v1 and how much to v2.

First, define the two subsets, v1 and v2:

```yaml linenums="1" title="helloworld-dr.yaml"
--8<-- "discovery/helloworld-dr.yaml"
```

Apply the updated destination rule to the cluster:

```shell
kubectl apply -f helloworld-dr.yaml
```

If we now inspect the list of clusters, note that there's one for each subset:

```shell
istioctl proxy-config cluster deploy/sleep \
  --fqdn helloworld.default.svc.cluster.local
```

With the subsets defined, we turn our attention to the routing specification.  We use a [VirtualService](https://istio.io/latest/docs/reference/config/networking/virtual-service/), in this case to direct 25% of traffic to v1 and 75% to v2:

```yaml linenums="1" title="helloworld-vs.yaml"
--8<-- "discovery/helloworld-vs.yaml"
```

Apply the VirtualService to the cluster:

```shell
kubectl apply -f helloworld-vs.yaml
```

Finally, we can inspect the routing rules applied to an Envoy client with our `proxy-config` diagnostic command:

```shell
istioctl proxy-config routes deploy/sleep --name 5000 -o yaml
```

Note the `weightedClusters` section in the routes output.

The `istioctl` CLI provides a convenient command to inspect the configuration of a service:

```shell
istioctl x describe svc helloworld
```

I think you'll agree the output of the `istioctl x describe` command is a little easier to parse in comparison to previous Envoy configurations.