# Deploy Elasticsearch as StatefulSet Pod

This directory contains Kubernetes configurations which run elasticsearch data and master pods as a [`StatefulSet`](https://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/), using storage provisioned using a [`StorageClass`](http://blog.kubernetes.io/2016/10/dynamic-provisioning-and-storage-in-kubernetes.html).

## Storage

The [`es-data.yaml`](es-data.yaml) and [`es-master.yaml`](es-master.yaml) files contain `volumeClaimTemplates` sections which request 5GB volume for each master node, data node and client node. This is plenty of space for a demonstration cluster, but will fill up quickly under moderate to heavy load. Consider modifying the disk size to your needs.

## Deploy
The root directory contains yaml files for deploying elasticsearch cluster and Kibana.

```
kubectl create -f es-namespace.yaml
```

```
kubectl create -f install/

persistentvolumeclaim/storage-es-client created
deployment.apps/es-client created
service/elasticsearch created
statefulset.apps/es-data created
service/elasticsearch-data created
deployment.apps/es-hq created
service/hq created
deployment.apps/es-kibana created
service/kibana created
statefulset.apps/es-master created
service/elasticsearch-discovery created
daemonset.extensions/fluentd created
serviceaccount/fluentd created
clusterrole.rbac.authorization.k8s.io/fluentd created
clusterrolebinding.rbac.authorization.k8s.io/fluentd created
service/fluentd created
```
This will spin up a three node Elasticsearch cluster along with a Loadbalancer service to expose the pod service to access from external.

Verify that both the pod and service were created:

```
NAME                             READY   STATUS    RESTARTS   AGE
pod/es-client-b8c45dbb7-ssn95    1/1     Running   0          10m
pod/es-data-0                    1/1     Running   0          10m
pod/es-data-1                    1/1     Running   0          10m
pod/es-data-2                    1/1     Running   0          9m45s
pod/es-hq-695677d7b4-vhs7w       1/1     Running   0          10m
pod/es-kibana-5f6f544b95-p7mtq   1/1     Running   0          10m
pod/es-master-0                  1/1     Running   0          10m
pod/es-master-1                  1/1     Running   0          10m
pod/es-master-2                  1/1     Running   0          9m31s

NAME                                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
persistentvolumeclaim/storage-es-client     Bound    pvc-38806e87-8af3-11e9-8fcb-ee29bfca3f5b   5Gi        RWO            do-block-storage   10m
persistentvolumeclaim/storage-es-data-0     Bound    pvc-395334b3-8af3-11e9-8fcb-ee29bfca3f5b   5Gi        RWO            do-block-storage   10m
persistentvolumeclaim/storage-es-data-1     Bound    pvc-4800e505-8af3-11e9-8fcb-ee29bfca3f5b   5Gi        RWO            do-block-storage   10m
persistentvolumeclaim/storage-es-data-2     Bound    pvc-5124577d-8af3-11e9-8fcb-ee29bfca3f5b   5Gi        RWO            do-block-storage   9m46s
persistentvolumeclaim/storage-es-master-0   Bound    pvc-3a7cba9f-8af3-11e9-8fcb-ee29bfca3f5b   5Gi        RWO            do-block-storage   10m
persistentvolumeclaim/storage-es-master-1   Bound    pvc-476135ad-8af3-11e9-8fcb-ee29bfca3f5b   5Gi        RWO            do-block-storage   10m
persistentvolumeclaim/storage-es-master-2   Bound    pvc-597bb49d-8af3-11e9-8fcb-ee29bfca3f5b   5Gi        RWO            do-block-storage   9m32s

NAME                              TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)          AGE
service/elasticsearch             LoadBalancer   10.245.5.133     167.99.24.69      9200:32178/TCP   10m
service/elasticsearch-data        ClusterIP      None             <none>            9300/TCP         10m
service/elasticsearch-discovery   ClusterIP      None             <none>            9300/TCP         10m
service/hq                        LoadBalancer   10.245.144.9     68.183.250.164    80:31980/TCP     10m
service/kibana                    LoadBalancer   10.245.156.111   138.197.237.111   80:30865/TCP     10m
```

while access Elasticsearch URL you could see response similar to below

```
~$ curl http://167.99.24.69:9200
{
  "name" : "es-client-b8c45dbb7-ssn95",
  "cluster_name" : "my-es",
  "cluster_uuid" : "x7VgoQ98SD-blstUq8ugqg",
  "version" : {
    "number" : "6.7.2",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "56c6e48",
    "build_date" : "2019-04-29T09:05:50.290371Z",
    "build_snapshot" : false,
    "lucene_version" : "7.7.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```
```
~$ curl curl http://167.99.24.69:9200/_cat/nodes?v
ip           heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
10.244.2.100           55          61   9    0.29    0.48     0.76 d         -      es-data-0
10.244.1.149           76          70  10    0.19    0.29     0.33 d         -      es-data-2
10.244.1.202           64          70  11    0.19    0.29     0.33 m         *      es-master-1
10.244.0.46            58          83  11    0.08    0.45     0.67 d         -      es-data-1
10.244.0.47            46          83  11    0.08    0.45     0.67 i         -      es-client-b8c45dbb7-ssn95
10.244.2.246           63          61  10    0.29    0.48     0.76 m         -      es-master-0
10.244.0.15            54          83  10    0.08    0.45     0.67 m         -      es-master-2
```
Fluentd pods are deployed in the kube-system namespace
```
kubectl get pods  -n kube-system -l k8s-app=fluentd-logging
NAME            READY   STATUS    RESTARTS   AGE
fluentd-6w9fq   1/1     Running   0          20m
fluentd-dfkcx   1/1     Running   0          20m
fluentd-pmjcf   1/1     Running   0          20m
```
Since we've deployed three node elasticluster, you can check the logs of es-master leader pod
```
kubectl logs -f es-master-0 -n elasticsearch | grep ClusterApplierService
[2019-06-09T20:16:11,774][INFO ][o.e.c.s.ClusterApplierService] [es-master-0] detected_master {es-master-1}{09wAzzn7TGCRKMdbm2dOIQ}{u-BlcBZ0SiS-PJY3G8HFHg}{10.244.1.202}{10.244.1.202:9300}{xpack.installed=true}, added {{es-master-1}{09wAzzn7TGCRKMdbm2dOIQ}{u-BlcBZ0SiS-PJY3G8HFHg}{10.244.1.202}{10.244.1.202:9300}{xpack.installed=true},}, reason: apply cluster state (from master [master {es-master-1}{09wAzzn7TGCRKMdbm2dOIQ}{u-BlcBZ0SiS-PJY3G8HFHg}{10.244.1.202}{10.244.1.202:9300}{xpack.installed=true} committed version [1]])
[2019-06-09T20:16:22,846][INFO ][o.e.c.s.ClusterApplierService] [es-master-0] added {{es-data-0}{cdBxMJBFSWq7wtD0AFkKlw}{am2X_wL4QB2-Hj2EodHH6Q}{10.244.2.100}{10.244.2.100:9300}{xpack.installed=true},}, reason: apply cluster state (from master [master {es-master-1}{09wAzzn7TGCRKMdbm2dOIQ}{u-BlcBZ0SiS-PJY3G8HFHg}{10.244.1.202}{10.244.1.202:9300}{xpack.installed=true} committed version [15]])
[2019-06-09T20:16:33,223][INFO ][o.e.c.s.ClusterApplierService] [es-master-0] added {{es-data-1}{VnhsBYoyR0mm0ZbmNqXPdQ}{dvhRyIykRTqtYily9MnabQ}{10.244.0.46}{10.244.0.46:9300}{xpack.installed=true},}, reason: apply cluster state (from master [master {es-master-1}{09wAzzn7TGCRKMdbm2dOIQ}{u-BlcBZ0SiS-PJY3G8HFHg}{10.244.1.202}{10.244.1.202:9300}{xpack.installed=true} committed version [16]])
[2019-06-09T20:16:37,912][INFO ][o.e.c.s.ClusterApplierService] [es-master-0] added {{es-client-b8c45dbb7-ssn95}{PAorC2GXSCeVq9nM6INdGg}{Co-uN-QbSKyr_cKoSNKxlg}{10.244.0.47}{10.244.0.47:9300}{xpack.installed=true},}, reason: apply cluster state (from master [master {es-master-1}{09wAzzn7TGCRKMdbm2dOIQ}{u-BlcBZ0SiS-PJY3G8HFHg}{10.244.1.202}{10.244.1.202:9300}{xpack.installed=true} committed version [17]])
[2019-06-09T20:17:00,236][INFO ][o.e.c.s.ClusterApplierService] [es-master-0] added {{es-master-2}{g1CYPitaQCy2VUYmr5TFmg}{pl1nE8yER7-YWmFrW-Zyog}{10.244.0.15}{10.244.0.15:9300}{xpack.installed=true},}, reason: apply cluster state (from master [master {es-master-1}{09wAzzn7TGCRKMdbm2dOIQ}{u-BlcBZ0SiS-PJY3G8HFHg}{10.244.1.202}{10.244.1.202:9300}{xpack.installed=true} committed version [22]])
[2019-06-09T20:17:08,973][INFO ][o.e.c.s.ClusterApplierService] [es-master-0] added {{es-data-2}{ZuyQ4kNKTC6kwIOrfTJTkQ}{mlvUfqe5TGyWoAU9l56CjA}{10.244.1.149}{10.244.1.149:9300}{xpack.installed=true},}, reason: apply cluster state (from master [master {es-master-1}{09wAzzn7TGCRKMdbm2dOIQ}{u-BlcBZ0SiS-PJY3G8HFHg}{10.244.1.202}{10.244.1.202:9300}{xpack.installed=true} committed version [23]])
```
To check the logs collected by Fluentd in Kibana dashboard, configure index patterns 

![EFK](/image/kibana-homepage.png?raw=true)
Click the "Create index pattern" button. Select the new Logstash index that is generated by the Fluentd DaemonSet. Click "Next step"
![EFK](/image/kibana-management.png?raw=true)
![EFK](/image/Create-index.png?raw=true)
![EFK](/image/Define-index.png?raw=true)
Set the "Time Filter field name" to "@timestamp". Then, click "Create index pattern".
Click "Discover" to view your pod logs.
![EFK](/image/Kibana-Discover.png?raw=true)

ElasticHQ helps you in managing and monitoring Elasticsearch clusters. You can access gui using LoadBalancer IP.

![EFK](/image/ElasticHQ.png?raw=true)