# Deeper into sidecar injection

The following exercise explores sidecar injection further.

1. Start with a pod yaml:

    ```shell
    kubectl run mywebserver --image nginx \
      --dry-run=client -oyaml > nginx-pod.yaml
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
    kubectl exec mywebserver -n myns \
      -c istio-proxy -it \
      -- pilot-agent istio-iptables --help
    ```
