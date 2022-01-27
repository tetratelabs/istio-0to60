# The application

In this lab you will deploy an application to your mesh.

- The application consists of two microservers, `web-frontend` and `customers`.

    ??? info

        The official Istio docs canonical example is the [BookInfo aplpication](https://istio.io/latest/docs/examples/bookinfo/).

        For this workshop we felt that an application involving fewer microservices would be more clear.

- The `customers` service exposes a REST endpoint that returns a list of customers in JSON format.  The `web-frontend` calls `customers` retrieves the list, and uses the informationt to render the customer listing in HTML.

- The respective docker images for these services have already been built and pushed to a docker repository.

- You will deploy the application to the `default` Kubernetes namespace.

But before proceeding, we must enable sidecar injection.

## Enable automatic sidecar injection

There are two options for [sidecar injection](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/): automatic and manual.

In this lab we will use automatic injection, which involves labeling the namespace where the pods are to reside.

1.  Label the default namespace

    ```shell
    kubectl label namespace default istio-injection=enabled
    ```

1. Verify that the label has been applied:

    ```shell
    kubectl get ns -Listio-injection
    ```

You can list the mutating webhooks in your Kubernetes cluster and confirm that the sidecar injector is present.

```shell
kubectl get mutatingwebhookconfigurations
```

If you have extra time, explore the `istioctl kube-inject` command.

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

    ```shell
    kubectl apply -f customers.yaml
    ```

    ```shell
    kubectl apply -f web-frontend.yaml
    ```

Confirm that:

- Two pods are running, one for each service
- Each pod consists of two containers, the one running the service image, plus the envoy sidecar

    ```shell
    kubectl get pod
    ```

## Verify access to each service

We wish to deploy a pod that runs a `curl` image so we can verify that each service is reachable from within the cluster.
The Istio distribution comes with a sample called `sleep` that will serve this purpose.

1. Deploy `sleep` to the default namespace.

    ??? tldr "sleep.yaml"
        ```yaml linenums="1"
        --8<-- "sleep/sleep.yaml"
        ```

    ```shell
    kubectl apply -f sleep.yaml
    ```

1. Capture the name of the sleep pod to an environment variable

    ```shell
    SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
    ```

1. Use the `kubectl exec` command to call the `customer` service.

    ```shell
    kubectl exec $SLEEP_POD -it -- curl customers
    ```

    The console output should show a list of customers in JSON format.

1. Call the `web-frontend` service

    ```shell
    kubectl exec $SLEEP_POD -it -- curl web-frontend
    ```

    The console output should show an HTML page listing customers using an HTML table.

## Next

In the next lab, we expose the `web-frontend` using an Istio Ingress Gateway.

This will allow us to see this application in a web browser.
