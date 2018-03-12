# Kubernetes The Hard Way (Vagrant/Ansible)

Vagrant  and Ansible configuration for a Kubernetes setup.

The setup follows closely https://github.com/kelseyhightower/kubernetes-the-hard-way
with the following exceptions:

* `docker` is used as as default container runtime, `cri-containerd` is also supported.
* `10.240.0.40` is the IP of the loadbalancer (haproxy) for HA controllers

![Kubernetes Components](docs/kubernetes-architecture.png)

## Requirements Host

* Vagrant (with VirtualBox)
* Minimum of 7x 512MB of free RAM
* ansible 2.4+


## Setup

Download external ansible galaxy roles ( wtanaka.haproxy and reallyenglish.hosts )

```
ansible-galaxy install -r requirements.yml -p roles/
```
> ourput

```
- downloading role 'haproxy', owned by wtanaka
- downloading role from https://github.com/wtanaka/ansible-role-haproxy/archive/master.tar.gz
- extracting wtanaka.haproxy to /home/mhersi/code/kubernetes-the-hard-way/roles/wtanaka.haproxy
- wtanaka.haproxy (master) was installed successfully
- downloading role 'hosts', owned by reallyenglish
- downloading role from https://github.com/reallyenglish/ansible-role-hosts/archive/1.2.0.tar.gz
- extracting reallyenglish.hosts to /home/mhersi/code/kubernetes-the-hard-way/roles/reallyenglish.hosts
- reallyenglish.hosts (1.2.0) was installed successfully

```

Create virtual machines:

```
vagrant up
```

> This takes around 5-10 mins, Take coffee/tea break.

Check status of virtual machines:

```
vagrant status
```
> output

```
Current machine states:

lb-0                      running (virtualbox)
controller-0              running (virtualbox)
controller-1              running (virtualbox)
controller-2              running (virtualbox)
worker-0                  running (virtualbox)
worker-1                  running (virtualbox)
worker-2                  running (virtualbox)
```

Test ansible host connectivity:

```
ansible -i inventory -m ping all
```
> output

```
controller-2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
lb-0 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
controller-1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
worker-0 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
controller-0 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
worker-1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
worker-2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Bootstrap Kubernetes 1.9 Cluster:

```
ansible-playbook -i inventory k8s_cluster.yml
```

> This takes around 10 mins, Take another coffee/tea break :)

## Check Cluster status

Check the health of the remote Kubernetes cluster:

```
kubectl get componentstatuses
```

> output

```
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-1               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}
```

Check Kubernetes nodes status:

```
kubectl get nodes
```

> output

```
NAME       STATUS    ROLES     AGE       VERSION
worker-0   Ready     <none>    1h        v1.9.3
worker-1   Ready     <none>    1h        v1.9.3
worker-2   Ready     <none>    1h        v1.9.3
```
## Setup DNS add-on

Deploy the kube-dns cluster add-on:

```
kubectl create -f ./manifests/kube-dns/kube-dns.yaml
```

> output

```
serviceaccount "kube-dns" created
configmap "kube-dns" created
service "kube-dns" created
deployment "kube-dns" created
```

List the pods created by the `kube-dns` deployment:

```
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> output

```
NAME                        READY     STATUS    RESTARTS   AGE
kube-dns-6c857864fb-c9j46   3/3       Running   0          5m

```

Create a `busybox` deployment:

```
kubectl run busybox --image=busybox --command -- sleep 3600
```

List the pod created by the `busybox` deployment:

```
kubectl get pods -l run=busybox
```

> output

```
NAME                       READY     STATUS    RESTARTS   AGE
busybox-855686df5d-hnsdn   1/1       Running   0          1m
```

Retrieve the full name of the `busybox` pod:

```
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Execute a DNS lookup for the `kubernetes` service inside the `busybox` pod:

```
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> output

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

## Smoke Test

In this lab you will complete a series of tasks to ensure your Kubernetes cluster is functioning correctly.

### Data Encryption

