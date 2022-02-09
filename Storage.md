# Storage
## Volumes
![volumes & mounts](https://i.imgur.com/4VcI9Dd.png)
### eg using aws EBS
``` yaml
volumes:
- name: data-volume
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ex4
```
## Persistent volumes
is a cluster wide pool of storage volumes configured by an administrator to be used by users deploying applications on the cluster.
![persistent volumes](https://i.imgur.com/UHcL5RJ.png) 

## Persistent volume claims
An admin creates a set of persistent volumes and a user creates persistent volume clains to use the storage.
``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: myclaim
spec:
 accessModes:
   - ReadWriteOnce
 resources:
   requests:
     storage: 500Mi
```

create by: `k create -f pvc-definition.yaml`

view by: `k get persistentvolumeclaim`

delete PVCs by: `k delete persistentvolumeclaim myclaim`
by default the persistent volume will remain until it is manually deleted by the admin.

`persistentVolumeReclaimPolicy: Recycle`: the data in the data volume will be scrubbed before making it available to other claims.
## Using PVCs in PODs
``` yaml
 apiVersion: v1
 kind: Pod
 metadata:
   name: mypod
 spec:
   containers:
     - name: myfrontend
       image: nginx
       volumeMounts:
       - mountPath: "/var/www/html"
         name: mypd
   volumes:
     - name: mypd
       persistentVolumeClaim:
         claimName: myclaim
```
## Storage Class
With storage class, you can define a provisional such as google storage that can automatcially provision storage on google cloud an attach that to pods when a claim is made, that's called dynamic provisioning of volumes.
# sc-definition.yaml
``` yaml
apiVersion: storage.k8s.io/v1
kind: storageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd
```
