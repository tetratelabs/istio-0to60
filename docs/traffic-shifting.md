# Traffic shifting

Version 2 of the customers service has been developed, and it's time to deploy it to production.
Whereas version 1 returned a list of customer names, version 2 also includes each customer's city.

## Deploying customers, v2

We wish to deploy the new service but aren't yet ready to direct traffic to it.

It would be prudent to separate the task of deploying the new service from the task of directing traffic to it.

### Labels

The customers service is labeled with `app=customers`.

Verify this with:

```{.shell .language-shell}
kubectl get pod -Lapp,version
```

Note the selector on the `customers` service in the output to the following command:

```{.shell .language-shell}
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

    ```{.shell .language-shell}
    kubectl get destinationrule
    ```

### VirtualServices

Armed with two distinct destinations, the `VirtualService` custom resource allows us to define a routing rule that sends all traffic to the v1 subset.

```yaml linenums="1" title="customers-virtualservice.yaml"
--8<-- "customers-virtualservice.yaml"
```

Above, note how the route specifies subset v1.

1. Apply the virtual service to the cluster.

1. Verify that it's been applied.

    ```{.shell .language-shell}
    kubectl get virtualservice 
    ```

### Finally deploy customers, v2

Apply the following Kubernetes deployment to the cluster.

??? tldr "customers-v2.yaml"
    ```yaml linenums="1"
    --8<-- "customers-v2.yaml"
    ```

### Check that traffic routes strictly to v1

1. Generate some traffic.

    === "With siege"

        ```{.shell .language-shell}
        siege --delay=3 --concurrent=3 --time=20M http://$GATEWAY_IP/
        ```

    === "With bash"

        ```{.shell .language-shell}
        while true; do curl -I http://$GATEWAY_IP/; sleep 0.5; done
        ```

1. Open a separate terminal and launch the Kiali dashboard

    ```{.shell .language-shell}
    istioctl dashboard kiali
    ```

Take a look at the graph, and select the `default` namespace.

The graph should show all traffic going to v1.

## Route to customers, v2

We wish to proceed with caution.  Before customers can see version 2, we want to make sure that the service functions properly.

## Expose "debug" traffic to v2

Review this proposed updated routing specification.

```yaml linenums="1" title="customers-vs-debug.yaml"
--8<-- "customers-vs-debug.yaml"
```

We are telling Istio to check an HTTP header:  if the `user-agent` is set to `debug`, route to v2, otherwise route to v1.

Open a new terminal and apply the above resource to the cluster; it will overwrite the currently defined VirtualService as both yaml files use the same resource name.

```{.shell .language-shell}
kubectl apply -f customers-vs-debug.yaml
```

### Test it

Open a browser and visit the application.

??? tip "If you need it"

    ```{.shell .language-shell}
    GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
    ```

We can tell v1 and v2 apart in that v2 displays not only customer names but also their city (in two columns).

The `user-agent` header can be included in a request in a number of ways:

=== "Developer Tools"

    If you're using Chrome or Firefox, you can customize the `user-agent` header as follows:

    1. Open the browser's developer tools
    2. Open the "three dots" menu, and select _More tools --> Network conditions_
    3. The network conditions panel will open
    4. Under _User agent_, uncheck _Use browser default_
    5. Select _Custom..._ and in the text field enter `debug`

    Refresh the page; traffic should be directed to v2.

=== "`curl`"

    ```{.shell .language-shell}
    curl -H "user-agent: debug" http://$GATEWAY_IP
    ```

=== "Using a custom browser extension"

    Check out [modheader](https://modheader.com/){target=_blank}, a convenient browser extension for modifying HTTP headers in-browser.


!!! tip

    If you refresh the page a good dozen times and then wait ~15-30 seconds, you should see some of that v2 traffic appear in Kiali.


## Canary

Well, v2 looks good; we decide to expose the new version to the public, but we're still prudent.

Start by siphoning 10% of traffic over to v2.

```yaml linenums="1" title="customers-vs-canary.yaml"
--8<-- "customers-vs-canary.yaml"
```

Above, note the `weight` field specifying 10 percent of traffic to subset `v2`.
Kiali should now show traffic going to both v1 and v2.

- Apply the above resource.
- In your browser:  undo the injection of the `user-agent` header, and refresh the page a bunch of times.

In Kiali, under the _Display_ pulldown menu, you can turn on traffic distribution, to see how much traffic is sent to each subset.

Most of the requests still go to v1, but some (10%) are directed to v2.


## Check Grafana

Before we open the floodgates, we wish to determine how v2 is faring.

```{.shell .language-shell}
istioctl dashboard grafana
```

In Grafana, visit the Istio Workload Dashboard and specifically look at the customers v2 workload.
Look at the request rate and the incoming success rate, also the latencies.

If all looks good, up the percentage from 90/10 to, say 50/50.

Watch the request volume change (you may need to click on the "refresh dashboard" button in the upper right-hand corner).

Finally, switch all traffic over to v2.

```yaml linenums="1" title="customers-virtualservice-final.yaml"
--8<-- "customers-virtualservice-final.yaml"
```

After you apply the above yaml, go to your browser and make sure all requests land on v2 (2-column output).
Within a minute or so, the Kiali dashboard should also reflect the fact that all traffic is going to the customers v2 service.

Though it no longer receives any traffic, we decide to leave v1 running a while longer before retiring it.

If you wish to go further, investigate [Flagger](https://flagger.app/){target=_blank}, an Istio-compatible tool that can be used to automate the process of progressive delivery (aka Canary rollouts).  [Here](https://github.com/eitansuez/istio-flagger) is an exploration of Flagger with Istio and its `bookinfo` sample application.