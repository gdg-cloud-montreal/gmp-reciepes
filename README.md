# Google Cloud Managed Service for Prometheus Recipes

This repository contains various use cases (aka Recipes) and examples of Google Cloud Managed Service for Prometheus(GMP). For each of the use-cases there are examples that show how these GMP capabilities should be used.

Each recipes is a self-contained example. With a full tutorial for how to set it up and tear it down.

If you're not familiar with the basics of Prometheus then check out [How to Setup Prometheus Monitoring On Kubernetes Cluster](https://devopscube.com/setup-prometheus-monitoring-on-kubernetes/). These resources should give you some of the foundations behind Prometheus Architecture and how set it up on Kubernetes.

To learn more about Google Cloud Managed Service for Prometheus checkout the [GMP official documentation](https://cloud.google.com/stackdriver/docs/managed-prometheus)

## Recipes

- Self-deployed collection
  - [Prometheus-operator scraping metrics to GMP](./docs/prometheus-operator-to-gmp.md) - Deploy Prometheus-operator on GKE cluster and replace it with GMP drop-in binary to enable HA and longterm storage capabilities.
  
- Managed collection with GKE Standard
  - [GMP setup with single GKE Cluster to scrape Custom App metrics](./docs/gmp-with-gkestandard-custom.md) - Deploy GMP to GKE cluster and scrape custom App metrics.
  - [GMP setup with single GKE Cluster to scrape Fluent-bit metrics](./docs/gmp-with-gkestandard-flientbit.md) - Deploy GMP to scrape metrics from Fluent-bit and observe dashboards in Grafana.
  - [GMP setup with single GKE Cluster to scrape Nginx-Ingress metrics](./docs/gmp-with-gkestandard-nginxingress.md) - Deploy GMP to scrape metrics from Nginx-Ingress Controller and observe dashboards in Grafana.
  - [Setup GMP and query Google Cloud Metrics with PromQL and Grafana](./docs/gmp-for-gcp-cloud-resources.md) - Monitor GCP Cloud Resources with GMP and Grafana.
  - [Centralized Multi-tenant GMP setup with GKE Clusters in Different Projects](./docs/gmp-multi-tenant.md) - WIP
  
- Setting up GMP with Terraform
  - [GMP setup with Terraform](./docs/gmp-with-terraform.md) - WIP

- Setting up GMP with KCC
  - [GMP setup with KCC](./docs/gmp-with-kcc.md) - WIP

- Managed collection with GKE Autopilot
  - [GMP setup with single GKE Autopilot](./docs/gmp-with-gkeautopilot.md)- WIP

- Managed Prometheus AlertManager