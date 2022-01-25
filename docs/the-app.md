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

You can list the mutating webhooks in your kubernetes cluster and confirm that the sidecar injector is present.

```shell
kubectl get mutatingwebhookconfigurations
```

If you have extra time, explore the `istioctl kube-inject` command.

## Deploy the applications

