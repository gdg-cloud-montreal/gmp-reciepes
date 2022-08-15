# Prometheus-operator scraping metrics to GMP


## 1 Configure Prometheus Operator on GKE

The `Prometheus` Operator is a tool developed and opnesourced by CoreOS that aims to automate manual operations of Prometheus and Alertmanager on Kubernetes using Kubernetes Custom Resource Definitions (CRDs).

The `Prometheus` Operator provides easy monitoring definitions for Kubernetes services and deployment and management of Prometheus instances.

Once installed, the Prometheus Operator provides the following features:

  * **Kubernetes Custom Resources:** Use `Kubernetes` custom resources to deploy and manage `Prometheus`, `Alertmanager`, and related components.
  * **Simplified Deployment Configuration:** Configure the fundamentals of Prometheus like versions, persistence, retention policies, and replicas from a native Kubernetes resource.
  * **Target Services via Labels:** Automatically generate monitoring target configurations based on familiar Kubernetes label queries; no need to learn a Prometheus specific configuration language.

**Objective:**

  * Explore the operators' deployment setup
  * Configure `ServiceMonitor`, which declaratively specifies how groups of Kubernetes services should be monitored. The Operator automatically generates Prometheus scrape configuration based on the current state of the objects in the API server.
  * Configure `PrometheusRule` and `AlertmanagerConfig` to be able to send Slack alerts


### 1.1 Create Regional GKE Cluster on GCP

**Step 1** Enable the Google Kubernetes Engine API.
```
gcloud services enable container.googleapis.com
```

**Step 2** From the cloud shell, run the following command to create a cluster with 1 node:

```
gcloud container clusters create k8s-prometheus-labs \
--region us-central1 \
--enable-ip-alias \
--enable-network-policy \
--num-nodes 1 \
--machine-type "e2-standard-4" \
--release-channel regular
```

```
gcloud container clusters get-credentials k8s-prometheus-labs --region us-central1
```


**Step 3: (Optional)** Setup `kubectx` and `kubens`

!!! info
    `kubectx` + `kubens`: Power tools for `kubectl`:
    - `kubectx` helps you switch between clusters back and forth
    - `kubens` helps you switch between Kubernetes namespaces smoothly

