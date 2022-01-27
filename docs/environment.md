# Lab environment

An environment has been provisioned for you on Google Cloud Platform (GCP), consisting mainly of a Kubernetes cluster using Google Kubernetes Engine (GKE).

## Log in to GCP

1. Log in to [GCP](https://console.cloud.google.com/) using credentials provided by your instructor.
1. Agree to the terms
1. You will be prompted to select your country, click "Agree and continue"

## Select your project

Select the GCP project you have been assigned, as follows:

1. Click the project selector "pulldown" menu from the top banner, which will open a popup dialog
1. make sure the "Select from" Organization is set to _tetratelabs.com_
1. Select the tab named "All"
1. You will see your GCP project name (istio-0to60..) listed under the organization tetratelabs.com
1. Select the project from the list

Verify that your project is selected:
- If you look in the banner now, you will see your selected project displayed.

## Launch the Cloud Shell

The Google Cloud Shell will serve as your terminal environment for these labs.

- Click the "Activate cloud shell" icon (top right); the icon looks like a shell prompt ">_"
- A dialog may pop up, click "Continue"
- Your cloud shell terminal should appear at the bottom of the screen
- Feel free to expand the size of the cloud shell, or even open it in a separate window (there's a button in the terminal header, on the right)

## Configure cluster access

1. Check that the `kubectl` CLI is installed

    ```shell
    kubectl version --short
    ```

1. From the top navigation menu (hamburger icon on the top left hand side of the page),
1. Locate and click on the product "Kubernetes Engine" (you may have to scroll down until you see it)
1. Your pre-provisioned 3-node Kubernetes cluster should appear in the main view
1. Click on that row's "three dot" menu and select the "Connect" option
1. A dialog prompt will appear with instructions
1. Copy the `gcloud` command shown and paste it in your cloud shell
1. Click "authorize" when prompted

You should see a message in the console that a _kubeconfig entry was generated_

1. Verify that your kubernetes context is set to your cluster

    ```shell
    kubectl config get-contexts
    ```

1. Run a token command such as `kubectl get node` or `kubectl get ns` to ensure that you can communicate with the Kubernetes Api Server.

    ```shell
    kubectl get ns
    ```

All instructions in subsequent labs assume you will be working from inside the Google Cloud Shell.

## Artifacts

The lab instructions reference Kubernetes yaml artifacts that you will need to apply to your cluster at specific points in time.

You have the option of copying and pasting the yaml directly from the lab instructions as you encounter them.

Another, perhaps simpler method is to clone the [GitHub repository for this workshop](https://github.com/tetratelabs/istio-0to60) to your Google Cloud Shell environment.  You will find all yaml artifacts in the subdirectory named `artifacts`.

```shell
git clone https://github.com/tetratelabs/istio-0to60.git
```

## Next

Now that we have access to our environment and to our Kubernetes cluster, we can proceed to install Istio.
