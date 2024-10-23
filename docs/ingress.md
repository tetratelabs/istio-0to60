# Ingress

The objective of this lab is to expose the `web-frontend` service to the internet.

## The Ingress gateway

When you installed Istio, in addition to deploying `istiod` to Kubernetes, the installation also provisioned an Ingress Gateway.

View the corresponding Istio ingress gateway pod in the `istio-system` namespace.

```{.shell .language-shell}
kubectl get pod -n istio-system
```

A corresponding _LoadBalancer_ type service was also created:

```{.shell .language-shell}
kubectl get svc -n istio-system
```

Make a note of the external IP address for the load balancer.

Assign it to an environment variable.

```{.shell .language-shell}
export GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway \
    -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
```

!!! warning "When using K3D"

    If you have opted to run Kubernetes directly on your local machine with K3D, use "127.0.0.1" instead:

    ```{.shell .language-shell}
    export GATEWAY_IP=127.0.0.1
    ```

??? tip ":material-console:{.gcp-blue} A small investment"

    When the cloud shell connection is severed, or when opening a new terminal tab, `$GATEWAY_IP` will no longer be in scope.

    Ensure `GATEWAY_IP` is set each time we start a new shell:

    ```{.shell .language-shell}
    cat << EOF >> ~/.bashrc

    export GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')

    EOF
    ```

In normal circumstances we associate this IP address with a hostname via DNS.
For the sake of simplicity, in this workshop we will use the gateway public IP address directly.

## Configuring ingress

Configuring ingress with Istio is performed in two parts:

1. Define a `Gateway` Custom Resource that governs the specific host, port, and protocol to expose.
1. Specify how requests should be routed with a `VirtualService` Custom Resource.

### Create a Gateway resource

1. Review the following Gateway specification.

    !!! tldr "gateway.yaml"
        ```yaml linenums="1"
        --8<-- "ingress/gateway.yaml"
        ```

    Above, we specify the HTTP protocol, port 80, and a wildcard ("*") host matcher which ensures that HTTP requests using the load balancer IP address `$GATEWAY_IP` will match.

    The selector _istio: ingressgateway_ selects the Envoy gateway workload to be configured, the one residing in the `istio-system` namespace.

1. Apply the gateway resource to your cluster.

    ```{.shell .language-shell}
    kubectl apply -f gateway.yaml
    ```

1. Attempt an HTTP request in your browser to the gateway IP address.

    ```shell
    curl -sv http://$GATEWAY_IP/ | head
    ```

    It should return a 404: not found.

### Create a VirtualService resource

1. Review the following VirtualService specification.

    ???+ tldr "web-frontend-virtualservice.yaml"
        ```yaml linenums="1"
        --8<-- "ingress/web-frontend-virtualservice.yaml"
        ```

    Note how this specification references the name of the gateway ("frontend-gateway"), a matching host ("*"), and specifies a route for requests to be directed to the `web-frontend` service.

1. Apply the VirtualService resource to your cluster.

    ```{.shell .language-shell}
    kubectl apply -f web-frontend-virtualservice.yaml
    ```

1. List virtual services in the default namespace.

    ```{.shell .language-shell}
    kubectl get virtualservice
    ```

    The output indicates that the VirtualService named `web-frontend` is bound to the gateway `frontend-gateway`, as well as any hostname that routes to the load balancer IP address.

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
        kubectl create -n istio-system secret tls webfrontend-credential \
          --key=example_certs/webfrontend.example.com.key \
          --cert=example_certs/webfrontend.example.com.crt
        ```

    1. Revise the gateway configuration to listen on port 443, and to reference the secret that the envoy listeners will present to incoming requests:

        ```yaml linenums="1" hl_lines="10-18"
        --8<-- "ingress/gateway-tls.yaml"
        ```

    1. Apply the revised gateway configuration:

        ```{.shell .language-shell}
        kubectl apply -f gateway-tls.yaml
        ```

    1. Test your implementation by making a request to the ingress gateway:

        ```{.shell .language-shell}
        curl -k https://webfrontend.example.com/ --resolve webfrontend.example.com:443:$GATEWAY_IP
        ```

    See the [Istio documentation](https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/){target=_blank} for additional examples relating to the topic of configuring secure gateways.

## Next

The application is now running and exposed on the internet.

In the next lab, we turn our attention to the observability features that are built into Istio.
