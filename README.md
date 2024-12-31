# Deploy WordPress to Kubernetes (k8s)

Minimalist project to deploy Wordpress with MySQL to K8s (single node
is fine).

# Requirements

You need to have working K8s cluster (working `kubectl` command) - single node is fine. I tested
following setups:

1. Ubuntu 24.04 LTS with kubeadm: https://devopscube.com/setup-kubernetes-cluster-kubeadm/ - tested `kubeadm  1.30.8-1.1`
2. Fedora 41 with kubeadm: https://docs.fedoraproject.org/en-US/quick-docs/using-kubernetes-kubeadm/
   but plaese wait before RPM installations with DNF.

Please note that 1st Ubuntu guide uses Calico as overlay network, while 2nd Fedora guide prefers Flannel.
Sidenote: if you use Calico at scale you should read about "Calico route reflectors"
on https://www.reddit.com/r/RedditEng/comments/11xx5o0/you_broke_reddit_the_piday_outage/?rdt=51169
or https://www.tigera.io/blog/configuring-route-reflectors-in-calico/

## Setup k8s with kubeadm on Fedora 41

We will mostly follow guide on: https://docs.fedoraproject.org/en-US/quick-docs/using-kubernetes-kubeadm/

If you use ZRAM (swapping RAM to compressed RAM - it is not typo!) you have to follow:
```shell
sudo systemctl stop swap-create@zram0
sudo dnf remove zram-generator-defaults
sudo reboot now
```

If you use regular swap (in `/etc/fstab`) you should deactivate it
with:
```shell
sudo swapoff -a
```

And comment out your `swap` entry in `/etc/fstab`.

