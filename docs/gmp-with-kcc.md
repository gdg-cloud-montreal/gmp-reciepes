# Deploy GKE with GMP enabled via Kubernetes Config Connector

To start we will need to create a cluster with the Config Connector Resources install, this can be GKE or a compliant K8s distributions (ie kind), see [here](https://cloud.google.com/config-connector/docs/concepts/installation-types) for installation types. 

In this example I will be using the Manual installation type so we can use the latest version of Config Connector (1.91 at the time of this writing) and a GKE cluster so we can use Workload Identity.

#  Deploy GKE Cluster with GMP enabled via Config Connector

1. Create a Kubernetes Cluster

In this example I will highlight the two ways to install Config Connector. The first will be the manual mode which can be used on both GKE clusters as well as other CNCF conformant clusters.

The second option will be using Config Controller which is a pre-configured managed cluster that comes pre-loaded with Config Connector as well as Config Sync and Policy Controller.

## Manual Install

1. Create a Kubernetes Cluster

    ```
    gcloud container clusters create kcc-configs --machine-type "e2-standard-4" --image-type "COS_CONTAINERD" \
    --num-nodes "3" --enable-ip-alias --project $PROJECT_ID --zone northamerica-northeast1-a \
    --workload-pool=${PROJECT_ID}.svc.id.goog
    ```

    ```
    gcloud container clusters get-credentials kcc-configs --region northamerica-northeast1-a
    ```
    
2. Install Config Connector

    - Download the latest Config Connector operator
        ```
        gsutil cp gs://configconnector-operator/latest/release-bundle.tar.gz release-bundle.tar.gz
        ```
    - Extract Tar
        ```
        tar zxvf release-bundle.tar.gz
        ```
    - Install the Operator
        ```
        kubectl apply -f operator-system/configconnector-operator.yaml
        ```
    - Create an Identity
        ```
        gcloud iam service-accounts create gmp-demo
        ```
    - Assign IAM Permissions
        ```
        gcloud projects add-iam-policy-binding ${PROJECT_ID} \
        --member="serviceAccount:gmp-demo@${PROJECT_ID}.iam.gserviceaccount.com" \
        --role="roles/editor"
        ```
    - Bind SA to Kubernetes SA for Config Connector to use
        ```
        gcloud iam service-accounts add-iam-policy-binding \
        gmp-demo@${PROJECT_ID}.iam.gserviceaccount.com \
        --member="serviceAccount:${PROJECT_ID}.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
        --role="roles/iam.workloadIdentityUser"
        ```
    - Configure Config Connector

```
cat << EOF>> configconnector.yaml
apiVersion: core.cnrm.cloud.google.com/v1beta1
kind: ConfigConnector
metadata:
    # the name is restricted to ensure that there is only one
    # ConfigConnector resource installed in your cluster
    name: configconnector.core.cnrm.cloud.google.com
spec:
    mode: cluster
    googleServiceAccount: "gmp-demo@${PROJECT_ID}.iam.gserviceaccount.com"
EOF
```

    - Deploy
        ```
        kubectl apply -f configconnector.yaml
        ```
    - Config Namespace for the resources and annotate it.
        ```
        kubectl create namespace config-control
        ```

        ```
        kubectl annotate namespace \
        config-control cnrm.cloud.google.com/project-id=${PROJECT_ID}
        ```
3. Create the Container Cluster manifest.

```
cat << EOF>> configconnector.yaml
apiVersion: core.cnrm.cloud.google.com/v1beta1
kind: ConfigConnector
metadata:
    # the name is restricted to ensure that there is only one
    # ConfigConnector resource installed in your cluster
    name: configconnector.core.cnrm.cloud.google.com
spec:
    mode: cluster
    googleServiceAccount: "gmp-demo@${PROJECT_ID}.iam.gserviceaccount.com"
EOF



cat >gmp-cluster.yaml <<EOL
apiVersion: container.cnrm.cloud.google.com/v1beta1
kind: ContainerCluster
metadata:
    labels:
        availability: dev
        target-audience: development
    name: gmp-enabled-cluster
    namespace: gmp-cluster
spec:
    description: A GMP enabled cluster
    location: northamerica-northeast1-a
    initialNodeCount: 1
    networkingMode: ROUTES
    clusterIpv4Cidr: 10.96.0.0/14
    masterAuthorizedNetworksConfig:
        cidrBlocks:
        - displayName: Trusted external network
            cidrBlock: 10.2.0.0/16
    addonsConfig:
        gcePersistentDiskCsiDriverConfig:
            enabled: true
        horizontalPodAutoscaling:
            disabled: true
        httpLoadBalancing:
            disabled: false
    loggingConfig:
        enableComponents:
        - "SYSTEM_COMPONENTS"
        - "WORKLOADS"
    monitoringConfig:
        enableComponents:
        - "SYSTEM_COMPONENTS"
        managedPrometheus:
            enabled: true
    workloadIdentityConfig:
        # Replace ${PROJECT_ID?} with your project ID.
        workloadPool: "${PROJECT_ID}.svc.id.goog"
EOL
```

=======
        config-control cnrm.cloud.google.com/project-id=${PROJECT_ID}
        ```
## Config Controller
1. Create a [Config Controller](https://cloud.google.com/anthos-config-management/docs/concepts/config-controller-overview) instance. For this I will assume you have a `default` network  setup but you can use a network of your choosing using the `--network=you-network --subnet=subnet`.

```
gcloud services enable krmapihosting.googleapis.com \
    container.googleapis.com \
    cloudresourcemanager.googleapis.com
gcloud anthos config controller create gmp-recipes --location northamerica-northeast1
```

2. Authenticate with the Instance

```
gcloud anthos config controller get-credentials gmp-recipes \
    --location northamerica-northeast1
```

3. Assign permissions to the Config Connector SA
```
export SA_EMAIL="$(kubectl get ConfigConnectorContext -n config-control \
    -o jsonpath='{.items[0].spec.googleServiceAccount}' 2> /dev/null)"
gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
    --member "serviceAccount:${SA_EMAIL}" \
    --role "roles/owner" \
    --project "${PROJECT_ID}"
```

The install will take about 15 minutes as the cluster gets provisioned and the resources are installed.

## Deploy the GMP Enabled Cluster

1. Create the Container Cluster manifest.

    ```
    cat >gmp-cluster.yaml <<EOL
    apiVersion: container.cnrm.cloud.google.com/v1beta1
    kind: ContainerCluster
    metadata:
        labels:
            availability: dev
            target-audience: gdg-cloud-montreal
        name: gmp-enabled-cluster
        namespace: config-control
    spec:
        description: A GMP enabled cluster
        location: northamerica-northeast1-a
        initialNodeCount: 1
        networkingMode: ROUTES
        clusterIpv4Cidr: 10.96.0.0/14
        masterAuthorizedNetworksConfig:
            cidrBlocks:
            - displayName: Trusted external network
              cidrBlock: 10.2.0.0/16
        addonsConfig:
            gcePersistentDiskCsiDriverConfig:
                enabled: true
            horizontalPodAutoscaling:
                disabled: true
            httpLoadBalancing:
                disabled: false
        loggingConfig:
            enableComponents:
            - "SYSTEM_COMPONENTS"
            - "WORKLOADS"
        monitoringConfig:
            enableComponents:
            - "SYSTEM_COMPONENTS"
            managedPrometheus:
                enabled: true
        workloadIdentityConfig:
            # Replace ${PROJECT_ID?} with your project ID.
            workloadPool: "${PROJECT_ID}.svc.id.goog"
    EOL
    ```

>>>>>>> 124c4629e3a48381b84cf9407ff56bdeb9a68457
4. Deploy the Manifest.

    ```
    kubectl apply -f gmp-cluster.yaml
    ```

## Verifying Installation

To verify the cluster is up and running we can run the following command or go to the gke console.

```
<<<<<<< HEAD
kubectl get containerclusters -n gmp-cluster
=======
kubectl get containerclusters -n config-control 
>>>>>>> 124c4629e3a48381b84cf9407ff56bdeb9a68457
```
Output:
```
NAME                  AGE   READY   STATUS     STATUS AGE
gmp-enabled-cluster   22m   True    UpToDate   3m3s
```

Congrats, you've just deployed a GMP enabled cluster via Config Connector!