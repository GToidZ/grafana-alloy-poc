# Kubernetes Cluster Usage Report

## Steps to Reproduce
0. Have the following tools setup and running in the cluster:
   - Prometheus (presumably, kube-prometheus-stack)
   - Grafana
1. Create and apply a new `ConfigMap` that contains the content for `alloy.config`,
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: grafana-alloy-config
     namespace: kube-prometheus-stack
   data:
     alloy.config: |
       // Use double-slash to denote the line as comment for "alloy.config"
       
       // Specify Prometheus remote-write endpoint
       // For this development case, I enabled the remote write endpoint on Prometheus server
       prometheus.remote_write "default" {
         endpoint {
           url = "http://kube-prometheus-stack-prometheus.kube-prometheus-stack:9090/api/v1/write"
         }
       }

       // Specify an exporter, in this case, unix is the same as node-exporter
       // ref: https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.exporter.unix/
       prometheus.exporter.unix "node" {}

       // Relabel job label on the metrics before exporting
       discovery.relabel "job_relabel" {
         targets = prometheus.exporter.unix.node.targets

         rule {
           target_label = "job"
           replacement = "alloy_node_exporter"
         }
       }

       // Specify a scraping job here
       prometheus.scrape "cluster" {
         targets    = discovery.relabel.job_relabel.output
         forward_to = [prometheus.remote_write.default.receiver]
       }
   ```
2. Install Grafana Alloy via Helm, per [official instructions](https://grafana.com/docs/alloy/latest/set-up/install/kubernetes/).
   I also modified the chart using `values.yaml` so that it should be using an existing ConfigMap,
   ```yaml
   alloy:
     configMap:
       create: false
       name: grafana-alloy-config
       key: alloy.config
     enableReporting: false
   ```
3. Port-forward or enable access to the Grafana service on the cluster,
   ```sh
   kubectl port-forward -n kube-prometheus-stack svc/kube-prometheus-stack-grafana 3001:80
   ```
4. Access Grafana UI in browser, then create a new dashboard using [Node Exporter Full](https://grafana.com/grafana/dashboards/1860-node-exporter-full/) template.
5. Access the dashboard then filter for the `job='alloy_node_exporter'`

## Results

The following image shows a Grafana dashboard containing data with metrics from `node-exporter` sent by Grafana Alloy, as evident by the job selected.
![Dashboard Image](/01-usage/cluster/grafana-dashboard-node-exporter-on-cluster.png)