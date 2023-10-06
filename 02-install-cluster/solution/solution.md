# Solution

You can find a full description of the [installation steps](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) in the official Kubernetes documentation. The instructions below describe the manual steps. This directory also contains `Vagrantfile` that creates the full cluster in one swoop without manual intervention.

## Initializing the Control Plane Node

After shelling into the control plane node with `vagrant ssh kube-control-plane`, run the `kubeadm init` command as root user. This initializes the control plane node. The output contains follow up command you will keep track of.

```
$ sudo kubeadm init --pod-network-cidr 172.18.0.0/16 --apiserver-advertise-address 192.168.56.10
...
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.56.10:6443 --token fi8io0.dtkzsy9kws56dmsp \
    --discovery-token-ca-cert-hash sha256:cc89ea1f82d5ec460e21b69476e0c052d691d0c52cce83fbd7e403559c1ebdac
```

Should you forget about the `join` command, run the following to retrieve it.

```
$ kubeadm token create --print-join-command
```

Next, execute the commands from the output:

```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

It's recommended to install a Pod network add-on. We'll use Calico here. The following command applies the manifest with version 3.22.

```
$ kubectl apply -f https://projectcalico.docs.tigera.io/archive/v3.22/manifests/calico.yaml
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
poddisruptionbudget.policy/calico-kube-controllers created
```

The list of nodes should show the following output:

```
$ kubectl get nodes
NAME                 STATUS   ROLES                  AGE   VERSION
kube-control-plane   Ready    control-plane,master   80s   v1.23.4
```

Exit the control plane node by running the `exit` command.

## Joining the Worker Nodes

Shell into worker node 1 or 2 with the command `vagrant ssh kube-worker-1` or `vagrant ssh kube-worker-2`, and run the `kubeadm join` command. You can simply copy the command from the output of the `init` command. The following command showns an example. Remember that the token and SHA256 hash will be different for you.

```
$ sudo kubeadm join 192.168.56.10:6443 --token fi8io0.dtkzsy9kws56dmsp --discovery-token-ca-cert-hash sha256:cc89ea1f82d5ec460e21b69476e0c052d691d0c52cce83fbd7e403559c1ebdac
```

After applying the `join` command for both worker nodes, the list of nodes should render the following output. The command needs to be run on the control plane node.

```
$ kubectl get nodes
NAME                  STATUS   ROLES                  AGE     VERSION
kube-control-plane    Ready    control-plane,master   5m1s    v1.23.4
kube-worker-1         Ready    <none>                 2m22s   v1.23.4
kube-worker-2         Ready    <none>                 32s     v1.23.4
```

## Verifying the Installation

To verify the installation, you can create a new Pod and inspect which node it is running on.

```
$ kubectl run nginx --image=nginx
pod/nginx created
$ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          9s
$ kubectl describe pod nginx | grep Node:
Node:         kube-worker-1/10.0.2.15
```
