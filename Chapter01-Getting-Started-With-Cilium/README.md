### Cilium Overview

Cilium provides Connectivity, Observability and Security capabilities in a Cloud Native World, and is based on eBPF.


–Identities, Protocol parsing & Observability

From inception, Cilium was designed for large-scale, highly-dynamic containerized environments.

Cilium:
    natively understands container identities
    parses API protocols like HTTP, gRPC, and Kafka
    and provides visibility and security that is both simpler and more powerful than traditional approaches



### Hubble

Also, meet Hubble: Hubble is a fully distributed networking and security observability platform for Cloud Native workloads.

Hubble is an open source software and is built on top of Cilium and eBPF to enable deep visibility into:

    the communication and behavior of services as well as
    the networking infrastructure in a completely transparent manner

cat kind-config.yaml
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  # localhost.run proxy
  - containerPort: 32042
    hostPort: 32042
  # Hubble relay
  - containerPort: 31234
    hostPort: 31234
  # Hubble UI
  - containerPort: 31235
    hostPort: 31235
- role: worker
- role: worker
networking:
  disableDefaultCNI: true
  kubeProxyMode: none

kubectl get nodes
NAME                 STATUS     ROLES           AGE   VERSION
kind-control-plane   NotReady   control-plane   61m   v1.31.0
kind-worker          NotReady   <none>          61m   v1.31.0
kind-worker2         NotReady   <none>          61m   v1.31.0


### Install Cilium for kind cluster
The cilium CLI tool is provided in this environment to install and check the status of Cilium in the cluster.

Let's start by installing Cilium on the Kind cluster:

exporter CILIUM_VERSION=1.16.2
cilium install --version $CILIUM_VERSION

### Check status of cilium:
cilium status --wait

### Check cilium pods are up and running:
root@server:~# kubectl get pods --all-namespaces 
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
default              deathstar-bf77cddc9-29fz7                    1/1     Running   0          5m52s
default              deathstar-bf77cddc9-llrhk                    1/1     Running   0          5m52s
default              tiefighter                                   1/1     Running   0          5m52s
default              xwing                                        1/1     Running   0          5m52s
kube-system          cilium-hbbwn                                 1/1     Running   0          11m
kube-system          cilium-jzhnh                                 1/1     Running   0          11m
kube-system          cilium-operator-78655b6885-sbt6z             1/1     Running   0          11m
kube-system          cilium-zjbff                                 1/1     Running   0          11m
kube-system          coredns-6f6b679f8f-sc5gh                     1/1     Running   0          75m
kube-system          coredns-6f6b679f8f-smvht                     1/1     Running   0          75m
kube-system          etcd-kind-control-plane                      1/1     Running   0          75m
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          75m
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          75m
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          75m
local-path-storage   local-path-provisioner-57c5987fd4-tk7kh      1/1     Running   0          75m

### Star Wars Demo

To learn how to use and enforce policies with Cilium, we have prepared a demo example.

In the following Star Wars-inspired example, there are three microservice applications: deathstar, tiefighter, and xwing.

  ***The deathstar service***

    The deathstar runs an HTTP webservice on port 80, which is exposed as a Kubernetes Service to load-balance requests to deathstar across two pod replicas.

    The deathstar service provides landing services to the empire’s spaceships so that they can request a landing port.

  ***Allowing ship access***

    The tiefighter pod represents a landing-request client service on a typical empire ship and xwing represents a similar service on an alliance ship.

    With this setup, we can test different security policies for access control to deathstar landing services. 


Let's deploy a simple empire demo application. It is made of several microservices, each identified by Kubernetes labels:

    the Death Star: org=empire, class=deathstar
    the Imperial TIE fighter: org=empire, class=tiefighter
    the Rebel X-Wing: org=alliance, class=xwing

The deployment also includes a deathstar-service, which load-balances traffic to all pods with label org=empire, class=deathstar.   

### Deploy apps and services
root@server:~# kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/http-sw-app.yaml
service/deathstar created
deployment.apps/deathstar created
pod/tiefighter created
pod/xwing created


