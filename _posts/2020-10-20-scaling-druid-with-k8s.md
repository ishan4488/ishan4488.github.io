---
title: "Auto-Scaling Druid with Kubernetes"
date: 2020-10-15
categories:
  - Druid
tags:
  - Apache Druid
  - Kubernetes
  - K8S
---

```ruby
Autoscale Druid using 
1. Kubernetes Cluster Autoscaler 
2. KEDA - Kubernetes Event Driven Autoscaling
```

> In this post we will cover the scaling of MiddleManager nodes. The autoscaling concepts remain the same for all the Druid services.

### Background and Problem Statement: 
In one of our projects, we had a large number of pipelines feeding data concurrently, via Airflow, to Druid. Due to a large number of concurrent ingestion tasks, the ingestion tasks were being queued and were taking a couple of hours to clear out. This was delaying the latest data availability in Druid and the requirement was to speed up ingestion.

Due to the spiky nature of the ingestion tasks and to keep costs in control, having dedicated MiddleManager nodes (dedicated infra) wasn't the best way forward.

### KEDA 

By using [KEDA](https://keda.sh/) users can dynamically scale up or scale down the pods (via Deployments or StatefulSets).

KEDA can determine the replica count by reading values from external systems. [These](https://keda.sh/docs/2.0/scalers/) integrations are supported in KEDA.

In Druid, the pending ingestion tasks will be read from Postgres. Based on the number of queued ingestion tasks, we will scale the number of MiddleManager nodes.


```
select count(*) + 1 from druid_tasks where active = 1
```
KEDA will autoscale the MiddleManager StatefulSet.


### How it works?
KEDA polls the Postgres DB every 30 seconds and checks for queued ingestion tasks. The moment KEDA notices pending ingestion tasks, it will start adding MiddleManager pods. 

The sql query given below defines the number of MiddleManager pods we need at any given time. We can define the upper and the lower cap in the KEDA configuration given below.

```
select count(*) + 1 from druid_tasks where active = 1
```

If there are not enough resources to schedule new pods, the Cluster Autoscaler will kick in and will increase the number of physical nodes in the cluster.

Same goes during scale down events. When the ingestion tasks finish, KEDA downscales the StatefulSet. 
Once the pods are deleted and there is buffer capacity with the cluster, the Cluster Autoscaler will downscale the physical nodes.

### KEDA Configuration

Sample KEDA configuration for autoscaling MiddleManager nodes:

To apply the following configuration: kubectl apply -f scale-middle-manager.yml -n `<namespace>`

```yml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: druid-middle-manager-scaler #  name of your scaled object
spec:
  scaleTargetRef:
    apiVersion:    apps/v1  
    kind:          StatefulSet                       
    name:          middle-manager                    # Mandatory. The name of the middle-manager StatefulSet in your installation
  pollingInterval: 30                                # Optional. Default: 30 seconds
  cooldownPeriod:  300                               # Optional. Default: 300 seconds
  minReplicaCount: 0                                 # Optional. Default: 0
  maxReplicaCount: 10                                # Optional. Default: 100
  triggers:                                          # The scaler type         
- type: postgresql
  metadata:
    userName: druid
    passwordFromEnv: PG_PASSWORD                     # IMPORTANT - 1 -- PLEASE READ BELOW ABOUT PROVIDING THE PASSWORD
    host: postgres.druid.cluster.local               # use the cluster-wide namespace as KEDA as lives in a different namespace from your postgres
    port: "5432"
    dbName: postgresql
    sslmode: disable
    query: "select count(*) + 1 from druid_tasks where active = 1"  # Users can define their own logic here.
    targetQueryValue: 1 
```

Few nuances to be taken care of: 

1. PG_PASSWORD in the above configuration should be available as an environment variable within the MiddleManager pods. 
You can update the Druid Helm chart to have the following:

Add PG_PASSWORD to the middlemanager.config in values.yml in [Druid Helm Chart](https://github.com/helm/charts/tree/master/incubator/druid).

values.yaml snippet:

```yml
middleManager:
  ## If false, middleManager will not be installed
  ##
  enabled: true
  name: middle-manager
  replicaCount: 1
  port: 8091
  serviceType: ClusterIP

  config:
    DRUID_XMX: 64m
    DRUID_XMS: 64m
    druid_indexer_runner_javaOptsArray: '["-server", "-Xms256m", "-Xmx256m", "-XX:MaxDirectMemorySize=300m", "-Duser.timezone=UTC", "-Dfile.encoding=UTF-8", "-XX:+ExitOnOutOfMemoryError", "-Djava.util.logging.manager=org.apache.logging.log4j.jul.LogManager"]'
    druid_indexer_fork_property_druid_processing_buffer_sizeBytes: '25000000'
    PG_PASSWORD: mypassword
```

PG_PASSWORD in middleManager.config is passed as an environment variable to the MiddleManager pods. 
Please refer to the [druid/templates/middleManager/statefulset.yaml](https://github.com/helm/charts/blob/master/incubator/druid/templates/middleManager/statefulset.yaml#L78)

statefulset.yaml snippet:
```yml
containers:
- name: druid
args: [ "middleManager" ]
env:
  range $key, $val := .Values.middleManager.config
        - name: {{ $key }}
          value: {{ $val | quote }}
  end
```


Please check KEDA documentation on using secrets and more authentication options.

### Tested using AWS EKS, Druid Helm chart, KEDA, EKS Cluster Autoscaler

1. [EKS Cluster Autoscaler](https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html): Installing cluster autoscaler on GCP or Azure should be equally easy.
3. [Druid Helm Chart](https://github.com/helm/charts/tree/master/incubator/druid)
4. [KEDA](https://keda.sh/): KEDA installation is very easy and straightforward.