In this section you will verify the ability to [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Create a generic secret:

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:

```
vagrant ssh controller-0 \
  -c "ETCDCTL_API=3 etcdctl get /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

> output

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a ea 7c 76 32 43 62 6f  |:v1:key1:.|v2Cbo|
00000050  44 02 02 8c b7 ca fe 95  a5 33 f6 a1 18 6c 3d 53  |D........3...l=S|
00000060  e7 9c 51 ee 32 f6 e4 17  ea bb 11 d5 2f e2 40 00  |..Q.2......./.@.|
00000070  ae cf d9 e7 ba 7f 68 18  d3 c1 10 10 93 43 35 bd  |......h......C5.|
00000080  24 dd 66 b4 f8 f9 82 77  4a d5 78 03 19 41 1e bc  |$.f....wJ.x..A..|
00000090  94 3f 17 41 ad cc 8c ba  9f 8f 8e 56 97 7e 96 fb  |.?.A.......V.~..|
000000a0  8f 2e 6a a5 bf 08 1f 0b  c3 4b 2b 93 d1 ec f8 70  |..j......K+....p|
000000b0  c1 e4 1d 1a d2 0d f8 74  3a a1 4f 3c e0 c9 6d 3f  |.......t:.O<..m?|
000000c0  de a3 f5 fd 76 aa 5e bc  27 d9 3c 6b 8f 54 97 45  |....v.^.'.<k.T.E|
000000d0  31 25 ff 23 90 a4 2a f2  db 78 b1 3b ca 21 f3 6b  |1%.#..*..x.;.!.k|
000000e0  dd fb 8e 53 c6 23 0d 35  c8 0a                    |...S.#.5..|
000000ea
```

The etcd key should be prefixed with `k8s:enc:aescbc:v1:key1`, which indicates the `aescbc` provider was used to encrypt the data with the `key1` encryption key.

### Deployments

In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```
kubectl run nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```
kubectl get pods -l run=nginx
```

> output

```
NAME                   READY     STATUS    RESTARTS   AGE
nginx-8586cf59-5sp9x   1/1       Running   0          1m
```

### Port Forwarding

In this section you will verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Retrieve the full name of the `nginx` pod:

```
POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:

```
kubectl port-forward $POD_NAME 8080:80
```

> output

```
Forwarding from 127.0.0.1:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:

```
curl --head http://127.0.0.1:8080
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.13.9
Date: Mon, 12 Mar 2018 10:29:42 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 20 Feb 2018 12:21:20 GMT
Connection: keep-alive
ETag: "5a8c12c0-264"
Accept-Ranges: bytes
```

Switch back to the previous terminal and stop the port forwarding to the `nginx` pod:

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs

In this section you will verify the ability to [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Print the `nginx` pod logs:

```
kubectl logs $POD_NAME
```

> output

```
127.0.0.1 - - [12/Mar/2018:10:29:42 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.53.1" "-"
```

### Exec

In this section you will verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:

```
kubectl exec -ti $POD_NAME -- nginx -v
```

> output

```
nginx version: nginx/1.13.9
```

### Services

In this section you will verify the ability to expose applications using a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Expose the `nginx` deployment using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:

```
kubectl expose deployment nginx --port 80 --type NodePort
```

> The LoadBalancer service type can not be used in vagrant environment, this is only supported  in  Cloud environment such as GCP/AWS/Azure. See [Type LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#type-loadbalancer) for more details.

Retrieve the node port assigned to the `nginx` service:

```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Make an HTTP request using the worker-${i} IP address and the `nginx` node port:

```
curl -I http://10.240.0.20:${NODE_PORT}
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.13.9
Date: Mon, 12 Mar 2018 10:33:47 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 20 Feb 2018 12:21:20 GMT
Connection: keep-alive
ETag: "5a8c12c0-264"
Accept-Ranges: bytes
```

## Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

```
vagrant destroy -f
```

> output

```
==> worker-2: Forcing shutdown of VM...
==> worker-2: Destroying VM and associated drives...
==> worker-1: Forcing shutdown of VM...
==> worker-1: Destroying VM and associated drives...
==> worker-0: Forcing shutdown of VM...
==> worker-0: Destroying VM and associated drives...
==> controller-2: Forcing shutdown of VM...
==> controller-2: Destroying VM and associated drives...
==> controller-1: Forcing shutdown of VM...
==> controller-1: Destroying VM and associated drives...
==> controller-0: Forcing shutdown of VM...
==> controller-0: Destroying VM and associated drives...
==> lb-0: Forcing shutdown of VM...
==> lb-0: Destroying VM and associated drives...
```
