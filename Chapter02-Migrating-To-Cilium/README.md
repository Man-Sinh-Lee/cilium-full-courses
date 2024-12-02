Migrating to Cilium

Migrating to Cilium from another CNI is a very common task. But how do we minimize the impact during the migration? How do we ensure pods on the legacy CNI can still communicate to Cilium-managed during pods during the migration? How do we execute the migration safely, while avoiding a overly complex approach or using a separate tool such as Multus?

With the use of the new Cilium CRD CiliumNodeConfig, running clusters can be migrated on a node-by-node basis, without disrupting existing traffic or requiring a complete cluster outage or rebuild.

In this lab, you will migrate your cluster from an existing CNI to Cilium. While we use Flannel in this simple lab, you can leverage the same approach for other CNIs (albeit migrating from a CNI like Calico might require more efforts).

Note that this feature is still beta and we expect further tooling and automation to be developed to support large cluster migrations.

### Setup kind cluster
root@server:~# cat cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
networking:
  disableDefaultCNI: true


Flannel was deployed to the cluster to provide network connectivity.

root@server:~# kubectl get nodes
NAME                 STATUS   ROLES           AGE     VERSION
kind-control-plane   Ready    control-plane   3m32s   v1.29.2
kind-worker          Ready    <none>          3m8s    v1.29.2
kind-worker2         Ready    <none>          3m10s   v1.29.2
root@server:~# kubectl get ds/kube-flannel-ds -n kube-flannel
NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kube-flannel-ds   3         3         3       3            3           <none>          3m12s

Flannel was deployed to the cluster to provide network connectivity.
root@server:~# kubectl get node -o jsonpath="{range .items[*]}{.metadata.name} {.spec.podCIDR}{'\n'}{end}" | column -t
kind-control-plane  10.244.0.0/24
kind-worker         10.244.2.0/24
kind-worker2        10.244.1.0/24

root@server:~# yq pod.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80

root@server:~# kubectl apply -f /root/pod.yaml
deployment.apps/nginx-deployment created
root@server:~# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
nginx-deployment-7c79c4bf97-5j7bw   1/1     Running   0          10s   10.244.1.6   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-68xhs   1/1     Running   0          10s   10.244.2.2   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-7zt4j   1/1     Running   0          10s   10.244.1.7   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-9k4n8   1/1     Running   0          10s   10.244.1.8   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-m52s4   1/1     Running   0          10s   10.244.2.6   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-mb8br   1/1     Running   0          10s   10.244.2.5   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-mkf26   1/1     Running   0          10s   10.244.1.9   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-mnjqv   1/1     Running   0          10s   10.244.1.5   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-mnn52   1/1     Running   0          10s   10.244.2.3   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-xxnvv   1/1     Running   0          10s   10.244.2.4   kind-worker    <none>           <none> 

Background (1/2)

When the kubelet creates a Pod’s Sandbox, the CNI (specified in the /etc/cni/net.d/ directory) is called. The CNI will handle the networking for a Pod - including allocating an IP address, creating & configuring a network interface, and (potentially) establishing an overlay network. The Pod’s network configuration shares the same life cycle as the PodSandbox.

When migrating CNIs, there are several approaches with pros and cons.

Background (2/2)
Migration Approaches

    The ideal scenario is to build a brand new cluster and to migrate workloads using a GitOps approach. But this can involve a lot of prep work and potential disruptions.
    Another method consists in reconfiguring /etc/cni/net.d/ to point to Cilium. However, any existing Pods will still have been configured by the old network plugin and any new Pods will be configured by the newer CNI. To complete the migration, all Pods on the cluster that are configured by the old CNI must be recycled in order to be a member of the new CNI.
    A naive approach to migrating a CNI would be to reconfigure all nodes with a new CNI and then gradually restart each node in the cluster, thus replacing the CNI when the node is brought back up and ensuring that all pods are part of the new CNI. This simple migration, while effective, comes at the cost of disrupting cluster connectivity during the rollout. Unmigrated and migrated nodes would be split in to two “islands” of connectivity, and pods would be randomly unable to reach one-another until the migration is complete.

In this lab, you will learn about a new approach.

Migration via dual overlays

Cilium supports a hybrid mode, where two separate overlays are established across the cluster. While Pods on a given node can only be attached to one network, they have access to both Cilium and non-Cilium pods while the migration is taking place. As long as Cilium and the existing networking provider use a separate IP range, the Linux routing table takes care of separating traffic.

