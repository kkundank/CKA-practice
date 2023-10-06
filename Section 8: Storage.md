Docker storage: 
2 Processes: 
- Storage drivers
- Volume drivers 

- Docker stores stores data on the local file system
  - file system: 
    - /var/lib/docker
      - aufs
      - containers
      - image
      - volumes
      
- Dockers layered architecture: Docker file - image
  - Layer 1 - Base Ubuntu Layer
  - Layer 2 - Changes in apt packages
  - Layer 3 - changes in pip packages
  - Layer 4 - source code 
  - Layer 5 - Update Entrypoint
  
  Dockerfile 2
  - Layer 1 - Base Ubuntu Layer <<<<<<<<<------------------ Docker uses previous layers from cache  
  - Layer 2 - Changes in apt packages <<<<<<<<<<----------- Docker uses previous layers from cache 
  - Layer 3 - changes in pip packages <<<<<<<<<<----------- Docker uses previous layers from cache 
  - Layer 4 - source code 
  - Layer 5 - Update Entrypoint
  
  - Layer 1 - Base Ubuntu Layer
  - Layer 2 - Changes in apt packages
  - Layer 3 - changes in pip packages
  - Layer 4 - source code 
  - Layer 5 - Update Entrypoint
  - Layer 6 - Container Layer <<<<<<<<<<<<----- When you run docker image, it creates new layer used to store data created
                                                by the container. Life if this layer, if container is destroyed so is this. 

- Layer 6: =  Read Write
- Image Layers: = Read Only 

Example:
app.py in Image layer, docker will automatically create a copy of this file in the container layer, than modifying different level 
version of the file in the read/write layer. 

- What happens if you get rid of the container?
  - All the data stored in the container layer, also gets delete, change made to app.py will also get removed.
  
Persist volume / peserve:
- Create Docker volume, 'docker volume create data_volume'
- Creates a new folder called data_volume
  - /var/lib/dockeer
    - volumes
    - data_volume
- Than running the docker container, can run this command to  store volume
  - docker run -v data_volume :/var/lib/mysql mysql 
  - this will create new container and mount the data volume to the data folder
  - even if the data is destroyed, the container is still active
  
  2 types of mounting:
  - Bind mounting: Mounts from any location on the docker host 
  - Volume Mounting: mounts a volume from the volumes directorory. 
  
Who is responsible for all this, maintaing the layered architecture, creating writable layer, 
moving layers across layers copy and write etc is:
- STORAGE DRIVERS

- Docker will choose the best storage driver available automatically based on the operating system. 
 
 Storage drivers: help manage storage on containers and images
 volumes: persist storage must create storage
          - local is the default volume driver plugin
          
- When you run a docker container, you can choose a specific volume driver such as rexray/ebs to provision
  a volume for amazon ebs. Will create a container and attach volume from the aws cloud, data saved in the cloud.
  
  Container storage interface (CSI). 
  - In the past k8s just used docker for CR, new runtimes come such as RKT/CRI-0 etc. 
  - Container runtime interface is a standard, how k8s communites with docker for example. 
  - CSI was created to support many other 3rd party vendors such as Portworkx etc
    - can write your own drivers to work with kubernetes
   
  CSI
  
  RPC - When a pod is created and requires a volume, k8s should call the create volume RPC and -
        pass detail such as volume name, storage driver should implement this RPC and provision new volume
        on the storag array 
  Create volume
  Delete Volume
  ControllerPublishVolume
 
Volumes:
- Docker containers are only meant to last for a short period of time
- attach volume to pod, data generated to pod, strored in volume even after pod is deleted
- use volumemounts field in containers to mount volume name
  - VolumeMounts:
  - mountPath: /opt
    name: data-volume
    
 Volumes:
 - name: data-volume
   hostPath:
      path: /data
      type: Directory

Persistent Volumes:
- when you have a large cluster for each pod/container, you have to usually
  configure volume for each, which can get messy. Everytime a change is to be made, 
  user would have to make changes to each pod manifest. 
