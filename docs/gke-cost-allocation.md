# GKE Cost Allocation

## 1 Configure GKE Cost Allocation

### 1.1 Enable Detailed usage cost data in BQ

Ensure that you have completed the steps to [Export detailed usage cost data to BigQuery](https://cloud.google.com/billing/docs/how-to/export-data-bigquery). If your organization is already exporting data, you must have access to query the tables.

### 1.2 Create GKE Cluster on GCP with GKE Cost Allocation enabled

**Step 1** Enable the Google Kubernetes Engine API.
```
gcloud services enable container.googleapis.com
```

**Step 2** From the cloud shell, run the following command to create a cluster with 1 node:

```
gcloud beta container clusters create k8s-cost-allocation-labs \
--region us-central1 \
--enable-ip-alias \
--enable-network-policy \
--machine-type "n1-standard-4" \
--release-channel regular \
--enable-cost-allocation
```

```
gcloud container clusters get-credentials k8s-cost-allocation-labs --region us-central1
```


### 1.3 Deploy Test Application 



#### 1.2.3 (Optional) Deploy onlineboutique application and observe app compute resources

Deploy microservices application `onlineboutique`:

**Step 1** Create Namespace `onlineboutique`

```
kubectl create ns onlineboutique
```

**Step 2** Deploy Microservice application
```
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
cd microservices-demo
```

```
kubens onlineboutique
kubectl apply -f ./release/kubernetes-manifests.yaml
```

**Step 3** Verify Deployment:

```
kubectl get pods
```

**Step 4** Observe Detailed Billing Information for Namespaces:


**Step 5** Observe Detailed Billing Information for Pods:

