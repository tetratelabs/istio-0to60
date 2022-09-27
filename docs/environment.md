# Lab environment

## Options

1. If you brought your own Kubernetes cluster:

    - Kubernetes versions 1.16 through 1.24 should all work.  Feel free to consult the Istio [support status of Istio releases page](https://istio.io/latest/docs/releases/supported-releases/#support-status-of-istio-releases){target=_blank} for version {{istio.version}}.

    - We recommend a 3-worker node cluster of machine type "e2-standard-2" or similar, though a smaller cluster will likely work just fine.

    If you have your own public cloud account:

    - On GCP, the following command should provision a GKE cluster of adequate size for the workshop:

        ```shell
        gcloud container clusters create my-istio-cluster \
          --cluster-version latest \
          --machine-type "e2-standard-2" \
          --num-nodes "3" \
          --network "default"
        ```

    - Feel free to provision a K8S cluster on any infrastructure of your choosing.

1. If you received Google credentials from the workshop instructors:

    - A Kubernetes cluster has already been provisioned for you.
    - Your instructor will demonstrate the process of accessing and configuring your environment, described below.
    - The instructions below explain in detail how to access your account, select your project, and launch the cloud shell.

1. Killercoda:  If you prefer to do away with having to setup your own Kubernetes environment, Killercoda offers a simple browser-based interactive environment.  The Istio 0 to 60 scenarios have been ported to Killercoda and can be launched from [here](https://killercoda.com/eitansuez/).

    If you choose this option, please disregard this page's remaining instructions.

1. Local:  Yet another option is to run a Kubernetes cluster on your local machine using Minikube, Kind, or similar tooling.  This option entails minimum resource (cpu and memory) requirements *and* you will need to ensure that ingress to loadbalancer-type services functions.  Here is a recipe for creating a local Kubernetes cluster with [k3d](https://k3d.io/):

    ```shell
    k3d cluster create my-istio-cluster \
        --api-port 6443 \
        --k3s-arg "--disable=traefik@server:0" \
        --port 80:80@loadbalancer \
        --port 443:443@loadbalancer
    ```

Be sure to:

- Configure your `kubeconfig` file to point to your cluster.
- Follow the instructions [at the bottom of this page](#artifacts) to download the artifacts you will need for the upcoming labs.


If you are bringing your own Kubernetes cluster, please skip ahead to the [artifacts](#artifacts) section at the bottom of this page.

## Log in to GCP

1. Log in to [GCP](https://console.cloud.google.com/){target=_blank} using credentials provided by your instructor.
1. Agree to the terms
1. You will be prompted to select your country, click "Agree and continue"

## Select your project

Select the GCP project you have been assigned, as follows:

1. Click the project selector "pulldown" menu from the top banner, which will open a popup dialog
1. Make sure the _Select from_ organization is set to _tetratelabs.com_
1. Select the tab named _All_
1. You will see your GCP project name (istio-0to60..) listed under the organization tetratelabs.com
1. Select the project from the list

Verify that your project is selected:

- If you look in the banner now, you will see your selected project displayed.

## Launch the Cloud Shell

The Google Cloud Shell will serve as your terminal environment for these labs.

- Click the _Activate cloud shell_ icon (top right); the icon looks like this: :material-console:{.gcp-blue}
- A dialog may pop up, click _Continue_
- Your cloud shell terminal should appear at the bottom of the screen
- Feel free to expand the size of the cloud shell, or even open it in a separate window (locate the icon button :material-open-in-new: in the terminal header, on the right)

!!! warning

    Your connection to the Cloud Shell gets severed after a period of inactivity.
    Click on the _Reconnect_ button when this happens.

## Configure cluster access

1. Check that the `kubectl` CLI is installed

    ```{.shell .language-shell}
    kubectl version --short
    ```

1. Generate a kubeconfig entry

    === "With the user interface"

        1. Activate the top navigation menu (++context-menu++ icon on the top left hand side of the page)
        1. Locate and click on the product _Kubernetes Engine_ (you may have to scroll down until you see it)
        1. Your pre-provisioned 3-node Kubernetes cluster should appear in the main view
        1. Click on that row's "three dot" menu and select the _Connect_ option
        1. A dialog prompt will appear with instructions
        1. Copy the `gcloud` command shown and paste it in your cloud shell

    === "From the command line"

        ```{.shell .language-shell}
        gcloud container clusters get-credentials \
            $(gcloud container clusters list --format="value(name)") \
            --zone $(gcloud container clusters list --format="value(location)") \
            --project $(gcloud config get-value project)
        ```

    Click _Authorize_ when prompted

    The console message will state that a _kubeconfig entry [was] generated for [your project]_

1. Verify that your Kubernetes context is set for your cluster

    ```{.shell .language-shell}
    kubectl config get-contexts
    ```

1. Run a token command such as `kubectl get node` or `kubectl get ns` to ensure that you can communicate with the Kubernetes API Server.

    ```{.shell .language-shell}
    kubectl get ns
    ```

!!! tip

    This workshop makes extensive use of the `kubectl` CLI.

    Consider configuring an alias to make typing a little easier.

    ```shell
    cat << EOF >> ~/.bashrc

    source <(kubectl completion bash)
    alias k=kubectl
    complete -F __start_kubectl k

    EOF

    source ~/.bashrc
    ```


All instructions in subsequent labs assume you will be working from the Google Cloud Shell.

## Artifacts

The lab instructions reference Kubernetes yaml artifacts that you will need to apply to your cluster at specific points in time.

You have the option of copying and pasting the yaml snippets directly from the lab instructions as you encounter them.

Another option is to clone the [GitHub repository for this workshop](https://github.com/tetratelabs/istio-0to60){target=_blank} from the Cloud Shell.  You will find all yaml artifacts in the subdirectory named `artifacts`.

```shell
git clone https://github.com/tetratelabs/istio-0to60.git && \
  mv istio-0to60/artifacts . && \
  rm -rf istio-0to60
```

## Next

Now that we have access to our environment and to our Kubernetes cluster, we can proceed to install Istio.