- administrator curve out a large ammount of storage, users than can curve out pieces
  from it, as required. 
- a PVC, is a cluster wide pool of storage configured by admin, used by users deploying application on the clusters.

apiVersion: v1
kind: PersistentVolume
metadata:
   name: pv-vol
spec:
  accessModes:
     - ReadWriteOnce
  capacity:
     - storage: 1GI
  hostPath:
    path: /tmp/data
    
- kubectl get persistentvolume
  
Persistence Volume Claims:
- Admin creates Persistent Volumes
- Users creates Persistent Volumes Claims
- Kubernetes binds the PV to PVC based on the request and properties
- kubernetes tries to find PV that has sufficient capacity has requested by the claim
- if there are multiple matches for a volume, you can use labels and selectors to bind to correct volume.
- if no Volumes avaiable, the PVC will remain in a pending state. 

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
   name: myclaim
spec:
  accessModes:
     - ReadWriteOnce
     
  resources:
     request:
       storage: 500Mi
       
- when the PVC is created, k8s looks at the previous PV created, if the 
  accessModules match 
  
- can delete the pvc, however  what happens to the underlying storag claim when the 
  PVC is deleted, you can choose what needs to happen to the volume, by default
  it is chosen to be 'retain' 

- recycle / delete

Practice test: Persistent volume claim


1. We have deployed a POD. Inspect the POD and wait for it to start running. 
2. The application stores logs at location /log/app.log. View the logs. 
   You can exec in to the container and open the file:
   kubectl exec webapp -- cat /log/app.log
3. If the POD was to get deleted now, would you be able to view these logs.
4. Configure a volume to store these logs at /var/log/webapp on the host.
   Use the spec provided below. 
   Use the command kubectl get po webapp -o yaml > webapp.yaml and add the given properties under the spec.volumes and spec.containers.volumeMounts.
   OR
   Use the command kubectl run to create a new pod and use the flag --dry-run=client -o yaml to generate the manifest file.
   In the manifest file add spec.volumes and spec.containers.volumeMounts property.
   kubectl replace -f webapp.yaml --force
5. Create a Persistent Volume with the given specification.
Volume Name: pv-log
Storage: 100Mi
Access Modes: ReadWriteMany
Host Path: /pv/log
Reclaim Policy: Retain
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: pv-log
  spec:
     persistentVolumeReclaimPolicy: Retain
     accessModes:
        - ReadWriteMany
    capacity:
        storage: 100Mi
    hostPath:
      path: /pv/log 
      
6. Let us claim some of that storage for our application. Create a Persistent Volume Claim with the given specification.
Volume Name: claim-log-1
Storage Request: 50Mi
Access Modes: ReadWriteOnce

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: claim-log-1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Mi
      
7. What is the state of the Persistent Volume Claim? Pending
   
8. What is the state of the Persistent Volume? Available
9. Why is the claim not bound to the available Persistent Volume? Access Mode Mismatch
10. Update the Access Mode on the claim to bind it to the PV.
    Delete and recreate the claim-log-1.
    kubectl delete pvc claim-log-1
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: claim-log-1
    spec:
      accessModes:
         - ReadWriteMany
      resources:
       requests:
      storage: 50Mi

11. You requested for 50Mi, how much capacity is now available to the PVC? 100Mi
12. Update the webapp pod to use the persistent volume claim as its storage.
    Replace hostPath configured earlier with the newly created PersistentVolumeClaim.
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: event-simulator
    image: kodekloud/event-simulator
    env:
    - name: LOG_HANDLERS
      value: file
    volumeMounts:
    - mountPath: /log
      name: log-volume

  volumes:
  - name: log-volume
    persistentVolumeClaim:
      claimName: claim-log-1

