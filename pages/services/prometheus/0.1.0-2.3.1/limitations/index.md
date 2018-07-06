---
layout: layout.pug
navigationTitle:  Limitations
title: Limitations
menuWeight: 110
excerpt: Understanding configuration Limitations
featureMaturity:
enterprise: false
---

## Out-of-band configuration

Out-of-band configuration modifications are not supported. The service's core responsibility is to deploy and maintain the service with a specified configuration. In order to do this, the service assumes that it has ownership of task configuration. If an end-user makes modifications to individual tasks through out-of-band configuration operations, the service will override those modifications at a later time. For example:

- If a task crashes, it will be restarted with the configuration known to the scheduler, not one modified out-of-band.
- If a configuration update is initiated, all out-of-band modifications will be overwritten during the rolling update.

Prometheus also does not offer durable long-term storage, anomaly detection, automatic horizontal scaling and user management. 
Prometheus is not a dashboarding solution, it features a simple UI for experimentation with PromQL queries but relies on Grafana for dashboarding, adding some additional setup complexity.

## Push Gateway

The Prometheus Pushgateway allows you to push time series from short-lived service-level batch jobs to an intermediary job which Prometheus can scrape.Push Gateway is out of scope in this release.

## Scaling in

To prevent accidental data loss, the service does not support reducing the number of pods.

## Disk changes

To prevent accidental data loss from reallocation, the service does not support changing volume requirements after initial deployment.

## Best-effort installation

If your cluster doesn't have enough resources to deploy the service as requested, the initial deployment will not complete until either those resources are available or until you reinstall the service with corrected resource requirements. Similarly, scale-outs following initial deployment will not complete if the cluster doesn't have the needed available resources to complete the scale-out.

## Virtual networks

When the service is deployed on a virtual network, the service may not be switched to host networking without a full re-installation. The same is true for attempting to switch from host to virtual networking.

## Task Environment Variables

Each service task has some number of environment variables, which are used to configure the task. These environment variables are set by the service scheduler. While it is possible to use these environment variables in adhoc scripts (e.g. via `dcos task exec`), the name of a given environment variable may change between versions of a service and should not be considered a public API of the service.

## Configurations

The “disk” configuration value is denominated in MB. We recommend you set the configuration value log_retention_bytes to a value smaller than the indicated “disk” configuration. See the Configuring section for instructions for customizing these values.

## Legacy User Support

Legacy Authorized Users File is not supported.

## LDAP Integration, TLS

LDAP integration, TLS is not supported.
https://prometheus.io/docs/introduction/faq/#why-don't-the-prometheus-server-components-support-tls-or-authentication?-can-i-add-those?


## Installation Limitations

The minimum memory requirement for Prometheus installation is hard to determine as it depends on the machine to machine and requirement of organization,

You could refer link to get an idea on how you can get and idea on Installation requirment https://www.robustperception.io/how-much-ram-does-my-prometheus-need-for-ingestion/

Prometheus installation will take time since the Prometheus application with base build launches alert manager 
The approximate installation time required would be around 5-10 Minutes for a HA-Prometheus Server and HA- Alert Manager.