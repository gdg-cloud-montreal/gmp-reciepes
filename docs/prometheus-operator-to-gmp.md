# Prometheus-operator scraping metrics to GMP


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


## 0 Create Regional GKE Cluster on GCP

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
--release-channel rapid
```

```
gcloud container clusters get-credentials k8s-prometheus-labs --region us-central1
```


**Step 3: (Optional)** Setup kubectx

```
sudo apt install kubectx
```

!!! note
    we've installed `kubectx` + `kubens`: Power tools for `kubectl`:
    - `kubectx` helps you switch between clusters back and forth
    - `kubens` helps you switch between Kubernetes namespaces smoothly

## 1 Install Prometheus Operator and Grafana Helm Charts

### 1.1 Install `kube-prometheus-stack` helm chart

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
cd ~/$MY_REPO/notepad-infrastructure/helm
helm pull prometheus-community/kube-prometheus-stack
tar -xvzf kube-prometheus-stack-36.6.1.tgz
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
helm install prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --values grafana_values.yaml
kubens monitoring
```


## 4 Observe Grafana Dashboards


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



## 3 Deploy onlineboutique application and observe app compute resources

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


## 4 Deploy application and scrape custom metrics

Deploy application `prom-example`:

**Step 1** Create Namespace `prom-test`

```
kubectl create ns prom-test
kubens prom-test
```

**Step 2** Create deployment `prom-example`
```
cat << EOF>> example-app.yaml
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
kubectl apply -f example-app.yaml
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

Create a file pod-monitor.yaml with the following content to add a `PodMonitor` so that the Prometheus server scrapes only its own metrics endpoints:

Define a PodMonitor in a manifest file `podmonitor.yaml` to select only this deployment pod 



```
cat << EOF>> podmonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: example-app
  labels:
    team: frontend
spec:
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

```
cat << EOF>> prometheus-service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
EOF
```

```
cat << EOF>> prometheus-cluster-role-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: default
EOF
```

```
cat << EOF>> prometheus-cluster-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/metrics
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
EOF
```


```
kubectl apply -f prometheus-service-account.yaml
kubectl apply -f prometheus-cluster-role.yaml
kubectl apply -f prometheus-cluster-role-binding.yaml
```


```
cat << EOF>> prom_resource.yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  podMonitorSelector:
    matchLabels:
      team: frontend
  resources:
    requests:
      memory: 400Mi
  enableAdminAPI: false
EOF
```


```
kubectl apply -f prom_resource.yaml
```





## 7 Cleanup 

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