Install As [kubectl plugins(macOS & Linux)](https://github.com/ahmetb/kubectx#kubectl-plugins-macos-and-linux)

```
kubectl krew install ctx
kubectl krew install ns
```

!!! note
    Google Cloud Console comes with `kubectx` and `kubens` setup.
    However above steps can be used to setup your local shell.



### 1.2 Install Prometheus Operator and Grafana Helm Charts

#### 1.2.1 Install `kube-prometheus-stack` helm chart

**kube-prometheus**
[kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) provides example configurations for a complete cluster monitoring
stack based on Prometheus and the Prometheus Operator.  This includes deployment of multiple Prometheus and Alertmanager instances,
metrics exporters such as the node_exporter for gathering node metrics, scrape target configuration linking Prometheus to various
metrics endpoints, and example alerting rules for notification of potential issues in the cluster.

**helm chart**
The [prometheus-community/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
helm chart provides a similar feature set to kube-prometheus. For more information, please see the [chart's readme](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#kube-prometheus-stack)



**Step 1** Configure `Helm` repository


```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```


**Step 2** Fetch  `Helm` repository to local filesystem

```
mkdir ~/gmp-setup
cd ~/gmp-setup
```

```
helm pull prometheus-community/kube-prometheus-stack --version 35.6.0
tar -xvzf kube-prometheus-stack-35.6.0.tgz
cd kube-prometheus-stack
tree -L 2
```

**Output:**

```
── charts
│   ├── grafana
│   ├── kube-state-metrics
│   └── prometheus-node-exporter
├── crds
│   ├── crd-alertmanagerconfigs.yaml
│   ├── crd-alertmanagers.yaml
│   ├── crd-podmonitors.yaml
│   ├── crd-probes.yaml
│   ├── crd-prometheuses.yaml
│   ├── crd-prometheusrules.yaml
│   ├── crd-servicemonitors.yaml
│   └── crd-thanosrulers.yaml
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── alertmanager
│   ├── exporters
│   │   ├── core-dns
│   │   ├── kube-api-server
│   │   ├── kube-controller-manager
│   │   ├── kube-dns
│   │   ├── kube-etcd
│   │   ├── kube-proxy
│   │   ├── kube-scheduler
│   │   ├── kube-state-metrics
│   │   ├── kubelet
│   │   └── node-exporter
│   ├── grafana
│   ├── prometheus
│   └── prometheus-operator
└── values.yaml
```

!!! summary 
    This chart is maintained by the Prometheus community and contains everything you need to get started including:

    * `prometheus operator`
    * `alertmanager`
    * `grafana` with predefined dashboards
    * `prometheus-node-exporter` - Prometheus exporter for hardware and OS metrics exposed by *NIX kernels, written in Go with pluggable metric collectors
    *  Kubernetes Control and Data plane  `exporters` such as `kube-api-server`, `core-dns`, `kube-controller-manager`, `kube-etcd`, `kube-scheduler`, `kubelet`
    * `kube-state-metrics` - is a simple service that listens to the Kubernetes API server and generates metrics about the state of the objects



**Step 3** Create custom Grafana configuration, that will allow to expose Grafana Dashboard:

Option 1 Ingress
```
cat << EOF>> grafana_values.yaml
grafana:
  adminPassword: admin
  ingress:
    enabled: true
    path: /*
    pathType: ImplementationSpecific
  service:
    type: NodePort
EOF
```

Option 1 LoadBalancer

```
cat << EOF>> grafana_values.yaml
grafana:
  adminPassword: admin
  service:
    type: LoadBalancer
EOF
```

**Step 4** Install Helm Chart to `monitoring` namespace


```
kubectl create ns monitoring
kubens monitoring
helm install prometheus-stack prometheus-community/kube-prometheus-stack --version 35.6.0 --values grafana_values.yaml
```


#### 1.2.2 Observe Grafana Dashboards


**Step 1:** Locate Grafana Dashboard URL:

```
kubectl get svc prometheus-stack-grafana
kubectl get ing
```

**Step 2:** Launch the Grafana Dashboard and see Predefined Dashboards:

```
loadbalancer_ip
```


We've setup `admin` user password as: `admin`. So we will use this values to login to Grafana dashboard:

```
admin/admin
```

!!! result
    You should see your Grafana interface


**Step 2** Access Grafana Dashboard and see Predefined Dashboards:

![alt text](images/grafana_dashboard.png)

**Step 3** Observe following Dashboards:


```
General /Kubernetes / Compute Resources / Cluster
```

!!! summary
    Observe Usage, Totals, Quotas, Requests for Memory and CPU per namespace
    This values could be good input for Pods `requests` and `limits` measurements


```
General /Kubernetes / Compute Resources / Nodes (Pods)
```

!!! summary
    Observe CPU and Memory per Node

```
General /Kubernetes / Compute Resources / Namespace (Pods)
```

!!! summary
    Observe CPU and Memory per Pods


Observe Dashboards for Control Plane monitoring:

```
General / Kubernetes / API server
General / Kubernetes / Kubelet
General / Kubernetes / Scheduler
General / Kubernetes / Controller Manager
General / etcd
```

!!! note
    some of the Dashboards (etcd, Scheduler, Controller Manager) are empty, this is because GKE is Managed Kubernetes so some metrics are not available for scraping.



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

**Step 4** Observe following Dashboards:


```
General /Kubernetes / Compute Resources / Namespace (Pods)
General /Kubernetes / Compute Resources / Namespace (Workloads)

```

Choose:
  * Relative time ranges: `Last 15 minutes`
  * Namespace: `onlineboutique`

!!! summary
    Observe CPU and Memory per Pods


#### 1.2.4 Deploy application and scrape custom metrics with `PodMonitor`

Deploy application `prom-example`:

**Step 1** Create Namespace `prom-test`

```
kubectl create ns prom-test
kubens prom-test
```

**Step 2** Create deployment `prom-example`
```
cat << EOF>> prom-example.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prom-example
  labels:
    app: prom-example
spec:
  selector:
    matchLabels:
      app: prom-example
  replicas: 3
  template:
    metadata:
      labels:
        app: prom-example
    spec:
      containers:
      - image: nilebox/prometheus-example-app@sha256:dab60d038c5d6915af5bcbe5f0279a22b95a8c8be254153e22d7cd81b21b84c5
        name: prom-example
        ports:
        - name: metrics
          containerPort: 1234
        command:
        - "/main"
        - "--process-metrics"
        - "--go-metrics"
EOF
```

**Step 4** Deploy `prom-example` application

```
kubectl apply -f prom-example.yaml
```

```
kubectl get pods
```

**Step 5** List the Prometheus Operator Custom Resources Definitions (CRD's)

```
kubectl get crd | grep monitoring
```

**Step 6** Scrape Pod Metrics using `PodMonitor` crd

The operator uses `ServiceMonitors` or `PodMonitor` to define a set of targets to be monitored by Prometheus. It uses label selectors to define which `Services` or `Pod` to monitor, the namespaces to look for, and the port on which the metrics are exposed.


First let's find out how `kind: Prometheus` will select `PodMonitor` using `serviceMonitorSelector` ?

```
kubens monitoring
kubectl get prometheuses.monitoring.coreos.com prometheus-stack-kube-prom-prometheus -o yaml | grep -C5 podMonitorSelector
```


**Output:** 

```
  podMonitorSelector:
    matchLabels:
      release: prometheus-stack
```

!!! result
    Prometheus Operator will select all `podMonitor` that match match label: `release: prometheus-stack`



Create a file `pod-monitor.yaml` with the following content to add a `PodMonitor` so that the Prometheus server scrapes only its own metrics endpoints:


Define a PodMonitor in a manifest file `podmonitor.yaml` by selecting `app: prom-example` labels in namespace `prom-test` and scrape metrics from `port: metrics`. Define `PodMonitor` labels as `release: prometheus-stack`, so that Prometheus operator can select them. 

```
cat << EOF>> podmonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: prom-example
  labels:
    release: prometheus-stack
spec:
  namespaceSelector:
    matchNames:
      - prom-test
  selector:
    matchLabels:
      app: prom-example
  podMetricsEndpoints:
  - port: metrics
EOF
```

```
kubectl apply -f podmonitor.yaml
```


**Step 3** Observe that `PodMonitor` been picked up by Prometheus Operator


```
kubectl -n monitoring get secret prometheus-prometheus-stack-kube-prom-prometheus -ojson | jq -r '.data["prometheus.yaml.gz"]' | base64 -d | gunzip | grep "prom-example"
```

#### 1.2.5 Deploy application and scrape custom metrics with `ServiceMonitor`

**Step 1** Deploy application `example-app` in `default` namespace:


```
cat << EOF>> example-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
      - name: example-app
        image: fabxc/instrumented_app
        ports:
        - name: web
          containerPort: 8080
EOF
```

```
cat << EOF>> example-svc.yaml
kind: Service
apiVersion: v1
metadata:
  name: example-app
  labels:
    app: example-app
spec:
  selector:
    app: example-app
  ports:
  - name: web
    port: 8080
EOF
```


```
kubens default
kubectl apply -f example-app.yaml
kubectl apply -f example-svc.yaml
```


**Step 2** Deploy `ServiceMonitor` in `monitoring` namespace


First let's find out how `kind: Prometheus` will select `PodMonitor` using `serviceMonitorSelector` ?

```
kubens monitoring
kubectl get prometheuses.monitoring.coreos.com prometheus-stack-kube-prom-prometheus -o yaml | grep -C5 serviceMonitorSelector
```


**Output:** 

```
  podMonitorSelector:
    matchLabels:
      release: prometheus-stack
```

!!! result
    Prometheus Operator will select all `podMonitor` that match match label: `release: prometheus-stack`

```
cat << EOF>> example-svc-mon.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  labels:
    release: prometheus-stack
spec:
  namespaceSelector:
    matchNames:
      - default
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: web
EOF
```

```
kubens monitoring
kubectl apply -f example-svc-mon.yaml
```


Verify `servicemonitor` has been created in `monitoring` namespace

```
kubectl get -A servicemonitor.monitoring.coreos.com
```

!!! note
    Creation of `servicemonitor` can be done in application namespace as well


**Step 3** Observe that ServiceMonitor been picked up by Prometheus Operator

```
kubectl -n monitoring get secret prometheus-prometheus-stack-kube-prom-prometheus -ojson | jq -r '.data["prometheus.yaml.gz"]' | base64 -d | gunzip | grep "example-app"
```


**Step 4** Observe Prometheus UI and check that `servicemonitor` has been discovered under `Targets` and `ServiceDiscovery`:

```
kubectl port-forward svc/prometheus-stack-kube-prom-prometheus  8080:9090
```


**Step 5** **Step 4** Observe Grafana UI and try to build a dashboards using on the exposed Prometheus metrics emitted by `example-app` app:

The following metrics are exposed:

- `version` - of type _gauge_ - containing the app version - as a constant metric value `1` and label `version`, representing this app version
- `http_requests_total` - of type _counter_ - representing the total numbere of incoming HTTP requests

The sample output of the `/metric` endpoint after 5 incoming HTTP requests shown below.

Note: with no initial incoming request, only `version` metric is reported.

```
# HELP http_requests_total Count of all HTTP requests
# TYPE http_requests_total counter
http_requests_total{code="200",method="get"} 5
# HELP version Version information about this binary
# TYPE version gauge
version{version="v0.3.0"} 1
```


## 2 Migrate from Prometheus Operator to Google Managed Prometheus with self-deployed collection 

### 2.0 Verify SA account has correct permissions. 


**Option 1:** Using GKE with `default` Compute Service Account, should have all required permissions as it is using Editor role (roles/editor) on the project. (without WI enabled)

**Option 2:** Using GKE with custom Service Account for Nodes (without WI enabled)

Make sure default compute `SA` has `monitoring.metricWriter` and `monitoring.viewer` roles:

```
gcloud projects get-iam-policy $PROJECT_ID
```

If you have `Editor` role or `roles/monitoring.metricWriter` and `role: roles/monitoring.viewer` skip the step.
Otherwise add following roles tp SA.


```
gcloud projects add-iam-policy-binding $PROJECT_ID \
 --member='serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com' \
 --role=roles/monitoring.metricWriter

gcloud projects add-iam-policy-binding $PROJECT_ID \
 --member='serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com' \
 --role=roles/monitoring.viewer
```

**Option 3:** If using GKE with WI, follow the [Configure a service account for Workload Identity](https://cloud.google.com/stackdriver/docs/managed-prometheus/setup-unmanaged#gmp-wli-svcacct) instructions

**Option 4:** If using Non-GKE Kubernetes clusters, follow the [Provide credentials explicitly](https://cloud.google.com/stackdriver/docs/managed-prometheus/setup-unmanaged#explicit-credentials) instructions



### 2.1 Setup GMP Collection


While Prometheus Operator, provides us great APIs to declaratively configure Prometheus tasks such as scrape metrics and configure alerts and etc.  Prometheus Operator itself doesn't solve problem of central monitoring system as it's deployed per K8s Cluster or per-namespace, and usually Organizations have many  K8s Clusters and they looking for a central pain of glass view.
Another challenge that Prometheus Operator  doesn't solve is long term storage.

To address above challenges we can use GMP with self-deployed collection. This way once can still use existing `Prometheus Operator` configuration using CRDs and Declarative APIs and take advantage of Google Managed Prometheus Datastore - based on - Monarch, that can handle incredible scale and store 2 years of metrics at no cost.

See Reference doc - [Get started with self-deployed collection](https://cloud.google.com/stackdriver/docs/managed-prometheus/setup-unmanaged)


**Step 1**  Create `prometheus-stack` Helm env var file, that will update from prometheus operator binary to GMP binary:

```
cat << EOF>> update_to_gmp.yaml
prometheus:
  prometheusSpec:
    image:
      repository: gke.gcr.io/prometheus-engine/prometheus
      tag: v2.35.0-gmp.2-gke.0
grafana:
  adminPassword: admin
  ingress:
    enabled: true
    path: /*
    pathType: ImplementationSpecific
  service:
    type: NodePort
EOF
```

**Step 1**  Update `prometheus-stack` Helm chart with GMP binary (image):

```
kubectl get -A prometheuses.monitoring.coreos.com
```

**Output:** 

```
          image: "quay.io/prometheus-operator/prometheus-operator:v0.56.3"
```

**Step 2** Let's replace the `prometheus-operator` image with `gmp` image with Helm:

```
helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack --version 35.6.0 --values update_to_gmp.yaml
```


**Step 3** Verify that Prometheus Operator is configured with GMP binary:

```
kubectl get -A prometheuses.monitoring.coreos.com
```

**Output:** 

```
monitoring   prometheus-stack-kube-prom-prometheus   v2.35.0-gmp.2-gke.0   1      50m
```



### 2.2 Verify GMP Collection

See Reference doc - [Managed Service for Prometheus page](https://cloud.google.com/stackdriver/docs/managed-prometheus/query#promui-deploy)


**Step 1** Verify that Prometheus data is being exported and stored at Managed Service for Prometheus Datastore:

The simplest way to verify that your Prometheus data is being exported is to use the PromQL-based Managed Service for Prometheus page in the Google Cloud console.

To view this page, do the following:

1. In the Google Cloud console, go to Monitoring or use the following button:

2. In the Monitoring navigation pane, click Managed Prometheus.

On the Managed Service for Prometheus page, you can use PromQL queries to retrieve and chart data collected with the managed service. This page can query only data collected by Managed Service for Prometheus.

The following screenshot shows a chart that displays the `up` metric


To deploy the Prometheus UI for Managed Service for Prometheus, run the following commands:

Deploy the frontend service and configure it to query the scoping project of your metrics scope of your choice:


### 2.3 Setup Prometheus UI for GMP


To deploy the Prometheus UI for Managed Service for Prometheus, run the following commands.

**Step 1**  Deploy the `Prometheus UI` frontend service inside of namespace where Grafana setup is running. 

!!! note 
    If you want to continue using a `Grafana` deployment installed by kube-prometheus, then deploy the Prometheus UI in the `monitoring` namespace instead. However for production use cases Grafana typically deployed with `Grafana` Helm Chart or `Grafana` Operator Chart, allowing operators deploy their customer Dashboards.


```
export GRAFANA_NAMESPACE=monitoring
```

```
curl https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.4.1/examples/frontend.yaml |
sed 's/\$PROJECT_ID/gcp-demos-331123/' |
kubectl apply -n $GRAFANA_NAMESPACE -f -
```

**Step 2**  Port-forward the `frontend` service to your local machine. The following example forwards the service to port 9090:

```
kubectl -n $GRAFANA_NAMESPACE port-forward svc/frontend 8080:9090
```

### 2.4 Setup Grafana for GMP


Google Cloud APIs all require authentication using OAuth2; however, Grafana doesn't support OAuth2 authentication for Prometheus data sources. To use Grafana with Managed Service for Prometheus, we've deployed Prometheus UI as an authentication proxy in a previous step.

**Option 1:** Configure existing Grafana Dashboard to use GMP

If you would like to take advantage of you existing Dashboards and already deployed Grafana,
the only steps needed is to create a `Data Source` for Google Managed Prometheus described in this [reference](https://cloud.google.com/stackdriver/docs/managed-prometheus/query#ui-grafana)


Once configured you can redirect existing dashboards to the new `Data Source`, or create the new once.


**Option 2:** Configure Grafana Dashboard that comes with GMP 

 Configure Grafana Dashboard that comes with GMP, following this [steps](https://cloud.google.com/stackdriver/docs/managed-prometheus/query#grafana-deploy)

!!! note
    This option is good only for quick testing Grafana

**Option 3:** Use Grafana Helm Charts or Grafana Operator

For production deployment it is preferred to use  [Grafana Helm Chart](https://artifacthub.io/packages/helm/grafana/grafana) or [Grafana Operator](https://artifacthub.io/packages/olm/community-operators/grafana-operator) as you can define and recreate you dashboards via json or CRDs in consistent manner.


### 2.5 Verify GMP Cost

**Option 1** Estimate Cost based on ingested metrics.

*Step 1* Create a new Grafana Dashboard:

```
Data Source: Managed Service for Prometheus

Metric Browser: `rate(gcm_export_samples_exported_total{job=~".+",instance=~".+"}[5m])`

Step: 10
```

![alt text](images/gmp_cost_calculation.png)


Example calculation:

N1. 770 Samples per second based on Dashboard value

N2. 770 samples per second * 2628000 seconds/mo = 2023 B samples/mo

N3. Since $0.15/million samples: first 0-50 billion samples#  [Reference N1](https://cloud.google.com/stackdriver/pricing#monitoring-costs)  

N4. We can calculate final cost: 2023 B samples/mo * 0.15 = ~303$ per month 





**Option 2** View your Google Cloud bill

1. In the Google Cloud console, go to the Billing page.

2. If you have more than one billing account, select Go to linked billing account to view the current project's billing account. To locate a different billing account, select Manage billing accounts and choose the account for which you'd like to get usage reports.

3. Select Reports.

4. From the Services menu, select the Stackdriver Monitoring option.

5. From the SKUs menu, select the following options:

  * Prometheus Samples Ingested
  * Monitoring API Requests


## 7 Cleanup

Delete Kubernetes cluster as it will enquire cost both for GKE and GMP.

Uninstall Helm Charts:

```
helm uninstall prometheus-stack -n monitoring
```

```
kubectl delete crd alertmanagerconfigs.monitoring.coreos.com
kubectl delete crd alertmanagers.monitoring.coreos.com
kubectl delete crd podmonitors.monitoring.coreos.com
kubectl delete crd probes.monitoring.coreos.com
kubectl delete crd prometheuses.monitoring.coreos.com
kubectl delete crd prometheusrules.monitoring.coreos.com
kubectl delete crd servicemonitors.monitoring.coreos.com
kubectl delete crd thanosrulers.monitoring.coreos.com
```

Delete GKE cluster:

```
gcloud container clusters delete k8s-prometheus-labs --region us-central1
```