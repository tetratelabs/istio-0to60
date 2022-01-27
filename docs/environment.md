# Lab environment

An environment has been provisioned for you on Google Cloud Platform (GCP), consisting mainly of a Kubernetes cluster.

Your instructor will demonstrate the process of accessing and configuring your environment, described below.

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

        1. Activate the top navigation menu (hamburger icon on the top left hand side of the page)
        1. Locate and click on the product _Kubernetes Engine_ (you may have to scroll down until you see it)
        1. Your pre-provisioned 3-node Kubernetes cluster should appear in the main view
        1. Click on that row's "three dot" menu and select the _Connect_ option
        1. A dialog prompt will appear with instructions
        1. Copy the `gcloud` command shown and paste it in your cloud shell

    === "From the command line"

        ```{.shell .language-shell}
        gcloud container clusters get-credentials \
            $(gcloud container clusters list --format="value(name)") \
            --zone us-central1-a \
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

All instructions in subsequent labs assume you will be working from the Google Cloud Shell.

## Artifacts

The lab instructions reference Kubernetes yaml artifacts that you will need to apply to your cluster at specific points in time.

You have the option of copying and pasting the yaml snippets directly from the lab instructions as you encounter them.

Another option is to clone the [GitHub repository for this workshop](https://github.com/tetratelabs/istio-0to60){target=_blank} from the Cloud Shell.  You will find all yaml artifacts in the subdirectory named `artifacts`.

```{.shell .language-shell}
git clone https://github.com/tetratelabs/istio-0to60.git
```

## Next

Now that we have access to our environment and to our Kubernetes cluster, we can proceed to install Istio.
