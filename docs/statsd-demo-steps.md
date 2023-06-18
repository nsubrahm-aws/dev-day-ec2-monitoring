# Demo - `statsd` with Amazon Managed Service for Prometheus and Amazon Managed Grafana

The demo with `statsd` has two variants depending on the way `statsd_exporter` can be configured viz. **Relay** and **Repeater** modes. This page introduces the modes and provides links to the actual demo pages.

- [Demo - `statsd` with Amazon Managed Service for Prometheus and Amazon Managed Grafana](#demo---statsd-with-amazon-managed-service-for-prometheus-and-amazon-managed-grafana)
  - [`statsd` overview](#statsd-overview)
  - [Demo](#demo)

## `statsd` overview

In the following figure, a typical deployment of an application performance monitoring solution with `statsd` is shown. In that deployment, an application is instrumented with `statsd` clients and writes its custom metrics to the `statsd` process configured to one or more back-ends.

![Image](../png/statsd-ec2-light.png#gh-light-mode-only)
![Image](../png/statsd-ec2-dark.png#gh-dark-mode-only)

To integrate with Prometheus server, [`statsd_exporter` – Prometheus official exporter for `statsd`](https://github.com/prometheus/statsd_exporter) – is used. There are two ways to utilize the exporter – **Repeater** and **Relay**. The Repeater use case may be used when the `statsd` process can be configured. The application need not be changed. The Relay use case may be used when the `statsd` process is unavailable for configuration. However, the application may have to be reconfigured to the `statsd_exporter` end-point.

## Demo

1. [Viewing custom metrics with `statsd` - Relay](./statsd-demo-relay-steps.md)
2. [Viewing custom metrics with `statsd` - Repeater](./statsd-demo-repeater-steps.md)