In this lab, we will use a model for live migrating between two deployed CNI implementations. This will have the benefit of reducing downtime of nodes and workloads and ensuring that workloads on both configured CNIs can communicate during migration.

For live migration to work, Cilium will be installed with a separate CIDR range and encapsulation port than that of the currently installed CNI. As long as Cilium and the existing CNI use a separate IP range, the Linux routing table takes care of separating traffic.

Requirements

Live migration requires the following:

    A new, distinct Cluster CIDR for Cilium to use
    Use of the Cluster Pool IPAM mode
    A distinct overlay, either protocol or port
    An existing network plugin that uses the Linux routing stack, such as Flannel, Calico, or AWS-CNI


Migration Overview

The migration process utilizes the per-node configuration feature to selectively enable Cilium CNI. This allows for a controlled rollout of Cilium without disrupting existing workloads.

Cilium will be installed, first, in a mode where it establishes an overlay but does not provide CNI networking for any pods. Then, individual nodes will be migrated.

In summary, the process looks like:

    Prepare the cluster and install Cilium in “secondary” mode.
    Cordon, drain, migrate, and reboot each node
    Remove the existing network provider
    (Optional) Reboot each node again

In the next task, you will prepare the cluster and install Cilium in "secondary" mode.


We have pre-created a Cilium Helm configuration file values-migration.yaml
yq values-migration.yaml
---
operator:
  unmanagedPodWatcher:
    restart: false # prevent Cilium from restarting Pods that are not being managed by Cilium
tunnelPort: 8473 # encapsulation (VXLAN)
cni:
  customConf: true # prevent Cilium from taking over immediately
  uninstall: false # prevent the CNI configuration file and plugin binaries to be removed
ipam: # use of cluster-pool IPAM mode and a distinct PodCIDR during the migration
  mode: "cluster-pool"
  operator:
    clusterPoolIPv4PodCIDRList: ["10.245.0.0/16"]
policyEnforcementMode: "never"
bpf:
  hostLegacyRouting: true # route traffic via host stack to provide connectivity during the migration

root@server:~# cilium install --helm-values values-migration.yaml --dry-run-helm-values >> values-initial.yaml
root@server:~# yq values-initial.yaml 
bpf:
  hostLegacyRouting: true
cluster:
  name: kind-kind
cni:
  customConf: true
  uninstall: false
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - 10.245.0.0/16
operator:
  replicas: 1
  unmanagedPodWatcher:
    restart: false
policyEnforcementMode: never
routingMode: tunnel
tunnelPort: 8473
tunnelProtocol: vxlan


root@server:~# helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --namespace kube-system --values values-initial.yaml
"cilium" has been added to your repositories
NAME: cilium
LAST DEPLOYED: Mon Dec  2 03:01:43 2024
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
You have successfully installed Cilium with Hubble.

Your release version is 1.16.4.

For any further help, visit https://docs.cilium.io/en/v1.16/gettinghelp

The Cilium agent process supports setting configuration on a per-node basis instead of constant configuration across the cluster. This allows overriding the global Cilium config for a node or set of nodes. It is managed by CiliumNodeConfig objects. Note the Cilium CiliumNodeConfig CRD was added in Cilium 1.13.

A CiliumNodeConfig object consists of a set of fields and a label selector. The label selector defines to which nodes the configuration applies.

Let's now create a per-node config that will instruct Cilium to “take over” CNI networking on the node.

root@server:~# cat <<EOF | kubectl apply --server-side -f -
apiVersion: cilium.io/v2
kind: CiliumNodeConfig
metadata:
  namespace: kube-system
  name: cilium-default
spec:
  nodeSelector:
    matchLabels:
      io.cilium.migration/cilium-default: "true"
  defaults:
    write-cni-conf-when-ready: /host/etc/cni/net.d/05-cilium.conflist
    custom-cni-conf: "false"
    cni-chaining-mode: "none"
    cni-exclusive: "true"
EOF
ciliumnodeconfig.cilium.io/cilium-default serverside-applied
root@server:~# kubectl -n kube-system get ciliumnodeconfigs.cilium.io cilium-default -o yaml
apiVersion: cilium.io/v2
kind: CiliumNodeConfig
metadata:
  creationTimestamp: "2024-12-02T03:08:28Z"
  generation: 1
  name: cilium-default
  namespace: kube-system
  resourceVersion: "5443"
  uid: 767cc2c1-4970-4bd7-9751-514d10cecb1e
spec:
  defaults:
    cni-chaining-mode: none
    cni-exclusive: "true"
    custom-cni-conf: "false"
    write-cni-conf-when-ready: /host/etc/cni/net.d/05-cilium.conflist
  nodeSelector:
    matchLabels:
      io.cilium.migration/cilium-default: "true"
root@server:~# 

Initially, this will not apply to any nodes.

As you can see in the spec.nodeSelector section, the CiliumNodeConfig only applies to nodes with the io.cilium.migration/cilium-default: "true" label. We will gradually migrate nodes by applying the label to each node, one by one.

Once the node is reloaded, the custom Cilium configuration will be applied, the CNI configuration will be written and the CNI functionality will be enabled.

### Cordon and drain the node:
Cordon a node will prevent new pods from being scheduled on the node.
Drain a node will gracefully evict all the running pods from the node

root@server:~# NODE="kind-worker"
root@server:~# kubectl cordon $NODE
node/kind-worker cordoned
root@server:~# kubectl scale deployment nginx-deployment --replicas=12
deployment.apps/nginx-deployment scaled
root@server:~# kubectl drain $NODE --ignore-daemonsets
node/kind-worker already cordoned
Warning: ignoring DaemonSet-managed Pods: kube-flannel/kube-flannel-ds-h6cgm, kube-system/cilium-envoy-tjk7l, kube-system/cilium-ssw7c, kube-system/install-cni-plugins-9655j, kube-system/kube-proxy-9dv4l
evicting pod kube-system/cilium-operator-597d7f99c5-bzrkg
evicting pod default/nginx-deployment-7c79c4bf97-68xhs
evicting pod default/nginx-deployment-7c79c4bf97-mb8br
evicting pod default/nginx-deployment-7c79c4bf97-xxnvv
evicting pod default/nginx-deployment-7c79c4bf97-mnn52
evicting pod default/nginx-deployment-7c79c4bf97-m52s4
pod/cilium-operator-597d7f99c5-bzrkg evicted
pod/nginx-deployment-7c79c4bf97-68xhs evicted
pod/nginx-deployment-7c79c4bf97-mb8br evicted
pod/nginx-deployment-7c79c4bf97-xxnvv evicted
pod/nginx-deployment-7c79c4bf97-m52s4 evicted
pod/nginx-deployment-7c79c4bf97-mnn52 evicted
node/kind-worker drained
root@server:~# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
nginx-deployment-7c79c4bf97-2kv48   1/1     Running   0          9s    10.244.1.12   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-4c4bv   1/1     Running   0          9s    10.244.1.15   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-4d6dm   1/1     Running   0          9s    10.244.1.16   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-5c55j   1/1     Running   0          9s    10.244.1.14   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-5j7bw   1/1     Running   0          40m   10.244.1.6    kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-7zt4j   1/1     Running   0          40m   10.244.1.7    kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-9k4n8   1/1     Running   0          40m   10.244.1.8    kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-frbdm   1/1     Running   0          19s   10.244.1.10   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-mkf26   1/1     Running   0          40m   10.244.1.9    kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-mnjqv   1/1     Running   0          40m   10.244.1.5    kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-p6m77   1/1     Running   0          9s    10.244.1.13   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-xzvkl   1/1     Running   0          19s   10.244.1.11   kind-worker2   <none>           <none>

We can now label the node: this causes the CiliumNodeConfig to apply to this node.

root@server:~# kubectl label node $NODE --overwrite "io.cilium.migration/cilium-default=true"
node/kind-worker labeled


Let's restart Cilium on the node. That will trigger the creation of CNI configuration file.

root@server:~# kubectl -n kube-system delete pod --field-selector spec.nodeName=$NODE -l k8s-app=cilium
kubectl -n kube-system rollout status ds/cilium -w
pod "cilium-ssw7c" deleted
Waiting for daemon set "cilium" rollout to finish: 2 of 3 updated pods are available...
daemon set "cilium" successfully rolled out

Restart the node:
root@server:~# docker restart $NODE
kind-worker

Check Cilium status
root@server:~# cilium status --wait
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium-envoy       Desired: 3, Ready: 3/3, Available: 3/3
DaemonSet              cilium             Desired: 3, Ready: 3/3, Available: 3/3
Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium-operator    Running: 1
                       cilium-envoy       Running: 3
                       cilium             Running: 3
