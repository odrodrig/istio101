# Exercise 3 - Deploy the Guestbook app with Istio Proxy

The Guestbook app is a sample app for users to leave comments. It consists of a web front end, Redis master for storage, and a replicated set of Redis slaves. 


## Download the Guestbook app

1. Clone the Guestbook app into the `workshop` directory.

    ```shell
    export WORK_DIR=$(pwd)
    git clone -b openshift-service-mesh https://github.com/odrodrig/guestbook.git
    ```

1. Navigate into the app directory.

    ```shell
    cd guestbook/v2
    ```

## Enable the automatic sidecar injection for the default namespace

In Kubernetes, a sidecar is a utility container in the pod, and its purpose is to support the main container. For Istio to work, Envoy proxies must be deployed as sidecars to each pod of the deployment. In Istio on Kubernetes it is possible to enable automatic sidecar injection for all pods in a namespace, however, in OpenShift with OpenShift Service Mesh this is not possible. Instead, you must enable sidecar injection for each deployment. This is so temporary workloads like build containers do not get a sidecar. Enabling sidecar injection in a deployment can be done with a simple annotation in the deployment object.

1. For each deployment that you want to enable sidecar injection with, you will need to add the annotation `sidecar.istio.io/inject: "true"` to the deployment object. For the purposes of this workshop, this has already been done for the guestbook and redis deployments. Run the following command to view the guestbook deployment and the sidecar injection annotation:

    ```bash
    cat $WORK_DIR/guestbook/v2/guestbook-deployment.yaml
    ```
    
    You should see the following:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
        name: guestbook-v2
        labels:
            app: guestbook
            version: "2.0"
    spec:
        selector:
            matchLabels:
                app: guestbook
        replicas: 3
        template:
            metadata:
                labels:
                    app: guestbook
                    version: "2.0"   
                annotations:
                    sidecar.istio.io/inject: "true"
            spec:
                containers:
                - name: guestbook
                    image: ibmcom/guestbook:v2
                    resources:
                        requests:
                            cpu: 100m
                            memory: 100Mi
                    ports:
                    - name: http
                    containerPort: 3000
    ```

    Notice the `sidecar.istio.io/inject: "true"` annotation at `spec.template.metadata.annotations`.

## Create a Redis database

The Redis database is a service that you can use to persist the data of your app. The Redis database comes with a master and slave modules.

1. Create the Redis deployments and services for both the master and the slave.

    ``` shell
    oc create -f redis-master-deployment.yaml
    oc create -f redis-master-service.yaml
    oc create -f redis-slave-deployment.yaml
    oc create -f redis-slave-service.yaml
    ```

1. Verify that the Redis deployments for the master and the slave are created.

    ```shell
    oc get deployment
    ```

    Output:

    ```shell
    NAME           READY   UP-TO-DATE   AVAILABLE   AGE
    redis-master   1/1     1            1           2m16s
    redis-slave    2/2     2            2           2m15s
    ```

1. Verify that the Redis services for the master and the slave are created.

    ```shell
    oc get svc
    ```

    Output:

    ```shell
    NAME           TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
    redis-master   ClusterIP      172.21.85.39    <none>          6379/TCP       5d
    redis-slave    ClusterIP      172.21.205.35   <none>          6379/TCP       5d
    ```

1. Verify that the Redis pods for the master and the slave are up and running.

    ```shell
    oc get pods
    ```

    Output:

    ```shell
    NAME                            READY     STATUS    RESTARTS   AGE
    redis-master-4sswq              2/2       Running   0          5d
    redis-slave-kj8jp               2/2       Running   0          5d
    redis-slave-nslps               2/2       Running   0          5d
    ```

## Install the Guestbook app

1. Inject the Istio Envoy sidecar into the guestbook pods, and deploy the Guestbook app on to the Kubernetes cluster. Deploy both the v1 and v2 versions of the app:

    ```shell
    oc apply -f $WORK_DIR/guestbook/v1/guestbook-deployment.yaml
    oc apply -f $WORK_DIR/guestbook/v2/guestbook-deployment.yaml
    ```

    These commands deploy the Guestbook app on to the OpenShift cluster. Since we enabled automation sidecar injection, these pods will be also include an Envoy sidecar as they are started in the cluster. Here we have two versions of deployments, a new version (`v2`) in the current directory, and a previous version (`v1`) in a sibling directory. They will be used in future sections to showcase the Istio traffic routing capabilities.

1. Create the guestbook service.

    ```shell
    oc create -f $WORK_DIR/guestbook/v2/guestbook-service.yaml
    ```

1. Verify that the service was created.

    ```shell
    oc get svc
    ```

    Output:

    ```shell
    NAME           TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
    guestbook      LoadBalancer   172.21.36.181   169.61.37.140   80:32149/TCP   5d
    ...
    ```

1. Verify that the pods are up and running.

    ```shell
    oc get pods
    ```

    Sample output:

    ```shell
    NAME                            READY   STATUS    RESTARTS   AGE
    guestbook-v1-98dd9c654-dz8dq    2/2     Running   0          30s
    guestbook-v1-98dd9c654-mgfv6    2/2     Running   0          30s
    guestbook-v1-98dd9c654-x8gxx    2/2     Running   0          30s
    guestbook-v2-8689f6c559-5ntgv   2/2     Running   0          28s
    guestbook-v2-8689f6c559-fpzb7   2/2     Running   0          28s
    guestbook-v2-8689f6c559-wqbnl   2/2     Running   0          28s
    redis-master-577bc6fbb-zh5v8    2/2     Running   0          4m47s
    redis-slave-7779c6f75b-bshvs    2/2     Running   0          4m46s
    redis-slave-7779c6f75b-nvsd6    2/2     Running   0          4m46s
    ```

    Note that each guestbook pod has 2 containers in it. One is the guestbook container, and the other is the Envoy proxy sidecar.

## [Continue to Exercise 4 - Telemetry](../exercise-4/README.md)
