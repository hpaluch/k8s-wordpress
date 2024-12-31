# Deploy WordPress to Kubernetes (k8s)

Minimalist project to deploy Wordpress with MySQL to K8s (single node
is fine).

# Requirements

You need to have working K8s cluster (working `kubectl` command) - single node is fine. I tested
following setups:

1. Ubuntu 24.04 LTS with kubeadm: https://devopscube.com/setup-kubernetes-cluster-kubeadm/ - tested `kubeadm  1.30.8-1.1`
2. Fedora 41 with kubeadm: https://docs.fedoraproject.org/en-US/quick-docs/using-kubernetes-kubeadm/ - tested
   `kubernetes-1.29.11-1.fc41` (switched to 1.32).

Please note that 1st Ubuntu guide uses Calico as overlay network, while Fedora guide prefers Flannel.
Sidenote: if you use Calico at scale you should read about "Calico route reflectors"
on https://www.reddit.com/r/RedditEng/comments/11xx5o0/you_broke_reddit_the_piday_outage/?rdt=51169
or https://www.tigera.io/blog/configuring-route-reflectors-in-calico/

So on Fedora,  using version 1.32 we have to run:
```shell
v=1.32; sudo dnf install --allowerasing kubernetes$v kubernetes$v-kubeadm kubernetes$v-client
sudo kubeadm config images pull
# if you get error: "found multiple CRI endpoints on the host", try:
sudo dnf remove containerd
# now follow above Fedora guide
```

Fedora warning: I had issues described
on: https://devops.stackexchange.com/questions/14891/cni0-already-has-an-ip-address
I had to delete bridge using `sudo ip link delete cni0 type bridge` and reboot to avoid
core-dns flapping...

In case of Fedora also disable "stub DNS" of useless systemd-resolved as described at the
bottom of Fedora guide.

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
