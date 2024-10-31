# Grafana Alloy External Modules

This PoC shows how Grafana Alloy can import modules (custom components) from external sources. This assumes that you have already read how Alloy is set up in [here](/01-usage/cluster/).

The PoC contains files being used / applied to a Kubernetes cluster:

## `alloy-config.yaml`
The file contains a declarative configuration for Grafana Alloy.
The configuration reference can be found [here](https://grafana.com/docs/alloy/latest/get-started/configuration-syntax/).

For the config in this PoC,
* The first block imports an external module from a private Git repository `GToidZ/grafana-alloy-poc-modules`.
  The `import.git` block can have two different authentication schemes: `basic_auth` or `ssh_key`.

  * `basic_auth` requires you to provide two things, a `username` and a `password`.
  * `ssh_key` requires two things, a `username` and a `key`; may also require `passphrase`.

  Read further about `import.git` block [here](https://grafana.com/docs/alloy/latest/reference/config-blocks/import.git/).

  The block also utilizes [`sys.env`](https://grafana.com/docs/alloy/latest/reference/stdlib/sys/#sysenv) function, as it will delegate environment variables to substitute in the config.

* The second block is for specifying a Promethues remote-write endpoint. See [here](https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.remote_write/).

* The last block refers to a custom function from an imported module. Refer to how it works below.

## `modules/node_exporter.alloy`
This is an example of external module for this PoC, it is hosted on GitHub in a private repository. However, the file contents are as followed:
```alloy
declare "create" {
    argument "forward_to" {
        optional = false
        comment = "Targets to forward the metrics to, takes list as input."
    }
    argument "job_name" {
        optional = true
        default = "node_exporter"
        comment = "Job name to relabel, defaults to `node_exporter`."
    }

    prometheus.exporter.unix "node_exporter_host" {}

    discovery.relabel "node_exporter_job_relabel" {
        targets = prometheus.exporter.unix.node_exporter_host.targets
        rule {
            target_label = "job"
            replacement = argument.job_name.value
        }
    }

    prometheus.scrape "node_exporter_scrape" {
        targets    = discovery.relabel.node_exporter_job_relabel.output
        forward_to = argument.forward_to.value
    }
}
```
The file declares a new component called `create` which can be called in other configuration using `imported_name.create {}`.

Within this component, it is able to receive two arguments, `forward_to` (required) and `job_name`.
The comments in both arguments are used to tell the administrator/operator what the arguments do.

The component instantiates three more components, [`prometheus.exporter.unix`](https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.exporter.unix/), [`discovery.relabel`](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.relabel/) and [`prometheus.scrape`](https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.scrape/).

## `known-hosts-config.yaml`
This file contains a `ConfigMap` containing data of a file `ssh_known_hosts`, which is used by our Grafana Alloy pods to be able to authenticate to GitHub's `ssh` server without the need to verify the host.

The ConfigMap is used by being mounted as a volume at `/etc/ssh/ssh_known_hosts` which is the default location for global known hosts.

## `secrets.yaml` (not uploaded)
This file contains a `Secret` containing data of two following keys:
* `ssh-privatekey`: raw content of an `ssh` private key file needed for authenticating with Git servers. This is mounted as a volume in the Grafana Alloy pods.
* `passphrase`: passphrase needed for unlocking the `ssh` private key.

## `values.yaml`
This file is a configuration for a Grafana Alloy `helm` chart.
It disables the creation of a new config map, and instead uses the existing one from `alloy-config.yaml`.
It also creates volumes and mount points from other `ConfigMap` and `Secret` present in this PoC.