Cluster Pods:          0/15 managed by Cilium
Helm chart version:    
Image versions         cilium             quay.io/cilium/cilium:v1.16.4@sha256:d55ec38938854133e06739b1af237932b9c4dd4e75e9b7b2ca3acc72540a44bf: 3
                       cilium-operator    quay.io/cilium/operator-generic:v1.16.4@sha256:c55a7cbe19fe0b6b28903a085334edb586a3201add9db56d2122c8485f7a51c5: 1
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.30.7-1731393961-97edc2815e2c6a174d3d12e71731d54f5d32ea16@sha256:0287b36f70cfbdf54f894160082f4f94d1ee1fb10389f3a95baa6c8e448586ed: 3


We are going to deploy a pod shortly. We will verify that Cilium attributes the IP to the pod.

Remember that we rolled out Cilium in cluster-scope IPAM mode where Cilium assigns per-node PodCIDRs to each node and allocates IPs using a host-scope allocator on each node. The Cilium operator will manage the per-node PodCIDRs via the CiliumNode resource.

The following command will check the CiliumNode resource and will show us the Pod CIDRs used to allocate IP addresses to the pods:
root@server:~# kubectl get cn kind-worker -o jsonpath='{.spec.ipam.podCIDRs[0]}'
10.245.2.0/24

### Verify pod picks an IP from the Cilium CIDR
root@server:~# kubectl run --attach --rm --restart=Never verify  --overrides='{"spec": {"nodeName": "'$NODE'", "tolerations": [{"operator": "Exists"}]}}'   --image alpine -- /bin/sh -c 'ip addr' | grep 10.245 -B 2
10: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether de:3d:bb:11:ad:5f brd ff:ff:ff:ff:ff:ff
    inet 10.245.2.4/32 scope global eth0

root@server:~# NGINX=($(kubectl get pods -l app=nginx -o=jsonpath='{.items[0].status.podIP}'))
echo $NGINX
10.244.1.12
root@server:~# kubectl run --attach --rm --restart=Never verify  --overrides='{"spec": {"nodeName": "'$NODE'", "tolerations": [{"operator": "Exists"}]}}'   --image alpine/curl --env NGINX=$NGINX -- /bin/sh -c 'curl -s $NGINX '
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
pod "verify" deleted    

### Uncordon the node and verify pods deployed on kind-worker
root@server:~# kubectl uncordon $NODE
node/kind-worker uncordoned

root@server:~# kubectl get pods --field-selector spec.nodeName=kind-worker
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7c79c4bf97-bgq75   1/1     Running   0          2m3s
nginx-deployment-7c79c4bf97-c5rrp   1/1     Running   0          2m3s
nginx-deployment-7c79c4bf97-sbsnq   1/1     Running   0          2m3s

### Cordon and drain kind-worker2 node:
root@server:~# NODE="kind-worker2"
kubectl cordon $NODE
node/kind-worker2 cordoned
root@server:~# kubectl drain $NODE --ignore-daemonsets
node/kind-worker2 already cordoned
Warning: ignoring DaemonSet-managed Pods: kube-flannel/kube-flannel-ds-9fd96, kube-system/cilium-7fmng, kube-system/cilium-envoy-5ld2s, kube-system/install-cni-plugins-p6rj7, kube-system/kube-proxy-9fkbx
evicting pod local-path-storage/local-path-provisioner-7577fdbbfb-wbr2l
evicting pod default/nginx-deployment-7c79c4bf97-mkf26
evicting pod default/nginx-deployment-7c79c4bf97-2kv48
evicting pod default/nginx-deployment-7c79c4bf97-7zt4j
evicting pod default/nginx-deployment-7c79c4bf97-4c4bv
evicting pod default/nginx-deployment-7c79c4bf97-xzvkl
evicting pod default/nginx-deployment-7c79c4bf97-mnjqv
evicting pod default/nginx-deployment-7c79c4bf97-5c55j
evicting pod default/nginx-deployment-7c79c4bf97-frbdm
evicting pod kube-system/coredns-76f75df574-22sm2
evicting pod kube-system/coredns-76f75df574-cx27x
evicting pod default/nginx-deployment-7c79c4bf97-4d6dm
evicting pod default/nginx-deployment-7c79c4bf97-p6m77
evicting pod default/nginx-deployment-7c79c4bf97-9k4n8
evicting pod default/nginx-deployment-7c79c4bf97-5j7bw
I1202 03:21:38.792986   66884 request.go:697] Waited for 1.000577592s due to client-side throttling, not priority and fairness, request: POST:https://127.0.0.1:34721/api/v1/namespaces/default/pods/nginx-deployment-7c79c4bf97-2kv48/eviction
pod/nginx-deployment-7c79c4bf97-frbdm evicted
pod/nginx-deployment-7c79c4bf97-7zt4j evicted
pod/nginx-deployment-7c79c4bf97-xzvkl evicted
pod/nginx-deployment-7c79c4bf97-4d6dm evicted
pod/nginx-deployment-7c79c4bf97-mkf26 evicted
pod/nginx-deployment-7c79c4bf97-4c4bv evicted
pod/nginx-deployment-7c79c4bf97-mnjqv evicted
pod/nginx-deployment-7c79c4bf97-9k4n8 evicted
pod/nginx-deployment-7c79c4bf97-5j7bw evicted
pod/nginx-deployment-7c79c4bf97-5c55j evicted
pod/nginx-deployment-7c79c4bf97-p6m77 evicted
pod/nginx-deployment-7c79c4bf97-2kv48 evicted
pod/coredns-76f75df574-22sm2 evicted
pod/coredns-76f75df574-cx27x evicted
pod/local-path-provisioner-7577fdbbfb-wbr2l evicted
node/kind-worker2 drained
root@server:~# 

