# Host Machine Usage Report

## Steps to Reproduce
0. Have the following tools setup and running in the machine:
   - Prometheus
   - Grafana
1. Install Grafana Alloy on host machine, (in this case, the host is Windows so we'll follow these [instructions](https://grafana.com/docs/alloy/latest/set-up/install/windows/))
2. Create a file `alloy.config` in other directory than the default,
   ```
   // Use double-slash to denote the line as comment for "alloy.config"

   // Specify Prometheus remote-write endpoint
   // For this development case, I enabled the remote write endpoint on Prometheus server
   prometheus.remote_write "default" {
     endpoint {
       url = "http://localhost:9090/api/v1/write"
     }
   }

   // Specify an exporter, in this case, host machine is Windows, so we're using "windows_exporter"
   // ref: https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.exporter.windows/
   prometheus.exporter.windows "node" {}

   // Relabel job label on the metrics before exporting
   discovery.relabel "job_relabel" {
     targets = prometheus.exporter.windows.node.targets

     rule {
       target_label = "job"
       replacement = "alloy_windows_exporter"
     }
   }

   // Specify a scraping job here
   prometheus.scrape "cluster" {
     targets    = discovery.relabel.job_relabel.output
     forward_to = [prometheus.remote_write.default.receiver]
   }
   ```
3. Modify the Grafana Alloy service to use the new config file. Change run arguments etc.
   
   My ref: https://grafana.com/docs/alloy/latest/configure/windows/#change-command-line-arguments

   Then, restart the service.
4. Access Grafana UI in browser, then create a new dashboard using [Windows Exporter Dashboard 2024](https://grafana.com/grafana/dashboards/20763-windows-exporter-dashboard-2024/) template.
5. Access the dashboard then filter for the `job='alloy_windows_exporter'`

## Results

The following image shows a Grafana dashboard containing data with metrics from `windows-exporter` sent by Grafana Alloy, as evident by the job selected.
![Dashboard Image](/01-usage/host/grafana-dashboard-windows-exporter-on-host.png)