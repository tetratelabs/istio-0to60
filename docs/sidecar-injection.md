# Sidecar injection

This lab explores sidecar injection in Istio.

## Preface

Istio provides both a manual and an automatic mechanism for injecting sidecars alongside workloads.

In this lab you will use the manual method, because it provides the opportunity to inspect the transformed deployment manifest even before applying it to a target Kubernetes cluster.

You will learn about automatic sidecar injection in the next lab.

## Generate a Pod spec

The `kubectl` command's `dry-run` flag provides a simple way to generate and capture a simple pod specification.

Generate a Pod spec for a simple web server, as follows:

```shell
kubectl run mywebserver --image nginx \
    --dry-run=client -oyaml > nginx-pod.yaml
```

Inspect the contents of the generated file.  Here it is below, slightly cleaned up:

```yaml linenums="1" title="nginx-pod.yaml"
--8<-- "nginx-pod.yaml"
```

The main thing to note at this point is that this Pod spec consists of a single container using the image `nginx`.

## Transform the Pod spec

The `istioctl` command provides the convenient `kube-inject` subcommand, that can transform such a specification into one that includes the necessary sidecar.

1. Learn the `kube-inject` command's usage:

    ```shell
    istioctl kube-inject --help
    ```

1. Use the command to generate and capture the full sidecar-injected manifest to a new file named `transformed.yaml`.

    ??? help "Show me how"
        ```shell
        istioctl kube-inject --filename ./nginx-pod.yaml > transformed.yaml
        ```

## Study the sidecar container specification

The modified Pod specification now includes a second container.

Here is the salient part:

```yaml linenums="1"
  - name: istio-proxy
    image: docker.io/istio/proxyv2:{{istio.version}}
    args:
    - proxy
    - sidecar
    - --domain
    - $(POD_NAMESPACE).svc.cluster.local
    - --proxyLogLevel=warning
    - --proxyComponentLogLevel=misc:error
    - --log_output_level=default:info
    - --concurrency
    - "2"
    env:
    - ...
```

The container name is `istio-proxy` and the docker image is `istio/proxyv2`.

???+ info "What command is actually run?"

    To find out what command actually runs inside that container, we can inspect the docker container specification and view the Entrypoint field:

    ```shell
    docker pull docker.io/istio/proxyv2:{{istio.version}}
    docker inspect istio/proxyv2:{{istio.version}} | grep Entrypoint -A 2
    ```

    Here is the output:

    ```json
    "Entrypoint": [
        "/usr/local/bin/pilot-agent"
    ],
    ```

    We learn that the name of the command is `pilot-agent`.

By extracting the arguments from the yaml, we can reconstitute the full command executed inside the sidecar container:

```shell
pilot-agent proxy sidecar \
    --domain $(POD_NAMESPACE).svc.cluster.local \
    --proxyLogLevel=warning \
    --proxyComponentLogLevel=misc:error \
    --log_output_level=default:info \
    --concurrency "2"
```

## Apply the manifest

1. Deploy the transformed manifest to Kubernetes:

    ```shell
    kubectl apply -f transformed.yaml
    ```

1. List pods in the `default` namespace

    ```shell
    kubectl get pod
    ```

    Once the pod reaches `Running` state, note the `READY` column in the output displays 2 out of 2 containers:

    ```console
    NAME          READY   STATUS    RESTARTS   AGE
    mywebserver   2/2     Running   0          36s
    ```

## Study the running processes

Run the `ps` command from inside the sidecar container, like so:

```shell
kubectl exec mywebserver -c istio-proxy -- ps -ef
```

Here is the output, slightly cleaned up, showing both the `pilot-agent` process, and the `envoy` process that it bootstrapped:

```console
PID  PPID CMD
  1     0 /usr/local/bin/pilot-agent proxy sidecar --domain ...
 16     1 /usr/local/bin/envoy -c etc/istio/proxy/envoy-rev.json ...
```

We can learn more about the `pilot-agent` command by running `pilot-agent --help` from inside the sidecar container:

```shell
kubectl exec mywebserver -c istio-proxy -- pilot-agent --help
```

## Study the `initContainers` specification

Besides injecting a sidecar container, the transformation operation also adds an [initContainers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/){target=_blank} section.

Here is the relevant section:

```yaml
  initContainers:
  - name: istio-init
    image: docker.io/istio/proxyv2:{{istio.version}}
    args:
    - istio-iptables
    - -p
    - "15001"
    - -z
    - "15006"
    - -u
    - "1337"
    - -m
    - REDIRECT
    - -i
    - '*'
    - -x
    - ""
    - -b
    - '*'
    - -d
    - 15090,15021,15020
    - --log_output_level=default:info
```

The "initContainer" uses the same image as the sidecar container: `istio/proxyv2`.  The difference lies in the command that is run when the Pod initializes.

Here is the reconstituted command with long-form versions of each option, to clarify the instruction:

```shell
pilot-agent istio-iptables \
    --envoy-port "15001" \
    --inbound-capture-port "15006" \
    --proxy-uid "1337" \
    --istio-inbound-interception-mode REDIRECT \
    --istio-service-cidr '*' \
    --istio-service-exclude-cidr "" \
    --istio-inbound-ports '*' \
    --istio-local-exclude-ports 15090,15021,15020 \
    --log_output_level=default:info
```

???+ tip

    For a full description of the `istio-iptables` subcommand and its options, run:

    ```shell
    kubectl exec mywebserver -c istio-proxy -- pilot-agent istio-iptables --help
    ```

The gist of the command is that, through `iptables` rules, the routing of network packets inside the Pod is reconfigured to give Envoy the chance to intercept and proxy inbound and outbound traffic.

We need not concern ourselves with the specific port numbers, exclusions, and other low-level details at this time.

The lesson of this exercise is to learn how to get at these details.

## Going forward..

The above process of transforming a deployment manifest on its way to the Kube API Server is streamlined when using automatic sidecar injection.

The next lab will walk you through how automatic sidecar injection is accomplished.

From here on, we will use automatic sidecar injection when deploying workloads to the mesh.

## Cleanup

```shell
kubectl delete -f transformed.yaml
```
