Manually scheduling:
     
- Kube Scheduler, schedule pods to nodes
- every pod has a field called node name which is not set when you create pod menifest file, k8s adds this automatically.
- Kube schedule looks at all the pods which do not have the 'node name' property set.
- If a pod does not have a node name set in the property, it schedule pods to a node. 
- if there is no scheduler you would need to manually assign pods to node yourself
     - so in the pod manifest, you can simply add the node name field and assign the pod to the node. 
- You can only specify the node name in creation time and not once created. 
- kubernetes wont allow you to modify the node name property of the pod after creation 
- Instead you can create a binding object and send post request to pod api. 
- if there is no scheduler, the pods will continue to be in a pending state. 

  
Manually schedule the pod on node01.
Delete and recreate the POD if necessary.
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  -  image: nginx
     name: nginx
  nodeName: node01 ------------------Adding this allows you to manually specify the pod to a node, however do need to delete the pod and create file again.
    
Now schedule the same pod on the controlplane node.
Delete and recreate the POD if necessary.
  
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  -  image: nginx
     name: nginx
  nodeName: controlplane
~                                                                                                                                                               
~               
     
Labels and Selectors:
     
- Are used to identify where objects belong to
- for each object attach labels as per your needs i.e app / function
- specify condition to filer out objects
- In pod definition file, you can specify under metadata section called labels and add as many labels as you like. 
- once pod is created you can select pod with specific labels using this command kubectl get pods --selector app=App1
- For example to create a replicaset consisting of 3 different pods you can labels the pod definition and use label the pod definition - 
  and group the pods by adding the spec.
- Need to be aware that there are 2 sections with labels
     - the top part where it specifies the label, is the label of the replicaset, the labels under the template section is the labels of the pods. 

  apiVersion: v1 
  kind: ReplicationSet
  metadata: 
    name: myapp-rc
    labels:
        app: myapp
        type: front-end
    annotations:
         buildversion: 1.34 -------------------------------Annotations can provide input and information. 
  spec:
  template:
    metadata:
      name: myapp-pod
      labels:
          app: myapp
          type: front-end
  spec:
    containers:
    - name: nginx-containers
      image: nginx

- In order the replicaset to the pod, we configure the selector field to match the labels defined on the pod. so the matchlabels label need to match the label on the metadata. 
- Annotation can be used to describe or provide information 

1. We have deployed a number of PODs. They are labelled with tier, env and bu. How many PODs exist in the dev environment?
Use selectors to filter the output = kubectl get pods --selector env=dev


2. How many PODs are in the finance business unit (bu)? kubectl get pods --selector bu==finance
    
3. How many objects are in the prod environment including PODs, ReplicaSets and any other objects? kubectl get all --selector env=prod

4. Identify the POD which is part of the prod environment, the finance BU and of frontend tier? kubectl get pods --selector env=prod --selector bu=finance --selector tier=frontend 

5. A ReplicaSet definition file is given replicaset-definition-1.yaml. Try to create the replicaset. There is an issue with the file. Try to fix it.
Need to set the pod label to frontend

---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
   name: replicaset-1
spec:
   replicas: 2
   selector:
      matchLabels:
        tier: frontend
   template:
     metadata:
       labels:
        tier: frontend
     spec:
       containers:
       - name: nginx
         image: nginx 




Taints and Tolerations:
  
are used to set restrictions on what pods can be sceduled on nodes. 
     
Taints: Prevents pods joining the node
     
Tolerations: attracts the pod to the node, so enforces that pod to that node. 
     
- if you have pods A B C D and Nodes 1 2 3
     - Placing a Taint to Node 1 called Taint =blue
     - None of the pods can tolerate a taint so they be placed on Node 1 
     - If we place toleration on Pod D to tolerate to go to Node 1 it works, the kube scheduler does this. #
     - Taints are set on Nodes and tolerations are set on Pods 

command:
    - Kubectl taint nodes node-name key=value:taint-effect
    - 3 types of options
        - NoSchedule = Pods not be scheduled on the node
        - PreferNoSchedule = the system will try to not schedule a pod to a node but this isnt guranteed
        - NoExecute = new pods will not be scheduled on the node and existing pods will be evivted if they do not tolerate the taint, - 
          may have been scheduled before taint was applied on the node.  
    - Example configuration:
           - kubectl taint nodes node1 app=blue:NoSchedule 

    - Toleration - Pods 
           - kubectl taint nodes node1 node1 app= blue:NoSchedule
---
apiVersion:
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  -  image: nginx
     name: nginx

toleration
- key: "app"
  operator: "Equal"
  value: "blue"
  effect: "NoSchedule"

- If you have multiple nodes, and have taint and toleration on pod and taint on node, wheras if the other nodes dont have anything on them as in no taint or no toleration -
     on other pods, the pod with the pod with toleration doesnt even have to be selected to go to pod with the correct taint, if it happens to land on another node it can go there - 
     the taint chooses what it accepts and what it doesnt, it doesnt restrict pod joining other nodes with no restrictions.   
          
- Taint is set on the master node automatically to prevent any pods being set on the Masternode. 
          to see this taint run , kubectl describe node kubemaster | grep Taint


 - 3. Create a taint on node01 with key of spray, value of mortein and effect of NoSchedule = ubectl taint node node01 spray=mortein:NoSchedule
 - 4. Create a new pod with the nginx image and pod name as mosquito. = kubectl run mosquito --image=nginx
 - 10 Remove the taint on controlplane, which currently has the taint effect of NoSchedule = kubectl edit node controlplane remove taint
             - can also do this, kubectl taint node master node-role.kubernetes.io/master:NoSchedule- (the key here is the minus in the end which removed the taint)
 - 7. Create another pod named bee with the nginx image, which has a toleration set to the taint mortein = 
      
---
apiVersion: v1
kind: Pod
metadata:
  name: bee
spec:
  containers:
  - image: nginx
    name: bee
  tolerations:
  - key: spray
    value: mortein
    effect: NoSchedule
    operator: Equal

- Taints can only be configured on a node and it prevents pods joining node
- toleration can only be configuered on a pod and it grants a policy allowing you to join a node 


NodeSelector:

- for example you have 3 nodes, Large which has higher CPU/Process, followed by medium and then small. 
     - However any pods can go to any nodes regardless if its over utilised or not. 
          
- to prevent the above from happening you can set limitations on the pods so they only run on particular nodes. 

          ---
apiVersion: v1
kind: Pod
metadata:
  name: bee
spec:
  containers:
  - image: nginx
    name: bee
  
  nodeselector:
    size: Large ------------------------------This is now included in the Pod definition yaml, this is the label which is on the node. 
       
- Must Label the node first. 
       - kubectl label nodes node-1 size=Large
- Using nodeselector has limitations, you can only select 1 Label and does not allow you to using conditional statements such as Large and Small and not Medium.
     - This is where nodeaffinity comes in. 


Node Affinity:
  
- Assigning specific pods to nodes 

          ---
apiVersion: v1
kind: Pod
metadata:
  name: bee
spec:
  containers:
  - image: nginx
    name: bee
  
 affinity:
  nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - matchExpressions:
            - key: size
              operator: In
              values: 
              - Large

can even do this:
     
          - matchExpressions:
            - key: size
              operator: NotIn
              values: 
              - Small

- what is someone changes the label in the future? will the pod stay on the node? this is define by underneath the nodeaffinity section:
     Node Affinity types:
       - requiredDuringSchedulingIgnoredDuringExecution: ------- Scheduling pod to node is mandated, the ignore during execution part states that even if labels are removed
                                                                 it does not matter, the pod will still be scheduled on this node until it removed. 
     
       - PreferredDuringSchedulingIgnoredDuringExecution: ------ Scheduling pod to node is not mandated if no Label exist. the ignore during execution part states that even if labels are removed
                                                                 it does not matter, the pod will still be scheduled on this node until it removed
     
       - PreferredDuringSchedulingrequiredDuringExecution: ------ Scheduling pod to node is not mandated if no Label exist. the required during execution part states that even if labels are removed
                                                                 it matters, the pods whos label does not match and will evict pods that does not meet the rules. e.g if label Large is removed from rules. 
     
       - Planned:
          - requiredDuringSchedulingRequiredDuringExecution:


3. Apply a label color=blue to node n code01 = kubectl label nodes node01 color=blue
4. Create a new deployment named blue with the nginx image and 3 replicas. = kubectl create deployment blue --image=nginx --replicas=3           
6. Set Node Affinity to the deployment to place the pods on node01 only: add after pod spec under deploment:
           affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: color
                operator: In
                values:
                - blue
7. Which nodes are the pods placed on now?: node01

8. Create a new deployment named red with the nginx image and 2 replicas, and ensure it gets placed on the controlplane node only.
   Use the label - node-role.kubernetes.io/master - set on the controlplane node. you should create the file without actually running the deployment though by doing this command,
     kubectl create deployment red --image=red --dry=run -o yaml > red.yaml

     affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/master
                operator: Exists

Taints and Toleration Vs Node Affinity:
     
- If you have 
     - Blue, Red, Green and 2 other Nodes. 
          - Apply taint marking each node with their colors i.e. Node 1 with Blue
          - set tolerations on the pods to tolerate only certain nodes 
     - However taints and toleration will not provide gurantee that the pods will go to specific nodes listed taints. 
          
 Now using NodeAffinity:
          
      - Label the nodes the same
      - set node selectors on the pods to tie pods to nodes. 
      - However there is always a chance of other pods going to nodes which may not be desired. 
      - Can use taints and toleration and also Nodeaffinity together:
            - Using Taints on Nodes to only accept certain pods
            - using toleration to only tolerate certain nodes
            - and using node affinity to prevent other pods being placed on the nodes. 

                 
Resource Requirements and Limits:
                 
- each node has a set of cpu, disk resources, memory.
- whenever a pod is set on the node, it consumed the nodes resources.
- kube scheduler schedules pods to nodes. 
                 - it takes into consideration the resources required and available on the nodes. 
- for example, if the scheduler, schedules a pod on node 2, if the node has no resources the scheduler avoids on that node and does on the node with resources. 
- if nothing is avaiable or no sufficient nodes to schedule node onto, then it will state pod as pending state.
- CPU / MEM / DISK 
     - for example pod requires 0.5 CPU and 256 Mi CPU, this is called resource request. 
                 
apiVersion: v1
kind: Pod
metadata:
  name: bee
spec:
  containers:
  - image: nginx
    name: bee   
    ports: 
     - containerPort: 8080
    resources:
       requests:
         memory: '1Gi"
         cpu: 1

- 1 count of CPU means, 0.1 = 100m, 1m = 1 vCPU, can request higher number of CPU
- Memory 256Mi or 268 etc, 1G

- Lets looks at Container running on a node, a docker container has no limit resources it can consume on the nodes, can potentially sufficate nodes/other containers. 
- By default k8s sets a limit of 1 Vcpu on container, so if you dont specify it will do this same as memory (512mb) on containers. 
     - Limits are set for each container on the pod etc. 
     - can change these limits on 

In the previous lecture, I said -
"When a pod is created the containers are assigned a default CPU request of .5 and memory of 256Mi". 
 For the POD to pick up those defaults you must have first set those as default values for request and limit by creating a LimitRange in that namespace.  
    
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
                 
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 0.5
    type: Container                 
                 
  Editing an existing pod:
       
Edit a POD
Remember, you CANNOT edit specifications of an existing POD other than the below.

spec.containers[*].image
spec.initContainers[*].image
spec.activeDeadlineSeconds
spec.tolerations

For example you cannot edit the environment variables, service accounts, resource limits (all of which we will discuss later) of a running pod.
But if you really want to, you have 2 options:
1. Run the kubectl edit pod <pod name> command.  This will open the pod specification in an editor (vi editor). 
Then edit the required properties. When you try to save it, you will be denied. This is because you are attempting to edit a field on the pod that is not editable.
                 
Scheduling, Test Resource Limits:
   
1. A pod called rabbit is deployed. Identify the CPU requirements set on the Pod = 1 (Describe command)
3. Another pod called elephant has been deployed in the default namespace. It fails to get to a running state. Inspect this pod and identify the Reason why it is not running.
     - What is OOM Killed (exit code 137) The OOM Killed error, also indicated by exit code 137, means that a container or pod was terminated because they used more memory than allowed. 
       OOM stands for “Out Of Memory”. Kubernetes allows pods to limit the resources their containers are allowed to utilize on the host machine.
