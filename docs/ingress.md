# Ingress

The main objective of this lab is to expose the web frontend to the public internet.

## The ingress gateway

When you installed Istio, in addition to deploying istiod to Kubernetes, the installation also provisioned an Ingress Gateway.

View the corresponding Istio ingress gateway pod in the `istio-system` namespace.

```shell
kubectl get pod -n istio-system
```

A corresponding LoadBalancer type service was also created:

```shell
kubectl get svc -n istio-system
```

Make a note of the external IP address for the load balancer.

Assign it to an environment variable.

```shell
GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
```

??? tip "A small investment"

    Inevitably at some point our session will end, or we'll open a new terminal and the above variable will be out of scope.

    We will be referencing `$GATEWAY_IP` in susequent labs.

    Ensure `GATEWAY_IP` is set each time we start a new shell:

    ```shell
    echo "export GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')" >> ~/.bashrc
    ```


We could create a DNS A record for this IP address, but for the sake of simplicity, we will access anything we expose from the mesh by using the IP address directly.

## Configuring ingress

Configuring ingress with Istio is split into two parts:

- Define a `Gateway` custom resource that governs the specific host, port, and protocol to expose
- Specify how requests should be routed with a `VirtualService` custom resource.

### Create a Gateway resource

1. Review the following Gateway specification.

    ??? tldr "gateway.yaml"
        ```yaml linenums="1"
        --8<-- "gateway.yaml"
        ```

    We specify the HTTP protocol, port 80, and the wildcard host value ensures a match against HTTP requests that reference the load balancer IP address `$GATEWAY_IP`.

    The selector _istio: ingressgateway_ ensures that this gateway resource binds to the physical ingress gateway.

1. Apply the gateway resource to your cluster.

    ```shell
    kubectl apply -f gateway.yaml
    ```

1. Attempt an HTTP request in your browser to the gateway IP address.  It should return a 404 (not found).

### Create a VirtualService resource

1. Review the following VirtualService specification.

    ??? tldr "web-frontend-virtualservice.yaml"
        ```yaml linenums="1"
        --8<-- "web-frontend-virtualservice.yaml"
        ```

    Note how this specification references the name of the gateway ("frontend-gateway"), a matching host ("*"), and specifies a route for requests to be directed to the `web-frontend` service.

1. Apply the virtual service resource to your cluster.

    ```shell
    kubectl apply -f web-frontend-virtualservice.yaml
    ```

1. List virtual services in the default namespace.

    ```shell
    kubectl get virtualservice
    ```

    The output indicates that the virtual service named `web-frontend` is bound to the gateway, as well as any hostname that routes to the load balancer IP address.

Finally, verify that you can now access `web-frontend` from your web browser using the gateway IP address.

## Candidate follow-on exercises

- Consider creating a DNS A record for the gateway IP, and narrowing down the scope of the gateway to only match that hostname.
- [Configuring a TLS ingress gateway](https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/#configure-a-tls-ingress-gateway-for-a-single-host)
 

## Next

The application is now running and exposed on the internet.

In the next lab, we turn our attention to the observability features that are built in to Istio.
