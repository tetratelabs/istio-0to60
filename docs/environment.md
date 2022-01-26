# Lab environment

An environment has been provisioned for you on Google Cloud Platform (GCP), consisting mainly of a Kubernetes cluster using Google Kubernetes Engine (GKE).

## Log in to GCP

Log in to [GCP](https://cloud.google.com/) using credentials provided by your instructor.

[instructions tbd]

## Select your project

Select the project you have been assigned.

[instructions tbd]

## Launch the Cloud Shell

The Google Cloud Shell will serve as your terminal environment for these labs.

[instructions tbd]

## Verify cluster access

1. Check that `kubectl` is installed

    ```shell
    kubectl version
    ```

1. Verify that your kubernetes context is set to your cluster

    ```shell
    kubectl config get-contexts
    ```

1. List namespaces

    ```shell
    kubectl get ns
    ```

## Artifacts

The lab instructions reference Kubernetes yaml artifacts that you will need to apply to your cluster at specific points in time.

You have the option of copying and pasting the yaml directly from the lab instructions.

Another, perhaps simpler method is to clone the [GitHub repository for this workshop](https://github.com/tetratelabs/istio-0to60) to your Google Cloud Shell environment.  You will find all yaml artifacts in the subdirectory named `artifacts`.

```shell
git clone https://github.com/tetratelabs/istio-0to60.git
```

## Next

Now that we have access to our environment and to our Kubernetes cluster, we can proceed to install Istio.
