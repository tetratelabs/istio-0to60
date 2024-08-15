# Service discovery and load balancing

This lab is a standalone exploration of service discovery and load balancing in Istio.

## Clusters and endpoints

The `istioctl` CLI's diagnostic command `proxy-status` provides a simple way to list all proxies that Istio knows about.

Run and study the output of the `proxy-status` command:

```shell
istioctl proxy-status
```

Since We have not yet deployed any workloads, the output should be rather anemic, citing the lone ingress gateway that was deployed when we installed Istio in the previous lab.

## Enable automatic sidecar injection

There are two options for [sidecar injection](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/){target=_blank}: automatic and manual.

In this lab we will use automatic injection, which involves labeling the namespace where the pods are to reside.

1.  Label the default namespace

    ```{.shell .language-shell}
    kubectl label namespace default istio-injection=enabled
    ```

1. Verify that the label has been applied:

    ```{.shell .language-shell}
    kubectl get ns -Listio-injection
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

Check the output of `proxy-status` again:

```shell
istioctl proxy-status
```

Confirm that the two `helloworld` workloads are listed and marked as "SYNCED".

While here, let us also deploy the sample app called `sleep`, that will serve the purpose of a client from which we might call the `helloworld` app:

```shell
kubectl apply -f samples/sleep/sleep.yaml
```


## The service registry

Istio maintains an internal service registry which can be observed through a debug endpoint `/debug/registryz` exposed by `istiod`:

`curl` the registry endpoint:

```shell
kubectl exec -n istio-system deploy/istiod -- \
  curl -s localhost:15014/debug/registryz
```

The output can be prettified, and filtered (to highlight the list of host names in the registry) with a tool such as [`jq`](https://stedolan.github.io/jq/){target=_blank}.

```shell
kubectl exec -n istio-system deploy/istiod -- \
  curl -s localhost:15014/debug/registryz | jq .[].hostname
```

Confirm that the `helloworld` service is listed in the output.

## The sidecar configuration

Review the deployments in the `default` namespace:

```shell
kubectl get deploy
```

The `istioctl` CLI's diagnostic command `proxy-config` will help us inspect the configuration of proxies.

Envoy's term for a service is "cluster".

Confirm that `sleep` knows about other services (`helloworld`, mainly):

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

We learn that Istio has communicated to the `sleep` workload information about both `helloworld` endpoints.

## Load balancing

The `sleep` pod's container image has `curl` pre-installed.

Make repeated calls to the `helloworld` service from the `sleep` pod:

```shell
for i in {1..3}; do
  kubectl exec deploy/sleep -- curl -s helloworld:5000/hello
done
```

Some responses will be from `helloworld-v1` while others from `helloworld-v2`, an indication that Envoy is load-balancing requests between these two endpoints.

Envoy does not use the ClusterIP service.  It performs client-side load-balancing using the endpoints you resolved above.

We can examine the `helloworld` "cluster" definition in a sample client to see what load balancing policy is in effect:

```shell
istioctl proxy-config cluster deploy/sleep \
  --fqdn helloworld.default.svc.cluster.local -o yaml | grep lbPolicy
```

To influence the load balancing algorithm that Envoy uses when calling `helloworld`, we can define a [traffic policy](https://istio.io/latest/docs/reference/config/networking/destination-rule/#LoadBalancerSettings-SimpleLB){target=_blank}, like so:

```yaml linenums="1" title="helloworld-lb.yaml"
--8<-- "discovery/helloworld-lb.yaml"
```

Apply the above traffic policy to the cluster:

```shell
kubectl apply -f helloworld-lb.yaml
```

Examine the updated load-balancer policy:

```shell
istioctl proxy-config cluster deploy/sleep \
  --fqdn helloworld.default.svc.cluster.local -o yaml | grep lbPolicy
```

Confirm that it now reads "RANDOM".

For more insight into the merits of the different load balancing options, read the blog entry [Examining Load Balancing Algorithms with Envoy](https://blog.envoyproxy.io/examining-load-balancing-algorithms-with-envoy-1be643ea121c){target=_blank} from the Envoy Proxy blog.


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

With the subsets defined, we turn our attention to the routing specification.  We use a [VirtualService](https://istio.io/latest/docs/reference/config/networking/virtual-service/){target=_blank}, in this case to direct 25% of traffic to v1 and 75% to v2:

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

I think you'll agree the output of the `istioctl x describe` command is a little easier to parse in comparison.