13. What is the Reclaim Policy set on the Persistent Volume pv-log? Run the command: kubectl get pv and look under the Reclaim Policy column.
14. What would happen to the PV if the PVC was destroyed? The PV is not deleted but not available
15. Try deleting the PVC and notice what happens.
If the command hangs, you can use CTRL + C to get back to the bash prompt OR check the status of the pvc from another terminal
Use the command kubectl delete to delete the pvc claim-log-1 and kubectl get to list the resource.
= The PV is stuck in terminating state
16. Why is the PVC stuck in Terminating state? The PVC is being used by the pod
17. 
18. What is the state of the PVC now? Deleted
19. What is the state of the Persistent Volume now? Released

Storage Class:

- Create PVs and PVCs to claim that storage.
- And use that PVC put claimname in pod definition.
- static provisioning is having to create disk prior on google cloud for example to creating.
  everytime application requires storage you usually have to manually provision the disk and then manually create the persistent volume definition file. 
- dynamic provision - volume get provisions automatically before application gets provisioned  
  storage class - define a provisioner such as azure storage, allows you to automatically/dynamically provision storage on azure cloud and attach
  - to pod when a claim is made. 
- create storage class manifest
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata: kubernetes.io/gc-pd 
- now have a storage class and no longer require a PV since we have a storage class which provisions storage
- PV no longer required since storageclass will automatically create it when storageclass is created
- For PVC now to use Storageclass, it is now simply mentioned here:
  kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: claim-log-1
    spec:
      accessModes:
         - ReadWriteMany
      storageClass: google-storage
      resources:
       requests:
      storage: 50Mi
 - StorageClass dynmaically and automatically provisions storage on pod as soon as claim has been made. 
 - next time a PVC is created the storage class associated with it, uses the define provisioners to provison new disk with required size on Google cloud.
 - still creates PV but its not manually created anymore, it is dynamically 
 - Creates PV, then binds PVC to that volume
 
 Practice test: Storage Class
 
1. How many StorageClasses exist in the cluster right now? = 1
  kubectl get storageclasses --all-namespaces
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  11m

2. How about now? How many Storage Classes exist in the cluster? = 3
controlplane ~ ✖ kubectl get storageclasses --all-namespaces
NAME                        PROVISIONER                     RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)        rancher.io/local-path           Delete          WaitForFirstConsumer   false                  12m
local-storage               kubernetes.io/no-provisioner    Delete          WaitForFirstConsumer   false                  10s
portworx-io-priority-high   kubernetes.io/portworx-volume   Delete          Immediate              false                  10s

3. What is the name of the Storage Class that does not support dynamic volume provisioning? Look for the storage class name that uses no-provisioner
controlplane ~ ➜  kubectl describe storageclasses local-storage
Name:            local-storage
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"local-storage"},"provisioner":"kubernetes.io/no-provisioner","volumeBindingMode":"WaitForFirstConsumer"}

Provisioner:           kubernetes.io/no-provisioner
Parameters:            <none>
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>

4. What is the Volume Binding Mode used for this storage class (the one identified in the previous question)? WaitForFirstConsumer

5. What is the Provisioner used for the storage class called portworx-io-priority-high? Provisioner: kubernetes.io/portworx-volume

6. Is there a PersistentVolumeClaim that is consuming the PersistentVolume called local-pv? No

7. Let's fix that. Create a new PersistentVolumeClaim by the name of local-pvc that should bind to the volume local-pv.

---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: local-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 500Mi
   

8.  What is the status of the newly created Persistent Volume Claim?
    controlplane ~ ➜  kubectl get persistentvolumeclaims 
     NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS    AGE
     local-pvc   Pending                                      local-storage   6m58s

9.  Why is the PVC in a pending state despite making a valid request to claim the volume called local-pv?
    Pod consuming the volume is not scheduled. 
    
10. The Storage Class called local-storage makes use of VolumeBindingMode set to WaitForFirstConsumer. 
    This will delay the binding and provisioning of a PersistentVolume until a Pod using the PersistentVolumeClaim is created.
    
11. Create a new pod called nginx with the image nginx:alpine. The Pod should make use of the PVC local-pvc and mount the volume at the path /var/www/html.
    The PV local-pv should in a bound state.
    
12. What is the status of the local-pvc Persistent Volume Claim now? Bound