### Verify pods, services are up and running:
root@server:~# kubectl get pods,svc
NAME                            READY   STATUS    RESTARTS   AGE
pod/deathstar-bf77cddc9-29fz7   1/1     Running   0          69s
pod/deathstar-bf77cddc9-llrhk   1/1     Running   0          69s
pod/tiefighter                  1/1     Running   0          69s
pod/xwing                       1/1     Running   0          69s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/deathstar    ClusterIP   10.96.19.239   <none>        80/TCP    69s
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   71m

### Each pod will also be represented in Cilium as an Endpoint
root@server:~# kubectl get cep --all-namespaces
NAMESPACE            NAME                                      SECURITY IDENTITY   ENDPOINT STATE   IPV4           IPV6
default              deathstar-bf77cddc9-29fz7                 8334                ready            10.244.2.229   
default              deathstar-bf77cddc9-llrhk                 8334                ready            10.244.1.95    
default              tiefighter                                20185               ready            10.244.2.228   
default              xwing                                     20171               ready            10.244.1.66    
kube-system          coredns-6f6b679f8f-sc5gh                  13063               ready            10.244.1.22    
kube-system          coredns-6f6b679f8f-smvht                  13063               ready            10.244.2.86    
local-path-storage   local-path-provisioner-57c5987fd4-tk7kh   20015               ready            10.244.1.25   


### Death Star access

From the perspective of the deathstar service, only the ships with label org=empire are allowed to connect and request landing!

To simulate our connectivity tests, we will be executing simple API calls using curl.

Let's test if we can land our TIE fighter & Xwing on the Death Star by running the following command:

root@server:~# kubectl exec tiefighter -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

root@server:~# kubectl exec xwing -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

### Cilium uses the labels assigned to pods to define security policies.

Writing Network Policies

We’ll start with a basic policy restricting deathstar landing requests to only the ships that have the label org=empire.

This blocks any ships without the org=empire label to even connect to the deathstar service.

This is a simple policy that filters only on network layer 3 (IP protocol) and network layer 4 (TCP protocol), so it is often referred to as a L3/L4 network security policy.

### Enforce network policy:
root@server:~# kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/sw_l3_l4_policy.yaml
ciliumnetworkpolicy.cilium.io/rule1 created

Now let's try to land the empire tiefighter again (HTTP POST from tiefighter to deathstar on the /v1/request-landing path):
root@server:~# kubectl exec tiefighter -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

In comparison, if you try to request landing from the xwing pod, you will see that the request will eventually time out:
root@server:~# kubectl exec xwing -- curl -s -m 5 -XPOST deathstar.default.svc.cluster.local/v1/request-landing
command terminated with exit code 28

We have successfully blocked access to the deathstar from an X-Wing ship. Let's now see how we could make this policy a bit more fine-grained using L7 rules.

### Limit access to set of HTTP requests it requires legitimate operations by filtering on HTTP

So we need to filter on a higher level: we need to filter the actual HTTP requests!

Consider that the deathstar service exposes some maintenance APIs which should not be called by random empire ships. To see why those APIs are sensitive, run:

root@server:~# kubectl exec tiefighter -- \
  curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Panic: deathstar exploded

goroutine 1 [running]:
main.HandleGarbage(0x2080c3f50, 0x2, 0x4, 0x425c0, 0x5, 0xa)
        /code/src/github.com/empire/deathstar/
        temp/main.go:9 +0x64
main.main()
        /code/src/github.com/empire/deathstar/
        temp/main.go:5 +0x85

Yes, there is a Panic: the Death Star just exploded!

rules:
  http:
  - method: "POST"
    path: "/v1/request-landing"

This will restrict API access to only the /v1/request-landing path and will thus prevent users from accessing the /v1/exhaust-port, which caused a crash as we saw earlier.

root@server:~# kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/sw_l3_l4_l7_policy.yaml
ciliumnetworkpolicy.cilium.io/rule1 configured

root@server:~# kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Access denied

As you can see, with Cilium L7 security policies, we are able to restrict tiefighter's access to only the required API resources on deathstar, thereby implementing a “least privilege” security approach for communication between microservices.
