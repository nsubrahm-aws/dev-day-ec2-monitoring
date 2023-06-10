# Introduction

This repo holds content to be used for demonstration of monitoring of EC2 instances with Amazon Managed Service for Prometheus and Amazon Managed Grafana.

- [Introduction](#introduction)
  - [Overview](#overview)

## Overview

Generally, a workload can generate three types of metrics viz. System metrics, custom metrics and external metrics.

- **System metrics**. These metrics describe the performance of underlying infra-structure e.g. CPU, RAM, storage, networking etc.
- **Custom metrics**. These metrics are specific to a given application e.g. number of logged in users, average response time of an API, etc.
- **External metrics**. These metrics are of components that are required for a given workload to function e.g. a load balancer, Apache Kafka broker, RDBMS, etc.

This repo describes the generation and visualization of all the above metrics type.

| Metric type | Demo documentation                                                                    |
| ----------- | ------------------------------------------------------------------------------------- |
| System      | [Viewing system metrics with `collectd`.](docs/collectd-demo-steps.md)                |
| Custom      | [Viewing custom metrics with `statsd`.](docs/statsd-demo-steps.md)                    |
| External    | [Viewing external metrics with `yace` and Amazon CloudWatch](docs/yace-demo-steps.md) |