# all pods are running on kind-worker
root@server:~# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP             NODE          NOMINATED NODE   READINESS GATES
nginx-deployment-7c79c4bf97-4jgh5   1/1     Running   0          66s     10.245.2.166   kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-6khk5   1/1     Running   0          65s     10.245.2.128   kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-6mmmm   1/1     Running   0          66s     10.245.2.193   kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-6mx4d   1/1     Running   0          66s     10.245.2.155   kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-bgq75   1/1     Running   0          3m43s   10.245.2.244   kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-c5rrp   1/1     Running   0          3m43s   10.245.2.198   kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-d46n8   1/1     Running   0          65s     10.245.2.56    kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-djczw   1/1     Running   0          66s     10.245.2.203   kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-gk5k5   1/1     Running   0          65s     10.245.2.25    kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-jwkvs   1/1     Running   0          66s     10.245.2.152   kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-kcl9d   1/1     Running   0          66s     10.245.2.164   kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-sbsnq   1/1     Running   0          3m43s   10.245.2.231   kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-vprn6   1/1     Running   0          66s     10.245.2.109   kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-z69jd   1/1     Running   0          66s     10.245.2.252   kind-worker   <none>           <none>
nginx-deployment-7c79c4bf97-zmg7s   1/1     Running   0          65s     10.245.2.61    kind-worker   <none>           <none>

Check Cilium status:
root@server:~# cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium-envoy       Desired: 3, Ready: 3/3, Available: 3/3
DaemonSet              cilium             Desired: 3, Ready: 3/3, Available: 3/3
Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium-operator    Running: 1
                       cilium-envoy       Running: 3
                       cilium             Running: 3
Cluster Pods:          17/18 managed by Cilium
Helm chart version:    
Image versions         cilium             quay.io/cilium/cilium:v1.16.4@sha256:d55ec38938854133e06739b1af237932b9c4dd4e75e9b7b2ca3acc72540a44bf: 3
                       cilium-operator    quay.io/cilium/operator-generic:v1.16.4@sha256:c55a7cbe19fe0b6b28903a085334edb586a3201add9db56d2122c8485f7a51c5: 1
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.30.7-1731393961-97edc2815e2c6a174d3d12e71731d54f5d32ea16@sha256:0287b36f70cfbdf54f894160082f4f94d1ee1fb10389f3a95baa6c8e448586ed: 3

### Label node, restart cilium, reboot and uncordon kind-worker2

root@server:~# kubectl label node $NODE --overwrite "io.cilium.migration/cilium-default=true"
node/kind-worker2 labeled
root@server:~# kubectl -n kube-system delete pod --field-selector spec.nodeName=$NODE -l k8s-app=cilium
kubectl -n kube-system rollout status ds/cilium -w
pod "cilium-7fmng" deleted
Waiting for daemon set "cilium" rollout to finish: 2 of 3 updated pods are available...
daemon set "cilium" successfully rolled out
root@server:~# docker restart $NODE
kind-worker2
root@server:~# kubectl uncordon $NODE
node/kind-worker2 uncordoned

