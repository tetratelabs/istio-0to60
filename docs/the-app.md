# The application

In this lab you will deploy an application to your mesh.

- The application consists of two microservices, `web-frontend` and `customers`.

    !!! note "Aside"

        The official Istio docs canonical example is the [BookInfo application](https://istio.io/latest/docs/examples/bookinfo/){target=_blank}.

        For this workshop, we felt that an application involving fewer microservices would be more clear.

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

## Deploy the application

1. Study the two Kubernetes yaml files: `web-frontend.yaml` and `customers.yaml`.

    ??? tldr "web-frontend.yaml"
        ```yaml linenums="1"
        --8<-- "the-app/web-frontend.yaml"
        ```

    ??? tldr "customers.yaml"
        ```yaml linenums="1"
        --8<-- "the-app/customers.yaml"
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
- Each pod consists of two containers, one running the service image, the other runs the Envoy sidecar

    ```{.shell .language-shell}
    kubectl get pod
    ```

!!! question "How did each pod end up with two containers?"

    Istio installs a Kubernetes object known as a [mutating webhook admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/){ target=_blank }: logic that intercepts Kubernetes object creation requests and that has the permission to alter (mutate) what ends up stored in etcd (the pod spec).

    You can list the mutating webhooks in your Kubernetes cluster and confirm that the sidecar injector is present.

    ```{.shell .language-shell}
    kubectl get mutatingwebhookconfigurations
    ```

## Verify access to each service

We wish to deploy a pod that runs a `curl` image so we can verify that each service is reachable from within the cluster.
The Istio distribution provides a sample app called `sleep` that will serve this purpose.

1. Deploy `sleep` to the default namespace.

    ??? tldr "sleep.yaml"
        ```yaml linenums="1"
        --8<-- "the-app/sleep.yaml"
        ```

    ```{.shell .language-shell}
    kubectl apply -f sleep.yaml
    ```

1. Use the `kubectl exec` command to call the `customers` service.

    ```{.shell .language-shell}
    kubectl exec deploy/sleep -- curl -s customers | jq
    ```

    The console output should show a list of customers in JSON format.

1. Call the `web-frontend` service

    ```{.shell .language-shell}
    kubectl exec deploy/sleep -- curl -s web-frontend | head
    ```

    The console output should show the start of an HTML page listing customers in an HTML table.

The application is now deployed and functioning.

## Next

In the next lab, we expose the `web-frontend` using an Istio Ingress Gateway.  This will allow us to access this application on the web.

Alternatively, you have the option of exploring the future [Kubernetes API Gateway](https://gateway-api.sigs.k8s.io/){target=_blank} version of the Ingress lab.

