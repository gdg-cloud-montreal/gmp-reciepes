apiVersion: monitoring.coreos.com/v1alpha1
kind: ServiceMonitor
metadata:
  name: fluentd
  namespace: logging
  labels:
    k8s-app: fluentd
spec:
  jobLabel: app
  selector:
    matchExpressions:
    - {key: k8s-app, operator: In, values: [fluentd, fluentd-aggregator]}
  namespaceSelector:
    any: false
    matchNames:
    - logging
  endpoints:
  - port: prometheus-metrics
    interval: 15s