### Verify if pods can be deployed on kind-worker2
root@server:~# kubectl get pods -owide
NAME                                READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
nginx-deployment-7c79c4bf97-4jgh5   1/1     Running   0          7m      10.245.2.166   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-5mc58   1/1     Running   0          44s     10.245.1.223   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-6khk5   1/1     Running   0          6m59s   10.245.2.128   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-6mmmm   1/1     Running   0          7m      10.245.2.193   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-6mx4d   1/1     Running   0          7m      10.245.2.155   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-8474z   1/1     Running   0          44s     10.245.1.216   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-bgq75   1/1     Running   0          9m37s   10.245.2.244   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-c5rrp   1/1     Running   0          9m37s   10.245.2.198   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-d46n8   1/1     Running   0          6m59s   10.245.2.56    kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-djczw   1/1     Running   0          7m      10.245.2.203   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-fdglp   1/1     Running   0          44s     10.245.1.234   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-gk5k5   1/1     Running   0          6m59s   10.245.2.25    kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-jwkvs   1/1     Running   0          7m      10.245.2.152   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-kcl9d   1/1     Running   0          7m      10.245.2.164   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-r5rtx   1/1     Running   0          44s     10.245.1.157   kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-sbsnq   1/1     Running   0          9m37s   10.245.2.231   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-vprn6   1/1     Running   0          7m      10.245.2.109   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-wzfjl   1/1     Running   0          44s     10.245.1.89    kind-worker2   <none>           <none>
nginx-deployment-7c79c4bf97-z69jd   1/1     Running   0          7m      10.245.2.252   kind-worker    <none>           <none>
nginx-deployment-7c79c4bf97-zmg7s   1/1     Running   0          6m59s   10.245.2.61    kind-worker    <none>           <none>

### Cordon, drain, label, restart and uncordon master node
root@server:~# NODE="kind-control-plane"
kubectl cordon $NODE
node/kind-control-plane cordoned
root@server:~# kubectl drain $NODE --ignore-daemonsets
node/kind-control-plane already cordoned
Warning: ignoring DaemonSet-managed Pods: kube-flannel/kube-flannel-ds-g7dll, kube-system/cilium-envoy-qjqws, kube-system/cilium-lgs6j, kube-system/install-cni-plugins-l68w7, kube-system/kube-proxy-8txvs
evicting pod kube-system/coredns-76f75df574-kk9jk
evicting pod kube-system/cilium-operator-597d7f99c5-wgzlj
pod/cilium-operator-597d7f99c5-wgzlj evicted
pod/coredns-76f75df574-kk9jk evicted
node/kind-control-plane drained

root@server:~# kubectl label node $NODE --overwrite "io.cilium.migration/cilium-default=true"
node/kind-control-plane labeled
root@server:~# kubectl -n kube-system delete pod --field-selector spec.nodeName=$NODE -l k8s-app=cilium
kubectl -n kube-system rollout status ds/cilium -w
pod "cilium-lgs6j" deleted
Waiting for daemon set "cilium" rollout to finish: 2 of 3 updated pods are available...
daemon set "cilium" successfully rolled out
root@server:~# docker restart $NODE
kind-control-plane
root@server:~# kubectl uncordon $NODE
node/kind-control-plane uncordoned
root@server:~# cilium status --wait

root@server:~# cilium install --helm-values values-initial.yaml --helm-set operator.unmanagedPodWatcher.restart=true --helm-set cni.customConf=false --helm-set policyEnforcementMode=default --dry-run-helm-values >> values-final.yaml
root@server:~# diff values-initial.yaml values-final.yaml
6c6
<   customConf: true
---
>   customConf: false
16,17c16,17
<     restart: false
< policyEnforcementMode: never
---
>     restart: true
> policyEnforcementMode: default

root@server:~# helm upgrade --namespace kube-system cilium cilium/cilium --values values-final.yaml
root@server:~# kubectl -n kube-system rollout restart daemonset cilium

root@server:~# cilium status --wait

root@server:~# kubectl delete -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
namespace "kube-flannel" deleted
serviceaccount "flannel" deleted
clusterrole.rbac.authorization.k8s.io "flannel" deleted
clusterrolebinding.rbac.authorization.k8s.io "flannel" deleted
configmap "kube-flannel-cfg" deleted
daemonset.apps "kube-flannel-ds" deleted