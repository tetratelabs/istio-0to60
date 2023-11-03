# Ingress with the Kubernetes Gateway API

Like the previous lab, the objective of this lab is to expose the `web-frontend` service to the internet.

Rather than use the Istio native `Gateway` and `VirtualService` resources, in this lab the implementation leverages the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/){target=_blank}.

!!! warning "When using K3D"

    This lab is not yet compatible with a local Kubernetes cluster setup using K3D.

## Prerequisites

Install the Kubernetes Gateway API Custom Resource Definitions (CRDs):

```shell
kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.0.0" | kubectl apply -f -
```

## The Ingress gateway

When you installed Istio, in addition to deploying `istiod` to Kubernetes, the installation also provisioned an Ingress Gateway.

Unlike Istio, with the Kubernetes Gateway API, a Gateway is deployed implicitly when a Gateway resource is applied to the Kubernetes cluster.

That is, this solution will not utilize the ingress gateway already deployed and running in the `istio-system` namespace.

## Configuring the Gateway

The K8S Gateway API preserves Istio's design of separating the configuration of the Gateway from the routing concerns by using two distinct resources.  The Gateway API of course defines its own CRDs:

1. `Gateway` (although the resource name is the same, the `apiVersion` field is `gateway.networking.k8s.io/v1beta1`)
4. `HttpRoute`

### Create a Gateway resource

1. Review the following Gateway specification.

    !!! tldr "k8s-gw.yaml"
        ```yaml linenums="1"
        --8<-- "ingress/k8s-gw.yaml"
        ```

    Above, we specify the HTTP protocol and port 80.  We leave the optional `hostname` field blank, to allow any ingress request to match.  This is similar to using a wildcard ("*") host matcher in Istio.

1. Apply the gateway resource to your cluster.

    ```{.shell .language-shell}
    kubectl apply -f k8s-gw.yaml
    ```

Before proceeding, wait until the associated Gateway deployment is provisioned:

```shell
kubectl wait --for=condition=programmed gtw frontend-gateway
```

Note above how the short name for this Gateway resource is `gtw` (Istio's Gateway resource's short name is `gw`).

### Verify the Gateway deployment

Note that a gateway deployment by the name of `frontend-gateway-istio` has been created:

```{.shell .language-shell}
kubectl get deploy
```

A corresponding _LoadBalancer_ type service was also created:

```{.shell .language-shell}
kubectl get svc
```

Make a note of the external IP address for the load balancer.

Assign it to an environment variable.

```{.shell .language-shell}
export GATEWAY_IP=$(kubectl get svc frontend-gateway-istio -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
```

??? tip ":material-console:{.gcp-blue} A small investment"

    When the cloud shell connection is severed, or when opening a new terminal tab, `$GATEWAY_IP` will no longer be in scope.

    Ensure `GATEWAY_IP` is set each time we start a new shell:

    ```{.shell .language-shell}
    cat << EOF >> ~/.bashrc

    export GATEWAY_IP=$(kubectl get svc frontend-gateway-istio -ojsonpath='{.status.loadBalancer.ingress[0].ip}')

    EOF
    ```

In normal circumstances we associate this IP address with a hostname via DNS.
For the sake of simplicity, in this workshop we use the gateway public IP address directly.

## Configure routing

Attempt an HTTP request in your browser to the gateway IP address.  It should return a 404 (not found).

Let us "fix" that issue by defining a route to the `web-frontend` service.

1. Review the following `HttpRoute` specification.

    ???+ tldr "web-frontend-route.yaml"
        ```yaml linenums="1"
        --8<-- "ingress/web-frontend-route.yaml"
        ```

    Note how this specification references the name of the gateway ("frontend-gateway").  The absence of a matching hostname will direct all requests to the `web-frontend` service, irrespective of host name.

1. Apply the `HttpRoute` resource to your cluster.

    ```{.shell .language-shell}
    kubectl apply -f web-frontend-route.yaml
    ```

1. List HTTP routes in the default namespace.

    ```{.shell .language-shell}
    kubectl get httproute
    ```

    The output indicates that the HTTP route named `web-frontend` is bound to the gateway, as well as any hostname that routes to the load balancer IP address.

Finally, verify that you can now access `web-frontend` from your web browser using the gateway IP address.



!!! question "What if I wanted to configure ingress with TLS?"

    Here is a recipe that illustrates how to configure secure ingress with a self-signed certificate:

    1. Generate the certificate:

        1. Generate a self-signed root certificate in the folder `example_certs`

            ```{.shell .language-shell}
            mkdir example_certs
            openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example_certs/example.com.key -out example_certs/example.com.crt
            ```

        1. Generate a certificate and private key for the hostname `webfrontend.example.com`:

            ```{.shell .language-shell}
            openssl req -out example_certs/webfrontend.example.com.csr -newkey rsa:2048 -nodes -keyout example_certs/webfrontend.example.com.key -subj "/CN=webfrontend.example.com/O=webfrontend organization"
            openssl x509 -req -sha256 -days 365 -CA example_certs/example.com.crt -CAkey example_certs/example.com.key -set_serial 0 -in example_certs/webfrontend.example.com.csr -out example_certs/webfrontend.example.com.crt
            ```

    1. Store the certificate as a secret in your Kubernetes cluster:

        ```{.shell .language-shell}
        kubectl create -n default secret tls webfrontend-credential \
          --key=example_certs/webfrontend.example.com.key \
          --cert=example_certs/webfrontend.example.com.crt
        ```

    1. Revise the gateway configuration to listen on port 443, and to reference the secret that the envoy listeners will present to incoming requests:

        ```yaml linenums="1" hl_lines="9-15"
        --8<-- "ingress/k8s-gw-tls.yaml"
        ```

    1. Apply the revised gateway configuration:

        ```{.shell .language-shell}
        kubectl apply -f k8s-gw-tls.yaml
        ```

    1. Test your implementation by making a request to the ingress gateway:

        ```{.shell .language-shell}
        curl -s -v --head -k https://webfrontend.example.com/ --resolve webfrontend.example.com:443:$GATEWAY_IP
        ```

    See the [Istio documentation](https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/){target=_blank} for additional examples relating to the topic of configuring secure gateways.

## Next

The application is now running and exposed on the internet.

In the next lab, we turn our attention to the observability features that are built in to Istio.
