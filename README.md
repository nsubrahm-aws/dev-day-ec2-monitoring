# Introduction

This repo holds content to be used for demonstration of monitoring of EC2 instances with Amazon Managed Service for Prometheus and Amazon Managed Grafana.

- [Introduction](#introduction)
  - [Overview](#overview)

## Overview

This repo describes the generation and visualization of the following metrics type.

| Metric type  | Short description                                                                                                                  | Demo documentation                                                                    |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| **System**   | Performance of underlying infra-structure e.g. CPU, RAM, storage, networking etc.                                                  | [Viewing system metrics with `collectd`.](docs/collectd-demo-steps.md)                |
| **Custom**   | Specific to a given application e.g. number of logged in users, average response time of an API, etc.                              | [Viewing custom metrics with `statsd`.](docs/statsd-demo-steps.md)                    |
| **External** | Pertaining to components that are required for a given workload to function e.g. a load balancer, Apache Kafka broker, RDBMS, etc. | [Viewing external metrics with `yace` and Amazon CloudWatch](docs/yace-demo-steps.md) |
