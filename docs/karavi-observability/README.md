<!--
Copyright (c) 2020 Dell Inc., or its subsidiaries. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0
-->

# Karavi Observability

Karavi Observability is part of the Karavi open source suite of Kubernetes storage enablers for Dell EMC products. Karavi Observability provides standardized approaches to gaining observability into Dell EMC products.  Below is a list of supported Dell EMC storage products:

| Dell EMC Storage Product   |
| --------- |
| PowerFlex v3.0/3.5 |

Karavi Observability can be decomposed into the following services:

| Karavi Observability Service | Description | Repository |
| --------- | --------- | --------- |
| Karavi Metrics for PowerFlex | Karavi PowerFlex Metrics captures telemetry data about Kubernetes storage usage and performance and pushes it to the OpenTelemetry Collector, so it can be processed, and exported in a format consumable by Prometheus. Prometheus can then be configured to scrape the OpenTelemetry Collector exporter endpoint to provide metrics so they can be visualized in Grafana. Please visit the repository for more information. | [Karavi Metrics for PowerFlex](https://github.com/dell/karavi-metrics-powerflex) |
| Karavi Topology | Karavi Topology provides visibility into Dell EMC CSI (Container Storage Interface) driver provisioned volume characteristics in Kubernetes correlated with volumes on the storage system.. Please visit the repository for more information. | [Karavi Topology](https://github.com/dell/karavi-topology) |

Each of the services listed about can be deployed independently by following the documentation provided in the associated repositories.  Alternatively, the services can all be deployed together as part of a single deployment of the Karavi Observability solution.

This document steps through the single deployment and configuration of Karavi Observability.

## Kubernetes

First and foremost, Karavi Observability requires a Kubernetes cluster that aligns with the supported versions listed below.

| Version   |
| --------- |
| 1.17-1.19 |

## Deploying Karavi Observability

This project is deployed using Helm. Usage information and available release versions can be found here: [Karavi Observability](https://github.com/dell/helm-charts/tree/main/charts/karavi-observability).

**Note**: Karavi Observability must be deployed first.  Once Karavi Observability has been deployed, you can proceed to deploying/configuring the required components below.

## Required Components

This application requires the following third party components to be deployed in the same Kubernetes cluster as the karavi-metrics-powerflex service:

* Prometheus
* Grafana

It is the user's responsibility to deploy these components in the same Kubernetes cluster as the karavi-metrics-powerflex service.  These components must be deployed according to the specifications defined below.

### Prometheus

As part of the Karavi Observability deployment, the OpenTelemetry Collector gets deployed.  The OpenTelemetry Collector is what the Karavi metrics services use to push metrics to that can be consumed by Prometheus.  This means that Prometheus must be configured to scrape the metrics data from the OpenTelemetry Collector.

| Supported Version | Image                   | Helm Chart                                                   |
| ----------------- | ----------------------- | ------------------------------------------------------------ |
| 2.22.0           | prom/prometheus:v2.22.0 | [Prometheus Helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus) |

**Note**: It is the user's responsibility to provide persistent storage for Prometheus if they want to preserve historical data.

Here is a sample minimal configuration for Prometheus. Please note that the configuration below uses insecure skip verify. If you wish to properly configure TLS, you will need to provide a ca_file in the Prometheus configuration. The certificate provided as part of Karavi Observability deployment should be signed by this same CA. For more information about Prometheus configuration, see [Prometheus configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#configuration).

```yaml
scrape_configs:
    - job_name: 'karavi-metrics-powerflex'
      scrape_interval: 5s
      scheme: https
      static_configs:
        - targets: ['otel-collector:443']
      tls_config:
        insecure_skip_verify: true
```

### Grafana

The Grafana dashboards require Grafana to be deployed in the same Kubernetes cluster as the Karavi Observability services.  Below are the configuration details required to properly setup Grafana to work with Karavi Observability.

| Supported Version | Helm Chart                                                |
| ----------------- | --------------------------------------------------------- |
| 7.3.0-7.3.2       | [Grafana Helm chart](https://github.com/grafana/helm-charts/tree/main/charts/grafana) |

Grafana must be configured with the following data sources/plugins:

| Name                   | Additional Information                                                     |
| ---------------------- | -------------------------------------------------------------------------- |
| Prometheus data source | [Prometheus data source](https://grafana.com/docs/grafana/latest/features/datasources/prometheus/)   |
| Data Table plugin      | [Data Table plugin](https://grafana.com/grafana/plugins/briangann-datatable-panel/installation) |
| Pie Chart plugin       | [Pie Chart plugin](https://grafana.com/grafana/plugins/grafana-piechart-panel)                 |
| JSON data source       | [JSON data source](https://grafana.com/grafana/plugins/simpod-json-datasource)                 |

Settings for the Grafana Prometheus data source:

| Setting | Value                     | Additional Information                          |
| ------- | ------------------------- | ----------------------------------------------- |
| Name    | Prometheus                |                                                 |
| Type    | prometheus                |                                                 |
| URL     | http://PROMETHEUS_IP:PORT | The IP/PORT of your running Prometheus instance |
| Access  | Proxy                     |                                                 |

Settings for the Grafana JSON data source:

| Setting             | Value                             |
| ------------------- | --------------------------------- |
| Name                | JSON |
| URL                 | Access Karavi Topology at https://karavi-topology.*namespace*.svc.cluster.local |
| Skip TLS Verify     | Enabled (If not using CA certificate) |
| With CA Cert        | Enabled (If using CA certificate) |

#### Importing the Karavi Observability Dashboards

Once Grafana is properly configured, you can import the pre-built observability dashboards. Log into Grafana and click the + icon in the side menu. Then click Import. From here you can upload the JSON files or paste the JSON text directly into the text area.  Below are the locations of the dashboards that can be imported:

| Dashboard           | Description |
| ------------------- | --------------------------------- |
| [I/O Performance by node dashboard](https://github.com/dell/karavi-metrics-powerflex/blob/main/grafana/dashboards/powerflex/sdc_io_metrics.json) | Provides visibility into the I/O performance metrics (IOPS, bandwidth, latency) by Kubernetes node |
| [I/O Performance by volume dashboard](https://github.com/dell/karavi-metrics-powerflex/blob/main/grafana/dashboards/powerflex/volume_io_metrics.json) | Provides visibility into the I/O performance metrics (IOPS, bandwidth, latency) by volume |
| [Storage consumption dashboard](https://github.com/dell/karavi-metrics-powerflex/blob/main/grafana/dashboards/powerflex/storage_consumption.json) | Provides visibility into the total, used, and available capacity for a storage class and associated underlying storage construct. |
| [Storage consumption dashboard](https://github.com/dell/karavi-metrics-powerflex/blob/main/grafana/dashboards/powerflex/storage_consumption.json) | Provides visibility into Dell EMC CSI (Container Storage Interface) driver provisioned volume characteristics in Kubernetes correlated with volumes on the storage system. |