5.  The elephant pod runs a process that consume 15Mi of memory. Increase the limit of the elephant pod to 20Mi.

---
apiVersion: v1
kind: Pod
metadata:
  name: elephant
  namespace: default
spec:
  containers:
  - args:
    - --vm
    - "1"
    - --vm-bytes
    - 15M
    - --vm-hang
    - "1"
    command:
    - stress
    image: polinux/stress
    name: mem-stress
    resources:
      limits:
        memory: 20Mi
      requests:
        memory: 5Mi

- what you can do is instead of deleting and re-creating new yaml file for elephant, you can simply re-create based on the existing pod.
     - kubectl get pods elephant -o yaml > pod.yaml
     - edit the yaml file from here 
                
Daemon Sets:
     
- Daemonsets are like replicasets, help deploy multiple instances of pods however 1 on each node.
- when a new node is added to the cluster, the daemonset is automatically assigned to the given node. 
- when a node is removed so is that daeomonet pod. 
- daemonset ensures copy of pod is presence on all cluster.
- for example you want to deploty a monitoring agents and log viewer, 
        -  dont have to worry about adding or removing daemonset will take care of it. 
                 
  
 apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # this toleration is to have the daemonset runnable on master nodes
      # remove it if your masters can't run pods
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
                 
- Daemonset scheduled pod to node using same as previous lessons, using kubescheduler/nodeaffinity. 
     
 6. Deploy a DaemonSet for FluentD Logging.    
Name: elasticsearch
Namespace: kube-system
Image: k8s.gcr.io/fluentd-elasticsearch:1.20

An easy way to create a DaemonSet is to first generate a YAML file for a Deployment with the command 
kubectl create deployment elasticsearch --image=k8s.gcr.io/fluentd-elasticsearch:1.20 -n kube-system --dry-run=client -o yaml > fluentd.yaml. 
Next, remove the replicas, strategy and status fields from the YAML file using a text editor. Also, change the kind from Deployment to DaemonSet.

Finally, create the Daemonset by running kubectl create -f fluentd.yaml

 Static Pods:
      
- The Kubelet can manage a node independtly 
- Kubelet can create pods, usually the API server sends instructions to kubelet about creating pod details. 
- Kubelet instead can read yaml files directly on the server i.e /etc/kubernetes/manifests, place the pod defintion files on the directory, kubelets check the directory 
  creates the pods on the hosts, if app crash, kubelet restarts the pod, recreates pod if file updated.
- called static pods. 
- Directory called --config=kubeconfig.yaml \\ (path called staticPodPath)
- can check static pods by running docker ps command. 
- but what if you do have a cluster/api server etc, can you still create both kind of pods same time = yes it can, kubelet can create pods from api server and can also create 
  static pods from kubelet.  
- kubelet can take request from different inputs, pod definition file from static pods folder and second is from http endpoint.   
- the API Server is still aware of kubectl get pods commands will tell you static pods, when a kubelet creates a static pod mirror object in the api server. 
- can only delete file by changing the pod manifest folder.      
- by default the pod name is appended with the node name by default.    
- static pods:
     - created by kubelet
     - deploy control plane components as static pods 
       
     
 1. How many static pods exist in this cluster in all namespaces? 2  (they have node name next to pod)
 2. Which of the below components is NOT deployed as a static pod? CoreDNS (Cause you can still have all these compononents listed) 
 3. Which of the below components is NOT deployed as a static POD? Run kubectl get pods --all-namespaces and look for the pod from the list that does not end with -controlplane
    kube-proxy   
 4. What is the path of the directory holding the static pod definition files? = /etc/kubernetes/manifests
      First idenity the kubelet config file: ps -aux | grep /usr/bin/kubelet
      grep -i staticpod /var/lib/kubelet/config.yaml As you can see, the path configured is the /etc/kubernetes/manifests directory.
 7. What is the docker image used to deploy the kube-api server as a static pod? = k8s.gcr.io/kube-apiserver:v1.20.0
      Check the image defined in the /etc/kubernetes/manifests/kube-apiserver.yaml manifest file 
 8. Create a static pod named static-busybox that uses the busybox image and the command sleep 1000 =  Create a pod definition file called static-busybox.yaml with the provided specs and place it under /etc/kubernetes/manifests directory.
      - Create a pod definition file in the manifests folder. To do this, run the command:
        kubectl run --restart=Never --image=busybox static-busybox --dry-run=client -o yaml --command -- sleep 1000 > /etc/kubernetes/manifests/static-busybox.yaml
      
      
 Multiple Scheduler:
    
 - can customise the sheduler and dictate how and when it schedule for example schedule a pod to a specific scheduler. 
 - can have multiple scheduler at the same time
 - leader elect is when you have multiple schedulers in a HA cluster master nodes scheduler running on them only 1 can be active at a time, 
   it allows you to elect which kubeschuler on node will be active on 1 time. 
 - to get multiple kube scheduler working must set leader elect option to false 
 - you copy from the kube-scueduler and then put create your customer yaml file for kube-scheduler. 
      
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf 
    - --leader-elect=true
    image: ....
    name: kube-scheduler
    
customer-shceduler config:
     
apiVersion: v1
kind: Pod
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf 
    - --leader-elect=true
    - --scheduler-name=my-custom-scheduler
    image: ....
    name: kube-scheduler
 
create a custom scheduler to a pod: allow pod to use the new kubernetes scheduler. 
  - do, kubectl get pods and copy the pods name to pod definition 
     
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  schedulerName: my-custom-scheduler

- How to tell which kube-scheduler picked up the pod allocation, this can be done by simply doing:
     - kubectl get events, will list all the events - success etc
      
3. Deploy an additional scheduler to the cluster following the given specification.
   Use the manifest file used by kubeadm tool. Use a different port than the one used by the current one.      
     
1. copy kube-scheduler.yaml from the directory /etc/kubernetes/manifests/ to any other location and then change the name to my-scheduler.
   Add or update the following command arguments in the YAML file:     
- --leader-elect=false
- --port=10282
- --scheduler-name=my-scheduler
- --secure-port=0

Here, we are setting leader-elect to false for our new custom scheduler called my-scheduler.
We are also making use of a different port 10282 which is not currently in use in the controlplane.
The default scheduler uses secure-port on port 10259 to serve HTTPS with authentication and authorization. This is not needed for our custom scheduler, so we can disable HTTPS by setting the value of secure-port to 0.

Finally, because we have set secure-port to 0, replace HTTPS with HTTP and use the correct ports under liveness and startup probes.
The final YAML file would look something like this:

---
apiVersion: v1
kind: Pod
metadata:
  labels:
    component: my-scheduler
    tier: control-plane
  name: my-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=false
    - --port=10282
    - --scheduler-name=my-scheduler
    - --secure-port=0
    image: k8s.gcr.io/kube-scheduler:v1.19.0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10282
        scheme: HTTP
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-scheduler
    resources:
      requests:
        cpu: 100m
    startupProbe:
      failureThreshold: 24
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10282
        scheme: HTTP
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/kubernetes/scheduler.conf
      name: kubeconfig
      readOnly: true
  hostNetwork: true
  priorityClassName: system-node-critical
  volumes:
  - hostPath:
      path: /etc/kubernetes/scheduler.conf
      type: FileOrCreate
    name: kubeconfig
status: {}      
      

Final challenge:
 
Configuring Kubernetes scheduler

1. What is the name of the POD that deploys the default kubernetes scheduler in this environment?
   = kube-scheduler-controlplane
2. What is the image used to deploy the kubernetes scheduler?
   Inspect the kubernetes scheduler pod and identify the image
   = k8s.gcr.io/kube-scheduler:v1.23.0
3.

4. Let's create a configmap that the new scheduler will employ using the concept of ConfigMap as a volume.
   Create a configmap with name my-scheduler-config using the content of file /root/my-scheduler-config.yaml
   = kubectl create configmap my-scheduler-config --from-file=/root/my-scheduler-config.yaml -n kube-system

5. Deploy an additional scheduler to the cluster following the given specification.
   Use the manifest file provided at /root/my-scheduler.yaml. Use the same image as used by the default kubernetes scheduler.
      
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: my-scheduler
  name: my-scheduler
  namespace: kube-system
