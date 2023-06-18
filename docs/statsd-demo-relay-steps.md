# Demo - `statsd` with Amazon Managed Service for Prometheus and Amazon Managed Grafana - Relay mode

This document has the steps to launch the demo to monitor custom metrics gathered by `statsd`. These custom metrics are written remotely to Amazon Managed Service for Prometheus and visualized in Amazon Managed Grafana.

> This demo will be launched in Singapore (`ap-southeast-1`) region. This demo has been tested with Amazon EC2 instances running Ubuntu-22.04 operating system.
 
- [Demo - `statsd` with Amazon Managed Service for Prometheus and Amazon Managed Grafana - Relay mode](#demo---statsd-with-amazon-managed-service-for-prometheus-and-amazon-managed-grafana---relay-mode)
  - [Overview](#overview)
  - [Step 1 - Launch EC2 instance](#step-1---launch-ec2-instance)
  - [Step 2 - Install, configure and launch `statsd` in Relay mode](#step-2---install-configure-and-launch-statsd-in-relay-mode)
  - [Step 3 - Set-up Amazon Managed Services for Prometheus](#step-3---set-up-amazon-managed-services-for-prometheus)
  - [Step 4 - Install, configure and launch ADOT server](#step-4---install-configure-and-launch-adot-server)
  - [Step 5 - Launch NodeJs application](#step-5---launch-nodejs-application)
  - [Step 7 - Install `hey` to simulate clients](#step-7---install-hey-to-simulate-clients)
  - [Step 8 - Set-up Amazon Managed Grafana](#step-8---set-up-amazon-managed-grafana)

## Overview

This demo will implement the architecture diagram shown below.

![Image](../png/statsd-relay-arch-light.png#gh-light-mode-only)
![Image](../png/statsd-relay-arch-dark.png#gh-dark-mode-only)

1. One EC2 instance will be launched named as `Linux`. Open port `4000` from your local environment to test API invocation.
2. The `statsd` metrics will be gathered by a Prometheus server.
3. The Prometheus server is configured to
   1. Scrape the `statsd_exporter` installed on `Linux` instance.
   2. Write remotely to Amazon Managed Service for Prometheus.
   3. Have an IAM role that allows write remotely to Amazon Managed Service for Prometheus.
4. One workspace each for Amazon Managed Service for Prometheus and Amazon Managed Grafana.
5. Recording rules are configured for Amazon Managed Service for Prometheus.
6. The Amazon Managed Grafana instance is configured to use Amazon Managed Service for Prometheus as a data source.
7. The Amazon Managed Grafana instance has a dashboard configured to display the system metrics gathered via `statsd`.

## Step 1 - Launch EC2 instance

This step will launch EC2 instances.

1. [Launch and connect to Amazon EC2 instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html) named `Linux`.
  1. Region - `ap-southeast-1`. See [here](https://docs.aws.amazon.com/prometheus/latest/userguide/what-is-Amazon-Managed-Service-Prometheus.html#AMP-supported-Regions) for a list of supported regions for Amazon Managed Service for Prometheus.
  2. Instance type - `t2.medium`.
  3. Select option for creating a new key pair.
  4. Security group - `ssh` and port `4000` from your IP address.

## Step 2 - Install, configure and launch `statsd` in Relay mode

The following instructions are to be run after [logging](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html#ec2-connect-to-instance-linux) into EC2 instance named `Linux`.

1. Install NodeJs

```bash
sudo apt-get update
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs
```

2. Install `statsd`

```bash
git clone https://github.com/statsd/statsd
cd statsd
npm i
cd $HOME
```

3. Save the following into `$HOME/statsd-config.js`.

```js
{
 backends: [ "./backends/console" ]
}
```

4. Install `statsd_exporter`

```bash
cd $HOME
curl -sOL https://github.com/prometheus/statsd_exporter/releases/download/v0.22.4/statsd_exporter-0.22.4.linux-amd64.tar.gz
tar -xvf statsd_exporter-0.22.4.linux-amd64.tar.gz
cd statsd_exporter-0.22.4.linux-amd64
cd $HOME
```

5. Save the following into `$HOME/statsd-mapping-config.yaml`

```yaml
mappings:
- match: "api.*.*.*"
  name: "api_metrics"
  labels:
    path: "$1"
    method: "$2"
    status: "$3"
```

6. Launch `statsd_exporter`

```bash
cd $HOME
statsd_exporter-0.22.4.linux-amd64/statsd_exporter --statsd.mapping-config=statsd-mapping-config.yaml --statsd.relay.address=localhost:8125
```

7. Open a new [connection](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html#ec2-connect-to-instance-linux) to the EC2 instance named `Linux` and run the following command to start `statsd`.

```bash
cd $HOME/statsd
node stats.js $HOME/statsd-config.js
```

## Step 3 - Set-up Amazon Managed Services for Prometheus

The following instructions are to be followed from the AWS console.

1. [Create](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-onboard-create-workspace.html) the Amazon Managed Service for Prometheus workspace.
2. [Upload](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-rules-upload.html) recording rules, as shown below, to Amazon Managed Service for Prometheus workspace created earlier.

```yaml
groups:
  - name: statsd
    rules:
    - record: metric:api_metrics
      expr: sum(rate(api_metrics[5m])) by (path, status)
```

## Step 4 - Install, configure and launch ADOT server

1. Install ADOT.

```bash
wget https://aws-otel-collector.s3.amazonaws.com/ubuntu/amd64/latest/aws-otel-collector.deb
sudo dpkg -i -E ./aws-otel-collector.deb 
```

2. Save the following into `adot-config.yaml` to configure ADOT. Change `regionId` to `ap-southeast-1` and `workspaceId` to the value copied in Step 5.

```yaml
receivers:
  prometheus:
    config:
      global:
        scrape_interval: 15s
        external_labels:
          monitor: 'statsd_relay_exporter'

      scrape_configs:
        - job_name: 'statsd_relay_exporter'
          static_configs:
            - targets: ['localhost:9102']

extensions:
  sigv4auth:
    region: "regionId"

exporters:
  prometheusremotewrite:
    endpoint: https://aps-workspaces.regionId.amazonaws.com/workspaces/workspaceId/api/v1/remote_write
    auth:
      authenticator: sigv4auth

service:
  extensions: [sigv4auth]
  pipelines:
    metrics:
      receivers: [prometheus]
      exporters: [prometheusremotewrite]
```

3. Launch ADOT server.

```bash
sudo /opt/aws/aws-otel-collector/bin/aws-otel-collector-ctl -c adot-config.yaml -a start
```

## Step 5 - Launch NodeJs application

Open a new [connection](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html#ec2-connect-to-instance-linux) to the EC2 instance named `Linux` and run the following commands to launch a sample NodeJs application.

1. Set-up for creating the Nodejs application.

```bash
mkdir app
cd app
npm init -y
npm i express node-statsd --save
```

2. Copy the following into `app/index.js`.

```js
const express = require('express')
const StatsD = require('node-statsd')

const statsdClient = new StatsD({
  host: "localhost",
  port: 9125
})

const app = express()
const port = 4000

let incrVar

app.get('/hello', (req, res) => {
  incrVar = `${req.path}.${req.method}.200`
  incrVar = `api.${incrVar.substring(1)}`
  statsdClient.increment(incrVar)

  res.status(200).send('Hello World!')
})

app.use('/', (req, res) => {
  incrVar = `${req.path}.${req.method}.404`
  incrVar = `api.${incrVar.substring(1)}`
  statsdClient.increment(incrVar)

  res.status(404).send('Not found.')
})

app.listen(port, () => {
  console.log(`Example app listening on port ${port}`)
})
```

3. Launch the NodeJs application with the following commands.

```bash
cd $HOME/app
node index.js
```

## Step 7 - Install `hey` to simulate clients

On your local environment, install `hey` following instructions from [GitHub repository](https://github.com/rakyll/hey).

1. Open two terminal instances on your local environment.
2. Run the following commands.

```bash
hey -z 5m http://ec2-public-ip:4000/hello
hey -z 5m http://ec2-public-ip:4000/bye
```

## Step 8 - Set-up Amazon Managed Grafana

The following instructions are to be followed from the AWS console for launching Amazon Managed Grafana instance.

1. [Create](https://catalog.workshops.aws/observability/en-US/amg/setupamg-saml) an Amazon Managed Grafana workspace with SAML user access.
2. [Add](https://docs.aws.amazon.com/grafana/latest/userguide/prometheus-data-source.html) the Amazon Managed Service for Prometheus data source.
3. Click on **Go to settings** and change the **Name** to `ec2-statsd`.
4. [Create](https://docs.aws.amazon.com/grafana/latest/userguide/dashboard-export-and-import.html#importing-a-dashboard) a new dashboard with the Import option.
5. Copy the following JSON document into a file and [upload](https://docs.aws.amazon.com/grafana/latest/userguide/dashboard-export-and-import.html#importing-a-dashboard) it to create a new dashboard.

```json
{"annotations":{"list":[{"builtIn":1,"datasource":"ec2-statsd","enable":true,"hide":true,"iconColor":"rgba(0, 211, 255, 1)","name":"Annotations & Alerts","target":{"limit":100,"matchAny":false,"tags":[],"type":"dashboard"},"type":"dashboard"}]},"editable":true,"fiscalYearStartMonth":0,"graphTooltip":0,"id":43,"links":[],"liveNow":true,"panels":[{"datasource":{"type":"prometheus","uid":"bqVqqm_4z"},"fieldConfig":{"defaults":{"color":{"mode":"palette-classic"},"custom":{"axisLabel":"","axisPlacement":"auto","barAlignment":0,"drawStyle":"line","fillOpacity":0,"gradientMode":"none","hideFrom":{"legend":false,"tooltip":false,"viz":false},"lineInterpolation":"smooth","lineWidth":1,"pointSize":5,"scaleDistribution":{"type":"linear"},"showPoints":"auto","spanNulls":false,"stacking":{"group":"A","mode":"none"},"thresholdsStyle":{"mode":"off"}},"mappings":[],"thresholds":{"mode":"absolute","steps":[{"color":"green","value":null},{"color":"red","value":80}]}},"overrides":[]},"gridPos":{"h":19,"w":24,"x":0,"y":0},"id":2,"options":{"legend":{"calcs":[],"displayMode":"list","placement":"bottom"},"tooltip":{"mode":"single","sort":"none"}},"targets":[{"datasource":{"type":"prometheus","uid":"bqVqqm_4z"},"exemplar":true,"expr":"metric:api_metrics{path=~\"bye|hello\"}","interval":"","legendFormat":"{{path}}-{{status}}","refId":"A"}],"title":"API metrics (rate)","type":"timeseries"}],"refresh":"5s","schemaVersion":35,"style":"dark","tags":[],"templating":{"list":[]},"time":{"from":"now-5m","to":"now"},"timepicker":{},"timezone":"browser","title":"statsd","uid":"5fn7dTY7z","version":1,"weekStart":""}
```

6.  Dashboard is live!

![repeater-dashboard](./../png/statsd-dashboard.png)
