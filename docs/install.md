# Install Istio

In this lab you will install Istio.


## Download Istio

1. Run the following command from your home directory.

    ```shell
    curl -L https://istio.io/downloadIstio | sh -
    ```

1. Navigate into the directory created by the above command.

    ```shell
    cd istio-1.12.2
    ```


## Add `istioctl` to your PATH

The `istioctl` CLI is located in the `bin/` subdirectory.

!!! note

    Cloud Shell only preserves files located inside your home directory across sessions.

    This means that if you install a binary to a `PATH` such as `/usr/local/bin`, chances are tomorrow that file will no longer be there!

    As a workaround, you will add `${HOME}/bin` to your `PATH` and place the binary there.


1. Create a `bin` subdirectory in your home directory:

    ```shell
    mkdir ~/bin
    ```

1. Copy the CLI to that subdirectory:

    ```shell
    cp ./bin/istioctl ~/bin
    ```

1. Add your home `bin` subdirectory to your PATH

    ```shell
    echo "export PATH=$HOME/bin:$PATH" >> ~/.bashrc
    source ~/.bashrc
    ```

Verify that `istioctl` is installed with:

```shell
istioctl version
```

With the CLI installed, proceed to install Istio to Kubernetes.

## Install Istio

1. Istio can be installed directly with the cli:

    ```shell
    istioctl install
    ```

1. When prompted, enter `y` to proceed to install Istio.

Take a moment to learn more about [Istio installation profiles](https://istio.io/latest/docs/setup/additional-setup/config-profiles/).

## Verify that Istio is installed

1. List Kubernetes namespaces and note the new namespace `istio-system`

    ```shell
    kubectl get ns
    ```

1. Verify that the `istiod` controller pod is running in that namespace

    ```shell
    kubectl get pod -n istio-system
    ```

1. Re-run `istioctl version`.  The output should include a _control plane_ version, indicating that Istio is indeed present in the cluster.

## Next

With Istio installed, we are ready to deploy an application to the mesh.