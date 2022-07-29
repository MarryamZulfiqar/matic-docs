---
id: validator-hosting
title: Validator Hosting
description: "Hosting requirements for Polygon Edge"
keywords:
- docs
- polygon
- edge
- hosting
- cloud
- setup
- validator
---

Below are the suggestions for properly hosting a validator node in a Polygon Edge network. Please pay careful attention to all the items listed below to make sure 
that your validator setup is properly configured to be secure, stable and performant.

# Minimum system requirements

| Type | Value | Influenced by |
| ---- | ----- | ----------- |
| CPU | 2 cores | <ul><li>Number of JSON-RPC queries</li><li>Size of the blockchain state</li><li>Block gas limit</li><li>Block time</li></ul>
| RAM | 2 GB | <ul><li>Number of JSON-RPC queries</li><li>Size of the blockchain state</li><li>Block gas limit</li></ul>
| Disk | <ul><li>10 GB root patition</li><li>30 GB root partition with LVM for disk extension</li></ul> | <ul><li>Size of the blockchain state</li></ul>


# Host configuration recommendations

## Service configuration

`polygon-edge` binary needs to run as a system service automatically upon established network connectivity and have start / stop / restart
functionalities. We recommend using a service manager like `systemd.` 

Example `systemd` system configuration file:
```
[Unit]
Description=Polygon Edge Server
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=10
User=ubuntu
ExecStart=/usr/local/bin/polygon-edge server --config /home/ubuntu/polygon/config.yaml

[Install]
WantedBy=multi-user.target
```

##  Binary

In production workloads `polygon-edge` binary should only be deployed from pre-built GitHub release binaries - not by manually compiling.

Please refer to [Installation](/docs/edge/get-started/installation) for a complete overview of installation method.

## Data storage

The `data/` folder  containing the entire blockchain state should be mounted on a dedicated disk / volume allowing for
automatic disk backups, volume extension and optionally mounting the disk/volume to another instance in case of failure.


## Log files

Log files need to be rotated on a daily basis (with a tool like `logrotate`).
If configured without log rotation, log files could use up all the available disk space which could disrupt the validator uptime.

Example `logrotate` configuration:
```
/home/ubuntu/polygon/logs/node.log
{
        rotate 7
        daily
        missingok
        notifempty
        compress
        prerotate
                /usr/bin/systemctl stop polygon-edge.service
        endscript
        postrotate
                /usr/bin/systemctl start polygon-edge.service
        endscript
}
```


Refer to the [Logging](#logging) section below for recommendations on log storage.

## Backup

Weekly backup procedure implemented on a VM / volume level (blockchain data volume).

## Additional dependencies

`polygon-edge` is statically compiled, requiring no additional host OS dependencies.

# Maintenance

Below are the best practices for maintaining a running validator node of a Polygon Edge network.

## Backup

There are two types of backup procedures recommended for Polygon Edge nodes. 

The suggestion is to use both, whenever possible, with the Polygon Edge backup being an always available option.

### Volume backup

A daily incremental backup of the `data/` volume of the Polygon Edge node, or of the complete VM if possible.

### Polygon Edge backup 

A daily CRON job which does regular backups of Polygon Edge and moves the `.dat` files to an offsite location or to a secure cloud object storage is recommended. 

The Polygon Edge backup should ideally not overlap with the Volume backup described above.

Refer to [Backup/restore node instance](edge/working-with-node/backup-restore) for instructions on how to perform backups of Polygon Edge.

### Logging

The logs outputted by the Polygon Edge nodes should:
- be sent to an external data store with indexing and searching capabilities
- have a log retention period of 30 days

If this is your first time setting up a Polygon Edge validator, we recommend to start the node
with the `--log-level=DEBUG` option to be able to quickly debug any issues you might face.

The `--log-level=DEBUG` will make the node's log output be as verbose as possible.

## OS security patches

Administrators need to ensure that the validator instance OS is always updated with the latest patches at least once every month.

## System metrics

Administrators need to setup some kind of system metrics monitor, (e.g. Telegraf + InfluxDB + Grafana or a 3rd party SaaS).

Metrics that need to be monitored and that need to have alarm notifications setup:

| Metric name | Alarm threshold |
| ----------- | --------------- |
| CPU usage (%) | > 90% for more than 5 minutes |
| RAM utilization (%) | > 90% for more than 5 minutes |
| Root disk utilization | > 90% |
| Data disk utilization | > 90% |

## Validator metrics

Administrators need to setup collection of metrics from Polygon Edge's Prometheus API to be able to
monitor the blockchain performance.

Refer to [Prometheus metrics](edge/configuration/prometheus-metrics) to understand which metrics are being exposed and how to set up Prometheus metric collection.


Extra attention needs to be paid to the following metrics:
- *Number of consensus rounds* - if there is more than 1 round, there is a potential problem with the validator set in the network
- *Number of peers* - if the number of peers drop, there is a connectivity issue in the network