> [!WARNING]
> Recent  systemd reincarnations may automatically turn on swap even
> without `/etc/fstab`! In such case you need also to run (see
> https://askubuntu.com/a/1401070):

```shell
sudo systemctl mask --now swap.target
sudo reboot
```

Now we will follow official guide:
```shell
sudo systemctl disable --now firewalld
sudo dnf install iptables iproute-tc

sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

IMPORTANT! We will now divert from guide, because as of December 2024 default
packages use K8s 1.29 which is considered outdated.  To use version 1.32 we have to run:

```shell
v=1.32; sudo dnf install cri-o$v containernetworking-plugins
v=1.32; sudo dnf install kubernetes$v kubernetes$v-kubeadm kubernetes$v-client
# now we should lock package versions (from guide)
v=1.32; sudo dnf versionlock add "kubernetes*-$v.*" "cri-o$v.*"
# verify that there is no error like "No package found for ..."
```

Now continue with Fedora guide on
https://docs.fedoraproject.org/en-US/quick-docs/using-kubernetes-kubeadm/
using:

```shell
sudo systemctl enable --now crio
sudo kubeadm config images pull
sudo systemctl enable --now kubelet
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
# and follow instructions to configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

If you plan to add nodes later you should also store Join instructions, for example:
```shell
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join YOUR_IP:6443 --token xxxxx \
	--discovery-token-ca-cert-hash sha256:xxxxxx
```

Now I recommend to create alias `k` for `kubectl` command (it is also popular on YouTube
videos). Append to your `~/.bashrc`:
```shell
echo "alias k=kubectl" >> ~/.bashrc
```
And reload it with:
```shell
source ~/.bashrc
```

Now try some commands:
```shell
$ k get nodes

NAME       STATUS   ROLES           AGE    VERSION
fed-k8s2   Ready    control-plane   5m3s   v1.32.0

$ k get pods -A

NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-668d6bf9bc-54m24           1/1     Running   0          5m15s
kube-system   coredns-668d6bf9bc-gt4vs           1/1     Running   0          5m15s
kube-system   etcd-fed-k8s2                      1/1     Running   0          5m21s
kube-system   kube-apiserver-fed-k8s2            1/1     Running   0          5m21s
kube-system   kube-controller-manager-fed-k8s2   1/1     Running   0          5m21s
kube-system   kube-proxy-l4gxx                   1/1     Running   0          5m15s
kube-system   kube-scheduler-fed-k8s2            1/1     Running   0          5m21s
```

To deploy Applications to our K8s cluster we need to:
- enable running Workload on our only control-plane node (but for production
  you should use dedicated Worker nodes):
  ```shell
  k taint nodes --all node-role.kubernetes.io/control-plane-
  ```
- now we need to deploy Networking plugin - Fedora manual recommends Flannel using:
  ```shell
  cd
  curl -fLO https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
  k apply -f kube-flannel.yml
  ```
- now poll this command:
  ```shell
  k get po -A
  ```
- and verify that all Pods are READY `1/1` and STATUS `Running`

To understand K8s components please see https://kubernetes.io/docs/concepts/overview/components/
We will use:
- Deployment: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/ set of Pods
- Pods are collocated Container(s) - running on same Node, see https://kubernetes.io/docs/concepts/workloads/pods/
- Service: NodePort - exposes otherwise internal (available to other Pods only) Service
  on *all* nodes with same Port, see https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport

Finally we have to really test some application - for example using my
guide from https://github.com/hpaluch/hpaluch.github.io/wiki/Fedora-k8s-kind

Here is script `deploy-hello.sh` that creates Deployment called `hello-deploy`
with Pods and expose it as Service called `hello-svc` on all Nodes at unique Port.
Note: bash normally does not expand aliases in script - recommended way is to emulate
it as function.

```shell
set -xeuo pipefail
function k
{
	kubectl "$@"
}

k create deployment hello-deploy --image=registry.k8s.io/e2e-test-images/agnhost:2.39 -- /agnhost netexec --http-port=8080
k expose deployment hello-deploy --type=NodePort --name=hello-svc --port 8080
exit 0
```

Now poll Pods in default namespace using:
```shell
$ k get po

NAME                            READY   STATUS              RESTARTS   AGE
hello-deploy-594d4494b5-9x8z9   0/1     ContainerCreating   0          2m33s
```



If it stays in `ContainerCreating` for too long you likely hit problem
discussed on: https://devops.stackexchange.com/questions/14891/cni0-already-has-an-ip-address

You can verify if it is the case with command like (Pod suffix will be likely different):

```shell
$ k describe pod hello-deploy-594d4494b5-9x8z9

Failed to create pod sandbox:

rpc error: code = Unknown desc = failed to create pod network sandbox
k8s_hello-deploy-594d4494b5-9x8z9_default_1515a8fd-de5a-4154-a9a0-b5b36445779d_0(e25ee15aa8d2c81d27e3f838218a3d170a6f9b926261a72b2dd2957c35b730f8):
error adding pod default_hello-deploy-594d4494b5-9x8z9 to CNI network "cbr0":
plugin type="flannel" failed (add): failed to set bridge addr: "cni0" already
has an IP address different from 10.244.0.1/24
```

To fix it simply reboot Host (manually deleting bridge will likely cause another problems).

After reboot try `k get po -A` - after while all Pods should have READY set `1/1` and
RUNNING `1/1`.

To call your first Deployment and Service you need to find
- any Node IP address
- Exposed Service port

Therefore this service is of type `NodePort` - service is exposed on all Nodes
at same Port even when it runs only on subset of Node(s). See
https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport for more
details.

Node IP address (we have only one, but you can use any Node as target):
```shell
$ k get node -o wide | awk '{print $1,$2,$3,$4,$5,$6}' | column -t
NAME      STATUS  ROLES          AGE  VERSION  INTERNAL-IP
fed-k8s2  Ready   control-plane  28m  v1.32.0  192.168.122.93
```

So we can use IP address `192.168.122.93` in this case (Host IP)

To get Exposed port we need to fetch details on our service:
```shell
$ k get svc

NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-svc    NodePort    10.99.178.237   <none>        8080:31925/TCP   11m
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          29m
```

Our service is named `hello-svc` and exposes internal port 8080 to Host's port 31925.
So finally we can run on your host:
```shell
$ curl -fsS 192.168.122.93:31925 ;echo
NOW: 2024-12-31 14:41:54.993945717 +0000 UTC m=+352.320741824
```

Please see my K8s Kind guide for more tips on K8s services on
https://github.com/hpaluch/hpaluch.github.io/wiki/Fedora-k8s-kind

# WordPress Setup in k8s

This guide is combined from two sources:

1. https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/
2. https://kubeadm.org/working-with-persistent-volumes/

It is because 1st guide does not cover local creation of Persistent Volumes
(they are automagically provided by specific Public Cloud) - so we need to do
some additional work described on 2nd guide.

First define new storage class (it has global scope - no namespacing possible):
```shell
kubectl apply -f storage-class.yaml
```

Now create new namespace, say `wp1`:
```shell
kubectl create ns wp1
```

Now *important* : edit `volumes.yaml` and replace  `fed-k8s.example.com` with
your Worker Node name (where should be these volumes created) - you have to
replace it on 2 places in that file. You can find node names using `kubectl get
node` FIXME: I should find some better way to do this transparently...

K8s will automatically restrict running WordPress and MySQL pods only on node
where these Volumes really exist (so be careful!).

On your target node (which you specified in `volumes.yaml`) create directories for
Persistent Volumes with commands:
```shell
sudo mkdir -p /data/volumes/wordpress/www-data /data/volumes/wordpress/mariadb
sudo chmod a+rwxt /data/volumes/wordpress/www-data /data/volumes/wordpress/mariadb
```

Than and only then run `kustomize` in your namespace:
```shell
kubectl apply  -n wp1 -k ./
```

Watch pods creations until all are running:
```shell
$ kubectl get po -n wp1

NAME                               READY   STATUS    RESTARTS   AGE
wordpress-7d98f4479-mhnfz          1/1     Running   0          19m
wordpress-mysql-644fd69879-2k2qq   1/1     Running   0          19m
```

You should also check Persistent Volumes (pv) and Persistent Volume Claims (pvc). Example
when finished:
```shell
$ kubectl get pv -n wp1

NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
mariadb    20Gi       RWO            Retain           Bound    wp1/mariadb    local-storage   <unset>                          29m
www-data   20Gi       RWO            Retain           Bound    wp1/www-data   local-storage   <unset>                          29m

$ kubectl get pvc -n wp1

NAME       STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
mariadb    Bound    mariadb    20Gi       RWO            local-storage   <unset>                 29m
www-data   Bound    www-data   20Gi       RWO            local-storage   <unset>                 29m
```

Please note that reported CAPACITY is bogus in our case (you can use any number
in case of local folder storage).  However it is mandatory in case of "real"
distributed storage - for example https://openebs.io/

To find target port use following command (port defined in `wordpress-deployment.yaml`):
```shell
$ kubectl get svc -n wp1

NAME              TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
wordpress         NodePort    10.97.249.8   <none>        80:30001/TCP   20m
wordpress-mysql   ClusterIP   None          <none>        3306/TCP       20m
```

It is `30001` in our case.

In our case you can try (single node setup) following command to show WordPress main
page:
```shell
curl -fL `hostname -i`:30001
```

You should now be able to connect to same IP as reported with `hostname -i` from your
browser using `http://IP_ADDRESS:30001` and configure WordPress installation.

Note that unlike "regular" (from package or .tar.gz) installation there is no
Database setup page - it is already provided by k8s (as Service).

--hp
