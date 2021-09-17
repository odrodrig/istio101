# Exercise 1 - Accessing an OpenShift cluster with IBM Cloud Kubernetes Service

You must already have an OpenShift cluster in order to complete the lab. Your cluster must have **3 or more worker nodes** with at least **4 cores and 16GB RAM** and run OpenShift 4.6 or greater.

## Access IBM Cloud Shell 

For this workshop we will be using IBM Cloud Shell which contains many tools needed without having to worry about installing anything locally. You can access the IBM Cloud Shell [here](https://cloud.ibm.com/shell).

## Access your cluster

In order to complete the lab, you will need to access to your OpenShift cluster from your terminal. This can be done by following the instructions found [here](https://ibm.github.io/workshop-setup/ROKS/)

## Clone the lab repo

1. From your command line, run:

    ```shell
    export WORK_DIR=$(pwd)

    git clone -b openshift-service-mesh https://github.com/odrodrig/istio101

    ```

    This is the working directory for the workshop. You will use the example `.yaml` files that are located in the `workshop/plans` directory in the following exercises.

### [Continue to Exercise 2 - Installing OpenShift Service Mesh](../exercise-2/README.md)
