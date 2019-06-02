# Deploy Elasticsearch as StatefulSet Pod

This directory contains Kubernetes configurations which run elasticsearch data and master pods as a [`StatefulSet`](https://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/), using storage provisioned using a [`StorageClass`](http://blog.kubernetes.io/2016/10/dynamic-provisioning-and-storage-in-kubernetes.html).

## Storage

The [`es-data-statefulset.yaml`](es-data-statefulset.yaml) and [`es-master-statefulset.yaml`](es-master-statefulset.yaml) files contain `volumeClaimTemplates` sections which request 5GB volume for each master node, and data node. This is plenty of space for a demonstration cluster, but will fill up quickly under moderate to heavy load. Consider modifying the disk size to your needs.

## Deploy
The root directory contains yaml files for deploying elasticsearch cluster and Kibana.

```
kubectl create -f kubernetes-elasticsearch-cluster/

persistentvolumeclaim/esclient-storage created
configmap/esclient-config created
deployment.apps/esclient created
service/elasticsearch created
service/elasticsearch-loadbalancer created
service/elasticsearch-cluster created
configmap/esdata-config created
statefulset.apps/esdata created
service/elasticsearch-data created
configmap/es-config created
statefulset.apps/esmaster created
service/kibana created
deployment.apps/kibana created
```
This will spin up a three node Elasticsearch cluster along with a NodePort service to expose the pod service to the outside world.

Verify that both the pod and service were created:

```
kubectl get pods,pvc,svc
NAME                            READY   STATUS             RESTARTS   AGE
pod/esclient-658d6b9bcd-dwkjc   0/1     CrashLoopBackOff   9          32m
pod/esclient-658d6b9bcd-rk2s4   0/1     Init:0/1           0          32m
pod/esdata-0                    1/1     Running            0          32m
pod/esdata-1                    1/1     Running            0          31m
pod/esdata-2                    1/1     Running            0          31m
pod/esmaster-0                  1/1     Running            0          32m
pod/esmaster-1                  1/1     Running            0          31m
pod/esmaster-2                  1/1     Running            0          31m
pod/kibana-5c8ddfc7ff-cd4zq     1/1     Running            0          32m

NAME                                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
persistentvolumeclaim/es-data-esmaster-0         Bound    pvc-4f8c55e7-854a-11e9-a106-aa6642a6ec3d   5Gi        RWO            do-block-storage   32m
persistentvolumeclaim/es-data-esmaster-1         Bound    pvc-7b213973-854a-11e9-a106-aa6642a6ec3d   5Gi        RWO            do-block-storage   31m
persistentvolumeclaim/es-data-esmaster-2         Bound    pvc-895bef28-854a-11e9-a106-aa6642a6ec3d   5Gi        RWO            do-block-storage   31m
persistentvolumeclaim/es-storage-data-esdata-0   Bound    pvc-4e3f1c43-854a-11e9-a106-aa6642a6ec3d   5Gi        RWO            do-block-storage   32m
persistentvolumeclaim/es-storage-data-esdata-1   Bound    pvc-731c0ac5-854a-11e9-a106-aa6642a6ec3d   5Gi        RWO            do-block-storage   31m
persistentvolumeclaim/es-storage-data-esdata-2   Bound    pvc-8e383cc8-854a-11e9-a106-aa6642a6ec3d   5Gi        RWO            do-block-storage   31m
persistentvolumeclaim/esclient-storage           Bound    pvc-4bb05891-854a-11e9-a106-aa6642a6ec3d   5Gi        RWO            do-block-storage   32m

NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/elasticsearch                NodePort    10.245.244.115   <none>        9200:32537/TCP   35m
service/elasticsearch-cluster        ClusterIP   None             <none>        9300/TCP         35m
service/elasticsearch-data           ClusterIP   None             <none>        9300/TCP         21m
service/elasticsearch-loadbalancer   NodePort    10.245.101.11    <none>        80:32200/TCP     35m
service/kibana                       NodePort    10.245.135.85    <none>        5601:32201/TCP   35m
```

while access Elasticsearch URL you could see response similar to below

```
~$ curl http://142.93.202.85:32200
{
  "name" : "esmaster-0",
  "cluster_name" : "escluster_dev",
  "cluster_uuid" : "KSooDxooTxuG2lbp7rYNIQ",
  "version" : {
    "number" : "7.1.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "606a173",
    "build_date" : "2019-05-16T00:43:15.323135Z",
    "build_snapshot" : false,
    "lucene_version" : "8.0.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```
```
~$ curl http://142.93.202.85:32200/_cat/nodes?v
ip           heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
10.244.2.168           23          93   5    0.22    0.29     0.44 m         *      esmaster-1
10.244.1.114           23          84   7    0.37    0.33     0.34 d         -      esdata-1
10.244.2.177           26          93   5    0.22    0.29     0.44 d         -      esdata-2
10.244.0.235           27          84   5    0.11    0.51     0.55 m         -      esmaster-2
10.244.0.155           23          84   6    0.11    0.51     0.55 d         -      esdata-0
10.244.1.140           22          84   8    0.37    0.33     0.34 m         -      esmaster-0
```