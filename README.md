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

#### Accesses
To properly run the NFS server, the container requires SCC privileged. To avoid elavated accesses being granted to the Default Storage Account, we created a NFS Service Account and granted the accesses to that Service Account only.  

We also granted the admin role to the NFS Service Account on the Storage namespace only.  

To automate the creation of the Persistent Volume and Persistent Volume Claim, we are leveraging the NFS Service Account. We had to grant the Service Account the folowing role: `system:persistent-volume-provisioner` on the storage namespace.   

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

### Improvments or tweaks

On the file [pod-demo.yaml](./pod-demo.yaml) I have used a configMap that executes a script to automate the PV creation since we need to gather the NFS SVC IP, as we cannot use NFS SVC FQDN for mounting the volumes. This is caused by a race condition, where the mounts are triggered prior to DNS resolution.  

If you would like to remove this automation and go manually, you may edit the following:  

**Step 1:**  
edit the following file: [deployment.yaml](./deployment.yaml)
Remove the following block:  
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: null
  name: system:persistent-volume-provisioner
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:persistent-volume-provisioner
subjects:
- kind: ServiceAccount
  name: nfs-server-sa
  namespace: storage
```  

**Step 2:**  
edit the following file: [pod-demo.yaml](./pod-demo.yaml)
Remove the following block:  
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: nfs-svc-ip-script
  namespace: storage
data:
  nfs-svc-finder.sh: |
    #!/bin/sh
    oc login --token=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT --insecure-skip-tls-verify
    SVC_IP=$(oc get svc nfs-service -n storage -o yaml | grep 'clusterIP: 172' | awk {'print $2'})
    echo "Generating the demo pod manifest"
    cat <<EOF > ./nfs-demo-pv-pvc.yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: nfs-pv
      namespace: storage
      labels:
        app: nfs-demo
    spec:
      capacity:
        storage: 1Gi
      accessModes:
        - ReadWriteMany
      nfs:
        server: "${SVC_IP}"
        path: "/shared"
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: nfs-pvc
      namespace: storage
    spec:
      accessModes:
        - ReadWriteMany
      storageClassName: ""
      resources:
        requests:
          storage: 1Gi
      selector:
        matchLabels:
          app: nfs-demo
    EOF
    echo "Applying the file nfs-demo-pv-pvc.yaml"
    oc create -f ./nfs-demo-pv-pvc.yaml
    echo "File Applied"
    echo "waiting until job is completed"
    oc wait --for=condition=complete --timeout=30s job/nfs-svc-ip-job
    echo "Job Completed"
---
apiVersion: batch/v1
kind: Job
metadata:
  name: nfs-svc-ip-job
  namespace: storage
spec:
  parallelism: 1    
  completions: 1    
  activeDeadlineSeconds: 1800 
  backoffLimit: 6   
  template:         
    metadata:
      name: nfs-svc-ip-job
    spec:
      serviceAccount: nfs-server-sa
      serviceAccountName: nfs-server-sa
      volumes:
      - name: script
        configMap:
          name: nfs-svc-ip-script
      containers:
      - name: nfs-svc-finder
        image: quay.io/openshift/origin-cli:4.11
        volumeMounts:
          - name: script
            mountPath: /usr/nfs-svc-finder
        command:
        - /bin/sh
        - /usr/nfs-svc-finder/nfs-svc-finder.sh
        imagePullPolicy: IfNotPresent
      restartPolicy: OnFailure
---
```

**Step 3:**  
Retrive the IP address of the NFS Service using th efollowing command:  
`oc get svc nfs-service -n storage -o yaml`  

You will need to create the PV and PVC using the following:  
```
cat <<EOF > ./nfs-demo-pv-pvc.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
    name: nfs-pv
    namespace: storage
    labels:
    app: nfs-demo
spec:
    capacity:
    storage: 1Gi
    accessModes:
    - ReadWriteMany
    nfs:
    server: "NFS_SVC_IP" # <-- Replace the value with the correct IP address of the NFS Service 
    path: "/shared"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: nfs-pvc
    namespace: storage
spec:
    accessModes:
    - ReadWriteMany
    storageClassName: ""
    resources:
    requests:
        storage: 1Gi
    selector:
    matchLabels:
        app: nfs-demo
EOF
```