# The application

In this lab you will deploy an application to your mesh.

- The application consists of two microservices, `web-frontend` and `customers`.

    ??? info

        The official Istio docs canonical example is the [BookInfo aplpication](https://istio.io/latest/docs/examples/bookinfo/){target=_blank}.

        For this workshop we felt that an application involving fewer microservices would be more clear.

- The `customers` service exposes a REST endpoint that returns a list of customers in JSON format.  The `web-frontend` calls `customers` to retrieve the list, which it uses to render to HTML.

- The respective Docker images for these services have already been built and pushed to a Docker registry.

- You will deploy the application to the `default` Kubernetes namespace.

But before proceeding, we must enable sidecar injection.

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

You can list the mutating webhooks in your Kubernetes cluster and confirm that the sidecar injector is present.

```{.shell .language-shell}
kubectl get mutatingwebhookconfigurations
```

??? example "Optional: Deeper into sidecar injection"

    _If you have extra time_, explore sidecar injection further with the following exercise.

    1. Start with a pod yaml:

        ```shell
        kubectl run mywebserver --image nginx --dry-run=client -oyaml > nginx-pod.yaml
        ```

    1. Generate the full sidecar-injected manifest:

        ```shell
        istioctl kube-inject -f ./nginx-pod.yaml > injected.yaml
        ```

    1.  Review the `injected.yaml` init-container `args` field:

        ```shell
        istio-iptables
        -p "15001"
        -z "15006"
        -u "1337"
        -m REDIRECT
        -i '*'
        -x ""
        -b '*'
        -d 15090,15021,15020
        ```

    1. Pull the container image and inspect it:

        ```shell
        docker pull docker.io/istio/proxyv2:1.12.2
        docker inspect 1c377f13f99f | grep Entrypoint -A 1
        ```

        ```json
        "Entrypoint": [
            "/usr/local/bin/pilot-agent"
        ```


    We learn that `istio-iptables` is a `pilot-agent` subcommand.

    1. Create a separate namespace that is not labeled for automatic injection

        ```shell
        kubectl create ns myns
        ```

    1. Apply the injected yaml

        ```shell
        kubectl apply -f injected.yaml -n myns
        ```

    1. Study the `pilot-agent istio-iptables` command's flag descriptions:

        ```shell
        kubectl exec mywebserver -n myns -c istio-proxy -it -- pilot-agent istio-iptables --help
        ```

## Deploy the application

1. Study the two Kubernetes yaml files: `web-frontend.yaml` and `customers.yaml`.

    ??? tldr "web-frontend.yaml"
        ```yaml linenums="1"
        --8<-- "web-frontend.yaml"
        ```

    ??? tldr "customers.yaml"
        ```yaml linenums="1"
        --8<-- "customers.yaml"
        ```

    Each file defines its corresponding deployment, service account, and ClusterIP service.

1. Apply the two files to your Kubernetes cluster.

    ```{.shell .language-shell}
    kubectl apply -f customers.yaml
    ```

    ```{.shell .language-shell}
    kubectl apply -f web-frontend.yaml
    ```

Confirm that:

- Two pods are running, one for each service
- Each pod consists of two containers, the one running the service image, plus the Envoy sidecar

    ```{.shell .language-shell}
    kubectl get pod
    ```

## Verify access to each service

We wish to deploy a pod that runs a `curl` image so we can verify that each service is reachable from within the cluster.
The Istio distribution provides a sample app called `sleep` that will serve this purpose.

1. Deploy `sleep` to the default namespace.

    ??? tldr "sleep.yaml"
        ```yaml linenums="1"
        --8<-- "sleep/sleep.yaml"
        ```

    ```{.shell .language-shell}
    kubectl apply -f sleep.yaml
    ```

1. Capture the name of the sleep pod to an environment variable

    ```{.shell .language-shell}
    SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
    ```

1. Use the `kubectl exec` command to call the `customers` service.

    ```{.shell .language-shell}
    kubectl exec $SLEEP_POD -it -- curl customers
    ```

    The console output should show a list of customers in JSON format.

1. Call the `web-frontend` service

    ```{.shell .language-shell}
    kubectl exec $SLEEP_POD -it -- curl web-frontend
    ```

    The console output should show an HTML page listing customers in an HTML table.

## Next

In the next lab, we expose the `web-frontend` using an Istio Ingress Gateway.

This will allow us to access this application on the web.
