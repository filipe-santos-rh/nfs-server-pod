# nfs-server-pod

This repo is currently a work in progress to generate an NFS server pod that will leverage the default StorageClass within the cluster to allow RWX PVC.  

### General info
Within this repository the following 3 use cases will be tested:  
*Case 1:*  
- Create an NFS server within the storage namespace  
- Create a Pod that will consume an NFS mount directly  

*Case 2:*
- Create a PV that will reference the NFS server mounts
- Create a PVC that will bing to the PV
- Create a POD that will consume the PVC  

*Case 3:*  
- Create a second Pod that will consume the same `pvc` previously mounted on the demo pod.  

### How to

**Scenario 1**  

*Step 1:*
*The first step is to create the NFS server using the [deployment.yaml](./deployment.yaml) file*  
`oc create -f deployment.yaml`  

*Confirm that the pods are running and no events are found*
`oc get events -w -n storage`  

Once we confirmed that the pods are running properly, we might move to Step 2.  

*Step 2:*  
We will be creating a folder that conatiners will be able to use as a NFS Share folder:  

```
POD=$(oc get pods -n storage -o name --selector='app=nfs-server')
oc -n storage exec $POD -- mkdir /exports/shared 
```

*You may confirm that the folder was created by running the following command:  
`oc -n storage exec $POD -- ls /exports/`

**Scenario 2**  

*Step 1:*  
We will now apply the [pod-demo.yaml](./pod-demo.yaml). This will generate the pod that will be mounting the NFS server share directly into the pod.  
`oc create -f pod-demo.yaml`

**Scenario 3**  

Within the deployment of [pod-demo.yaml](./pod-demo.yaml), we deployed 2 application pods "nfs-demo-1 and nfs-demo-2".  
Both pods will be mounting the same PVC, since the access mode is set to RWX.  
```
NAME                                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/nfs-pvc         Bound    nfs-pv                                     1Gi        RWX                           4m30s
```


### Outcome
You should now have the following resources within the namespace `storage` the following:  
```
NAME                              READY   STATUS      RESTARTS   AGE
pod/nfs-demo                      1/1     Running     0          3m29s
pod/nfs-server-6bf9f4f7f6-d485t   1/1     Running     0          5m26s
pod/nfs-svc-ip-job-ts9p9          0/1     Completed   0          3m29s

NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/nfs-service   ClusterIP   172.30.109.116   <none>        2049/TCP,20048/TCP,111/TCP   5m26s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-server   1/1     1            1           5m26s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/nfs-server-6bf9f4f7f6   1         1         1       5m26s

NAME                       COMPLETIONS   DURATION   AGE
job.batch/nfs-svc-ip-job   1/1           35s        3m30s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                       STORAGECLASS   REASON   AGE
persistentvolume/nfs-pv                                     1Gi        RWX            Retain           Bound    storage/nfs-pvc                                                     4m29s
persistentvolume/pvc-6f4bbfea-e7ae-4bec-8ebe-e8b421a4f740   8Gi        RWO            Delete           Bound    storage/nfs-srv-claim                       gp2                     6m23s

NAME                                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/nfs-pvc         Bound    nfs-pv                                     1Gi        RWX                           4m30s
persistentvolumeclaim/nfs-srv-claim   Bound    pvc-6f4bbfea-e7ae-4bec-8ebe-e8b421a4f740   8Gi        RWO            gp2            6m30s
```

