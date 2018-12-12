# Resizing(increasing only not shrinking) Persistent Volumes(PV) using Kubernetes v1.10

I got the cassandra clusters' volumes got full 100%. 
Cassandra data volume/storage is AWS EBS with its size 100GB

1. Identify the cassandra PVs attached to k8s node to identify AWS EBS volume
2. Increase Modify AWS EBS in AWS console or awscli up to 500GBs.
3. You can't change the storage size 100Gi to 500Gi in k8s cassandra statefulset because the k8s 1.10 doesn't support it.
4. Update the size of cassandra PVs up to 500Gi
5. delete cassandra pods and identify k8s nodes which  are attached to. Then, run lsblk command again to see 

6. run the resize2fs cmd to resize the filesystem
7. Run the nodestatus cmd in cassandra nodes to check the cassandra cluster status.

## command output:

1.
```
kubectl get pv  |grep cassandra
pvc-027deb04-d551-11e8-bb7d-0af134bb8194   100Gi      RWO            Delete           Bound     default/data-cassandra-cassandra-2           gp2                      51d
pvc-63fb5eda-d550-11e8-bb7d-0af134bb8194   100Gi      RWO            Delete           Bound     default/data-cassandra-cassandra-0           gp2                      51d
pvc-b2eb9873-d550-11e8-bb7d-0af134bb8194   100Gi      RWO            Delete           Bound     default/data-cassandra-cassandra-1           gp2                      51d
```
3.
```
- metadata:
      creationTimestamp: null
      labels:
        app: cassandra-cassandra
        chart: cassandra-0.2.6
        heritage: Tiller
        release: cassandra
      name: data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
```

4.
```
 kubectl patch pv pvc-027deb04-d551-11e8-bb7d-0af134bb8194  -p '{"spec":{"capacity":{"storage":"500Gi"}}}'

 kubectl patch pv pvc-63fb5eda-d550-11e8-bb7d-0af134bb8194  -p '{"spec":{"capacity":{"storage":"500Gi"}}}'

 kubectl patch pv pvc-b2eb9873-d550-11e8-bb7d-0af134bb8194  -p '{"spec":{"capacity":{"storage":"500Gi"}}}'
```
5. 
```
kubectl delete pod -l app=cassandra-cassandra
pod "cassandra-cassandra-0" deleted
pod "cassandra-cassandra-1" deleted
pod "cassandra-cassandra-2" deleted
```
6.
```
admin@ip-172-31-37-174:~$ lsblk 
NAME    MAJ:MIN   RM  SIZE RO TYPE MOUNTPOINT
xvda    202:0      0  128G  0 disk 
└─xvda1 202:1      0  128G  0 part /
xvdca   202:19968  0  500G  0 disk /var/lib/kubelet/pods/95ecdada-fde3-11e8-b01e-0629aa53ac34/volumes/kubernetes.io~aws-ebs/pvc

admin@ip-172-31-37-174:~$ sudo resize2fs /dev/xvdca
--> repeat for other volumes



kubectl get pod -l app=cassandra-cassandra -o wide
NAME                    READY     STATUS    RESTARTS   AGE       IP              NODE
cassandra-cassandra-0   1/1       Running   0          4m        100.104.74.19   ip-172-31-59-21.ap-northeast-1.compute.internal
cassandra-cassandra-1   0/1       Running   0          2m        100.99.16.136   ip-172-31-72-112.ap-northeast-1.compute.internal
```

7.
``` 
kubectl exec -it --namespace default $(kubectl get pods --namespace default -l app=cassandra-cassandra -o jsonpath='{.items[0].metadata.name}') nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address         Load       Tokens       Owns (effective)  Host ID                               Rack
UN  100.99.16.132   58.47 GiB  256          100.0%            bcd22662-f313-4356-9c09-d0601f70b864  rac1
UN  100.104.74.20   58.34 GiB  256          100.0%            97b048ff-24ca-4cd8-a374-1e701d73917c  rac1
UN  100.100.104.26  57.58 GiB  256          100.0%            14ca7595-7b4e-4421-93d3-2658000bfdee  rac1
```