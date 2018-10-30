# kubernetes-graylog-cluster
Quote pires/kubernetes-elasticsearch-cluster (Elasticsearch (5.5.1) cluster), Build a multi-node, highly available graylog service

### Table of Contents

* [Important Notes](#important-notes)
* [Pre-Requisites](#pre-requisites)
* [Build-Images(optional)](#build-images)
* [Test (deploying & accessing)](#test)
* [Pod anti-affinity](#pod-anti-affinity)
* [Install plug-ins](#plugins)
* [FAQ](#faq)
* [Troubleshooting](#troubleshooting)

## Architecture decomposition

Elasticsearch best-practices recommend to separate nodes in three roles:
* `Master` nodes - intended for clustering management only, no data, no HTTP API
* `Client` nodes - intended for client usage, no data, with HTTP API
* `Data` nodes - intended for storing and indexing data, no HTTP API

Graylog server cluster(In this case):
* `graylog master` node - Mainly to provide Web Ui, user-friendly management, does not assume the function of data input
* `graylog data` node - The main function is to be responsible for receiving the data and processing it and then storing it in elasticsearch, no Web ui

Mongodb server cluster(In this case)
* `mongodb server` node - storing meta information and configuration data and doesn’t need many resources(example:input、save search、node list...etc)

In this case, Graylog has three data processing nodes and a master management node, in the data node front has a load balancing server (in this case, by the k8s service as LB). According to the official multi-node cluster construction, Mongodb needs to build mongo rs, but in this case, did not do so. Because K8s rs can be a perfect management mongodb, we only need to mongodb pod storage mount to the shared storage server can be, so Just a mongodb instance and a persistent store. The important part is the elasticsearch cluster, we used the best practice, namely data, client, management server, separate run. Perform their duties. They are through the 9300 port between the mutual discovery, the establishment of cluster relations.


<a id="important-notes">

## (Very) Important notes

* Elasticsearch pods need for an init-container to run in privileged mode, so it can set some VM options. For that to happen, the `kubelet` should be running with args `--allow-privileged`, otherwise
the init-container will fail to run.
In other ways, you can also use ansible such a similar tool to batch modify k8s node kernel parameters.(example: vm.max_map_count)


* By default, `ES_JAVA_OPTS` is set to `-Xms256m -Xmx256m`. This is a *very low* value but many users, i.e. `minikube` users, were having issues with pods getting killed because hosts were out of memory.
One can change this in the deployment descriptors available in this repository.


<a id="pre-requisites">

## Pre-requisites

* Kubernetes cluster with **alpha features enabled** (tested with v1.7.2 on top of [Vagrant + CoreOS](https://github.com/pires/kubernetes-vagrant-coreos-cluster))
* `kubectl` configured to access the cluster master API Server

<a id="build-images">

## Build images (optional)

* elasticsearch 5.5.1：`https://github.com/aliasmee/docker-elasticsearch-kubernetes`
* graylog 2.3.0: `https://github.com/aliasmee/graylog-docker`
* mongodb 3: `https://github.com/aliasmee/mongo`


## Test

### K8s Deploy
#### Create elasticsearch cluster 
1. create es client service ，Responsible for LB backend 9200 port

```kubectl create -f es-svc-qcloud-lb.yaml```

2. create es discovery service ，Responsible for publishing es service discovery 9300 port

```kubectl create -f es-discovery-svc.yaml```

3. create es-master deployment，Responsible for index finding, routing, and maintaining cluster information.Election master

```kubectl create -f es-master.yaml```


4. create es-data deployment，Responsible for storing index data, where the local disk is used to persist (mount the /data/esnode directory on Node)

```kubectl create -f es-data.yaml```

5. create es-client，Responsible for the client to access es cluster entry, also provide api call.

```kubectl create -f es-client.yaml```

OK，Now the whole es cluster has been running up. The next step is to create mongodb

Tips：When creating an es cluster, some container states may appear in the following case: "su-exec: /elasticsearch/bin/elasticsearch: Text file busy" ，Do not worry, wait for a while

#### Create mongodb service
1. create mongodb deployment，esponsible for storing metadata and configuration information for graylog. Persistent storage（NFS）

```kubectl create -f mongodb.yaml```

2. create mongodb svc，just provide access to graylog

```kubectl create -f mongodb-svc.yaml```

#### Create graylog master node
1. create graylog  service，Used to be responsible for the entire graylog api, web ui access interface.

```kubectl create -f graylog-svc.yaml```

2. create graylog-master node

```kubectl create -f graylog.yaml```


#### Create graylog node（receive data source）node
1. create graylog-node deploy，Responsible for receiving the main data source input. Share a mongo instance with graylog master

```kubectl create -f graylog-node```

2. create graylog-node service ，The input of the data source accesses the ip interface(Can open more ports according to actual needs.)

```kubectl create -f graylog-node-svc.yaml```

#### Login graylog's web interface
`Here to do a little modification`
1. click system-->indices,Modify as follows
```
Index shards：3
Index replicas：1
```

Tips：shards According to your es node to plan, here is the three es-data node, so here is shard set to 3. And set up a replicas.
When two of the nodes are damaged, the cluster will still provide services. Not red When the node is restored, the index is rebuilt. Will change from yellow to green. Note: If your replicas is 0, then the broken one node, the entire es cluster will immediately become red. Can not provide services, unless the restoration of that bad node.

Example graylog Error info：Failed to index [4] messages. Please check the index error log in your web interface for the reason. Error: One or more of the items in the Bulk request failed, check BulkResult.getItems() for more information

👌，Show time!

### Access the service

*Don't forget* that services in Kubernetes are only acessible from containers in the cluster. For different behavior one should [configure the creation of an external load-balancer](http://kubernetes.io/v1.1/docs/user-guide/services.html#type-loadbalancer). While it's supported within this example service descriptor, its usage is out of scope of this document, for now.


```
$ kubectl get svc elasticsearch 
NAME            CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
elasticsearch   10.100.68.102   <pending>     9200:30536/TCP   3m
```

From any host on the Kubernetes cluster (that's running `kube-proxy` or similar), run:

```
curl http://10.100.68.102:9200
```

One should see something similar to the following:

```json
{
  "name" : "es-client-3159607856-cj49h",
  "cluster_name" : "graylog",
  "cluster_name" : "graylog",
  "cluster_uuid" : "m0jm8ASlSaKOS0wT52R5cA",
  "version" : {
    "number" : "5.5.1",
    "build_hash" : "19c13d0",
    "build_date" : "2017-07-18T20:44:24.823Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.0"
  },
  "tagline" : "You Know, for Search"
}
```

Or if one wants to see cluster information:

```
curl http://10.100.68.102:9200/_cluster/health?pretty
```

One should see something similar to the following:

```json
{
  "cluster_name" : "graylog",
  "cluster_name" : "graylog",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 7,
  "number_of_data_nodes" : 2,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```
<a id="pod-anti-affinity">

## Pod anti-affinity

One of the main advantages of running Elasticsearch on top of Kubernetes is how resilient the cluster becomes, particularly during
node restarts. However if all data pods are scheduled onto the same node(s), this advantage decreases significantly and may even
result in no data pods being available.

It is then **highly recommended**, in the context of the solution described in this repository, that one adopts [pod anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#inter-pod-affinity-and-anti-affinity-beta-feature)
in order to guarantee that two data pods will never run on the same node.

Here's an example:
```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: role
              operator: In
              values:
              - data
          topologyKey: kubernetes.io/hostname
  containers:
  - (...)
```

<a id="plugins">

## Install plug-ins

The image used in this repo is very minimalist. However, one can install additional plug-ins at will by simply specifying the `ES_PLUGINS_INSTALL` environment variable in the desired pod descriptors. For instance, to install Google Cloud Storage and X-Pack plug-ins it would be like follows:
```yaml
- name: "ES_PLUGINS_INSTALL"
  value: "repository-gcs,x-pack"
```

## FAQ

### Why does `NUMBER_OF_MASTERS` differ from number of master-replicas?
The default value for this environment variable is 2, meaning a cluster will need a minimum of 2 master nodes to operate. If a cluster has 3 masters and one dies, the cluster still works. Minimum master nodes are usually `n/2 + 1`, where `n` is the number of master nodes in a cluster. If a cluster has 5 master nodes, one should have a minimum of 3, less than that and the cluster _stops_. If one scales the number of masters, make sure to update the minimum number of master nodes through the Elasticsearch API as setting environment variable will only work on cluster setup. More info: https://www.elastic.co/guide/en/elasticsearch/guide/1.x/_important_configuration_changes.html#_minimum_master_nodes


### How can I customize `elasticsearch.yaml`?
Read a different config file by settings env var `path.conf=/path/to/my/config/`. Another option would be to build one's own image from  [this repository](https://github.com/pires/docker-elasticsearch-kubernetes)

## Troubleshooting

### No up-and-running site-local

One of the errors one may come across when running the setup is the following error:
```
[2016-11-29T01:28:36,515][WARN ][o.e.b.ElasticsearchUncaughtExceptionHandler] [] uncaught exception in thread [main]
org.elasticsearch.bootstrap.StartupException: java.lang.IllegalArgumentException: No up-and-running site-local (private) addresses found, got [name:lo (lo), name:eth0 (eth0)]
	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:116) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.bootstrap.Elasticsearch.execute(Elasticsearch.java:103) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.cli.SettingCommand.execute(SettingCommand.java:54) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:96) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.cli.Command.main(Command.java:62) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:80) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:73) ~[elasticsearch-5.0.1.jar:5.0.1]
Caused by: java.lang.IllegalArgumentException: No up-and-running site-local (private) addresses found, got [name:lo (lo), name:eth0 (eth0)]
	at org.elasticsearch.common.network.NetworkUtils.getSiteLocalAddresses(NetworkUtils.java:187) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.common.network.NetworkService.resolveInternal(NetworkService.java:246) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.common.network.NetworkService.resolveInetAddresses(NetworkService.java:220) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.common.network.NetworkService.resolveBindHostAddresses(NetworkService.java:130) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.transport.TcpTransport.bindServer(TcpTransport.java:575) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.transport.netty4.Netty4Transport.doStart(Netty4Transport.java:182) ~[?:?]
 	at org.elasticsearch.common.component.AbstractLifecycleComponent.start(AbstractLifecycleComponent.java:68) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.transport.TransportService.doStart(TransportService.java:182) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.common.component.AbstractLifecycleComponent.start(AbstractLifecycleComponent.java:68) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.node.Node.start(Node.java:525) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.bootstrap.Bootstrap.start(Bootstrap.java:211) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.bootstrap.Bootstrap.init(Bootstrap.java:288) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:112) ~[elasticsearch-5.0.1.jar:5.0.1]
 	... 6 more
[2016-11-29T01:28:37,448][INFO ][o.e.n.Node               ] [kIEYQSE] stopping ...
[2016-11-29T01:28:37,451][INFO ][o.e.n.Node               ] [kIEYQSE] stopped
[2016-11-29T01:28:37,452][INFO ][o.e.n.Node               ] [kIEYQSE] closing ...
[2016-11-29T01:28:37,464][INFO ][o.e.n.Node               ] [kIEYQSE] closed
```

This is related to how the container binds to network ports (defaults to ``_local_``). It will need to match the actual node network interface name, which depends on what OS and infrastructure provider one uses. For instance, if the primary interface on the node is `p1p1` then that is the value that needs to be set for the `NETWORK_HOST` environment variable.
Please see [the documentation](https://github.com/pires/docker-elasticsearch#environment-variables) for reference of options.

In order to workaround this, set `NETWORK_HOST` environment variable in the pod descriptors as follows:
```yaml
- name: "NETWORK_HOST"
  value: "_eth0_" #_p1p1_ if interface name is p1p1, _ens4_ if interface name is ens4, and so on.
```

### (IPv6) org.elasticsearch.bootstrap.StartupException: BindTransportException

Intermittent failures occur when the local network interface has both IPv4 and IPv6 addresses, and Elasticsearch tries to bind to the IPv6 address first.
If the IPv4 address is chosen first, Elasticsearch starts correctly.

In order to workaround this, set `NETWORK_HOST` environment variable in the pod descriptors as follows:
```yaml
- name: "NETWORK_HOST"
  value: "_eth0:ipv4_" #_p1p1:ipv4_ if interface name is p1p1, _ens4:ipv4_ if interface name is ens4, and so on.
```