spec:
  serviceAccountName: my-scheduler
  containers:
  - command:
    - /usr/local/bin/kube-scheduler
    - --config=/etc/kubernetes/my-scheduler/my-scheduler-config.yaml
    image: k8s.gcr.io/kube-scheduler:v1.23.0 # changed ---------------------------------------This was changed. 
    livenessProbe:
      httpGet:
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 15
    name: kube-second-scheduler
    readinessProbe:
      httpGet:
        path: /healthz
        port: 10259
        scheme: HTTPS
    resources:
      requests:
        cpu: '0.1'
    securityContext:
      privileged: false
    volumeMounts:
      - name: config-volume
        mountPath: /etc/kubernetes/my-scheduler
  hostNetwork: false
  hostPID: false
  volumes:
    - name: config-volume
      configMap:
        name: my-scheduler-config     
      
6. A POD definition file is given. Use it to create a POD with the new custom scheduler.
   File is located at /root/nginx-pod.yaml    
      
---
apiVersion: v1 
kind: Pod 
metadata:
  name: nginx 
spec:
  schedulerName: my-scheduler ------------------------------changed
  containers:
  - image: nginx
    name: nginx
      
Cluster roles:
 
- Roles and Role binding are Namespaced meaning they are created within Namespace
- Cluster scope = nodes / pvc / clusteroles / clusterrolebindings / certificatesigningrequests / namespaces
- To see namespaced resources run this:
  - kubectl api-resources --namespaced=true
  - kubectl api-resources --namespaced=false
- Clusterroles 
  - cluster admin = can view nodes | can create nodes | can delete nodes 
  - storage admin = can view pvs | can create pvcs | can delete pvcs 
- cluster-admin-role.yaml:
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata: 
       name: cluster-administrator
  rules: 
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list", "get", "create", "delete"]
- kubectl create -f cluster-admin-role.yaml
- next step is to link the user to that role, for this another object called cluster role binding:
     
  cluster-admin-role-binding.yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBindinding
  metadata: 
       name: cluster-admin-role-binding
  subjects:
  - kind: User
    name: cluster-admin
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: cluster-administrator
    apiGroup: rbac.authorisation.k8s.io

- Can also create cluster role for namespace so whatever is in the namespace, that is what the user has access too. 

 Practice test: test-cluster-roles
 1. For the first few questions of this lab, you would have to inspect the existing ClusterRoles and ClusterRoleBindings that have been created in this cluster.
    = kubectl get clusterrole
 2. How many ClusterRoles do you see defined in the cluster? = 69
 3. How many ClusterRoleBindings exist on the cluster? = 54 kubectl get clusterrolebindings --no-headers | wc -l
 4. What namespace is the cluster-admin clusterrole part of?  = Cluster roles are not based on namespaced and cluster wide
 5. What user/groups are the cluster-admin role bound to? = ubectl get clusterrolebindings cluster-admin -o wide = system:masters
 6. What level of permission does the cluster-admin role grant? = perform any action on the resource
 7. A new user michelle joined the team. She will be focusing on the nodes in the cluster. 
    Create the required ClusterRoles and ClusterRoleBindings so she gets access to the nodes.
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: node-admin
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "watch", "list", "create", "delete"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: michelle-binding
subjects:
- kind: User
  name: michelle
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-admin
  apiGroup: rbac.authorization.k8s.io

8. michelle's responsibilities are growing and now she will be responsible for storage as well. 
   Create the required ClusterRoles and ClusterRoleBindings to allow her access to Storage.
   Get the API groups and resource names from command kubectl api-resources. Use the given spec
   = 
9. michelle's responsibilities are growing and now she will be responsible for storage as well. Create the required ClusterRoles and ClusterRoleBindings to allow her access to Storage.
   Get the API groups and resource names from command kubectl api-resources. Use the given spec:
   = 
        ---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: storage-admin
rules:
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "watch", "list", "create", "delete"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "watch", "list", "create", "delete"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: michelle-storage-admin
subjects:
- kind: User
  name: michelle
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: storage-admin
  apiGroup: rbac.authorization.k8s.io
       
     





       
