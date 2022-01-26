# Traffic shifting

Version 2 of the customers service has been developed and it's time to deploy it to production.

## Deploying customers, v2

We wish to deploy the new service but aren't yet ready to direct traffic to it.

It would be prudent to separate the task of deploying the new service from the task of directing traffic to it.

### Labels

The customers service is labeled with `app=customers`.

Verify this with:

```shell
kubectl get pod -Lapp,version
```

Note the selector on the customers service:

```shell
kubectl get svc customers -o wide
```

If we were to just deploy v2, the selector would match both versions.

### DestinationRules

We can inform Istio that two distinct subsets of the `customers` service exist, and we can use the `version` label as the discriminator.

```yaml linenums="1" title="customers-destinationrule.yaml"
--8<-- "customers-destinationrule.yaml"
```

1. Apply the above destination rule to the cluster.

1. Verify that it's been applied.

    ```shell
    kubectl get destinationrule
    ```

### VirtualServices

Armed with two distinct destinations, the `VirtualService` custom resource allows us to define a routing rule.

```yaml linenums="1" title="customers-virtualservice.yaml"
--8<-- "customers-virtualservice.yaml"
```

Note above how the route specifies subset v1.

1. Apply the virtual service to the cluster.

1. Verify that it's been applied.

    ```shell
    kubectl get virtualservice 
    ```

### Finally deploy customers, v2

Apply the following Kubernetes deployment to the cluster.

??? tldr "customers-v2.yaml"
    ```yaml linenums="1"
    --8<-- "customers-v2.yaml"
    ```

### Check that traffic routes strictly to v1

1. Let's bring back our friend `siege` to generate some traffic.

    ```shell
    siege --delay=3 --concurrent=3 --time=20M http://$GATEWAY_IP/
    ```

1. Open a separate terminal and launch the Kiali dashboard

    ```shell
    istioctl dashboard kiali
    ```

Take a look at the graph, and select the `default` namespace.

The graph should show all traffic going to v1.

## Route to customers, v2

Being the cautious developers we are, our plan is to "check out v2" before customers do.

## Expose debug traffic to v2

Review this proposed updated routing specification.

```yaml linenums="1" title="customers-v2-debug.yaml"
--8<-- "customers-v2-debug.yaml"
```

We are telling Istio to check an HTTP header:  If the `user-agent` is set to `debug`, route to v2, otherwise route to v1.

Apply the above yaml to the cluster; it will overwrite the currently defined virtualservice as both yamls use the same resource name.

```shell
kubectl apply -f customers-v2-debug.yaml
```

### Test it

Open a browser and visit your $GATEWAY_IP.

```shell
GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
```

We can tell v1 and v2 apart in that v2 displays customers with two columns.

If you're using Chrome or Firefox, you can customize the `user-agent` header as follows:

1. Open the browser's developer tools
2. Open the "three dots" menu, and select "more tools" -> "network conditions"
3. The network conditions panel will open
4. Under "user agent", uncheck "Use browser default"
5. Select _Custom..._ and in the text field enter `debug`

Now refresh the page, and traffic should have been directed to v2.

If you refresh the page a good dozen times and then wait ~15-30 seconds, you should be able to see some of that v2 traffic in Kiali too.

## Canary

Well, v2 looks good; we decide to expose the new version to the public, but we're still prudent.

Start by siphoning 10% of traffic over to v2.

```yaml linenums="1" title="customers-v2-canary.yaml"
--8<-- "customers-v2-canary.yaml"
```

Note above the `weight` field specifying 10 percent of traffic to v2.
Kiali should now showing traffic going to both v1 and v2.

In your browser:  undo the user agent customizations and refresh the page a bunch of times.  Most of the requests still go to v1.

## Check Grafana

Before we open the floodgates, we wish to determine how v2 is fairing.

```shell
istioctl dashboard grafana
```

In grafana, visit the Istio Workload Dashboard and specifically look at the customers v2 workload.
Look at the request rate and the incoming success rate, also the latencies.

If all looks good, up the percentage from 90/10 to, say 50/50.

Watch the request volume change (you may need to click on the "refresh dashboard" button in the upper right-hand corner).

Finally, let's switch all traffic over to v2.

```yaml linenums="1" title="customers-virtualservice-final.yaml"
--8<-- "customers-virtualservice-final.yaml"
```

After you apply the above yaml, go to your browser and make sure all requests land on v2 (2-column output).

Though it no longers receives any traffic, we decide to leave v1 running a while longer before retiring it.
