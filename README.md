# Vagrant -> Ansible -> Kubernetes


## Setup

Vagrant does most of the initial setup here. It uses an Ansible provisioner to
get a very basic install following the instructions from official docs starting
at [Installing kubeadm] and going as far as [Creating a single control-plane
cluster with kubeadm]. It uses the [Docker Container Runtime Interface] (CRI)
and the [Calico Container Network Interface] (CNI)


[Installing kubeadm]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
[Creating a single control-plane cluster with kubeadm]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
[Docker Container Runtime Interface]: https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker
[Calico Container Network Interface]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network

There are multiple assumptions for this walkthough based on my personal setup. You may need to adjust to suit.

- Fedora 29-31
- vagrant from the fedora repos 
- vagrant-libvirt from fedora repos

### Set up access to NFS shares
To allow the users in the `wheel` group to configure NFS shares for vagrant
create the file `/etc/sudoers.d/vagrant-syncedfolders` with the following
content.

```
sudo vim /etc/sudoers.d/vagrant-syncedfolders
```

```
Cmnd_Alias VAGRANT_EXPORTS_CHOWN = /bin/chown 0\:0 /tmp/*
Cmnd_Alias VAGRANT_EXPORTS_MV = /bin/mv -f /tmp/* /etc/exports
Cmnd_Alias VAGRANT_NFSD_CHECK = /usr/bin/systemctl status --no-pager nfs-server.service
Cmnd_Alias VAGRANT_NFSD_START = /usr/bin/systemctl start nfs-server.service
Cmnd_Alias VAGRANT_NFSD_APPLY = /usr/sbin/exportfs -ar
%wheel ALL=(root) NOPASSWD: VAGRANT_EXPORTS_CHOWN, VAGRANT_EXPORTS_MV, VAGRANT_NFSD_CHECK, VAGRANT_NFSD_START, VAGRANT_NFSD_APPLY
```

```
sudo chmod 644 /etc/sudoers.d/vagrant-syncedfolders
sudo chown root.root /etc/sudoers.d/vagrant-syncedfolders
```

### Polkit access for `wheel` users to manage libvirt.

Create `/etc/polkit-1/rules.d/80-libvirt-manage.rules` with the following
content. This will allow people in the `wheel` group to use libvirt without
requiring a password.

Once again Debian users need to change the `wheel` group to `vagrant`
```
sudo vim /etc/polkit-1/rules.d/80-libvirt-manage.rules
```

```
polkit.addRule(function(action, subject) {
    if (action.id == "org.libvirt.unix.manage" && subject.local && subject.active && subject.isInGroup("wheel")) {
    return polkit.Result.YES;
    }
});
```


```
sudo chmod 644 /etc/polkit-1/rules.d/80-libvirt-manage.rules
sudo chown root.root /etc/polkit-1/rules.d/80-libvirt-manage.rules
```

### Firewall access for NFS


Depending on your setup you may need to allow NFS through your firewall for libvirt.

    firewall-cmd  --permanent --zone=libvirt --add-service=nfs
    firewall-cmd  --permanent --zone=libvirt --add-service=mountd
    firewall-cmd  --permanent --zone=libvirt --add-service=rpc-bind
    firewall-cmd  --permanent --zone=libvirt --add-port=2049/tcp
    firewall-cmd  --permanent --zone=libvirt --add-port=2049/udp
    firewall-cmd --reload 

## Create the cluster with Vagrant
Run Vagrant to bring up the boxes.

    vagrant up

There is a race condition where sometimes the workers try to join before the
join command has been exported from the master. If that is the case then
re-provision using vagrant.

    vagrant provision

When all is complete there should be a `kubeadmin.conf` file in the same
directory as the `Vagrantfile`.  This is the authentication config for
`kubectl`:

    export KUBECONFIG=${PWD}/tmp/kubeadmin.conf

Next see how the kubernetes cluster is running:

    kubectl get nodes
    kubectl get all --all-namespaces


## Checking if your cluster is correctly running.

See the [README for
sonobouy](https://github.com/vmware-tanzu/sonobuoy/blob/master/README.md)

Once you have sonobouy installed the way to check is just to run it. Running it
with `--mode quick` is fast and only runs 1 test to quickly validate a working
cluster.

    sonobuoy  run --wait --mode quick

When it is complete you can retrieve the results and display them.

    results=$(sonobuoy retrieve)
    sonobuoy results $results

The output should look similar this with 1 test passed and a lot of skipped:

    Plugin: e2e
    Status: passed
    Total: 4897
    Passed: 1
    Failed: 0
    Skipped: 4896

## Management

### Deploying the Dashboard.

The dashboard is a pretty interface with some monitoring in it.

The main instructions are
[here](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml

Create a user and make it a member of the role cluster-admin (You should use something more secure in production).

    kubectl create serviceaccount -n kubernetes-dashboard dashboard-admin-sa 
    kubectl create clusterrolebinding dashboard-admin-sa --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin-sa 

Get the secret token to access the dashboard which is required to log into the dashboard.
    
    kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep dashboard-admin-sa | awk '{print $1}')

Kubectl has a proxy in it to make accessing the web interface of the cluster
possible in a secure way. Most of it is the API but services can quite often be
accessed in this way.

    kubectl proxy

- http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

The second way is to create a port forward. This way often works best as the application may be using redirects that break using `kubectl proxy`. This command requires the node to have 'socat' installed.

    kubectl --namespace kubernetes-dashboard port-forward svc/kubernetes-dashboard 8443:443
    
- https://127.0.0.1:8443/
    
    

## Monitoring
The original application for this was named **heapster** but it has now been deprecated. This is being replaced by a new service named **metrics-server** but like all things in k8s this is interchangable with other solutions such as **prometheus**.

Provided below are installation instructions for both **metrics-server** and **Prometheus**. You can only choose one.

### Prometheus

[Prometheus]: https://prometheus.io/
[kube-prometheus]: https://github.com/coreos/kube-prometheus


The [kube-prometheus] project makes [prometheus] run nativly on kubernetes.  
These instructions are based on the quickstart from the kube-prometheus 
[README](https://github.com/coreos/kube-prometheus/blob/master/README.md) 
which you should read for further details.


Download and install kube-prometheus

    git clone https://github.com/coreos/kube-prometheus tmp/kube-prometheus
    kubectl create -f tmp/kube-prometheus/manifests/setup
    until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
    kubectl create -f tmp/kube-prometheus/manifests/
    
You can then access it from you local machine using a portforward 

    kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090
    
Open you browser to http://127.0.0.1:9090

You can access the rest of the services in a similar way.

- `kubectl --namespace monitoring port-forward svc/grafana 3000` http://127.0.0.1:3000
- `kubectl --namespace monitoring port-forward service/alertmanager-main 9093` http://127.0.0.1:9093

### Metrics Server. 

[Metrics Server]: https://github.com/kubernetes-sigs/metrics-server

[Metrics Server] is a cluster-wide aggregator of resource usage data.

To install the metrics server, clone the git repo and then checkout the latest
revision. To list the available revisions use `git tag -l`.

    git clone https://github.com/kubernetes-sigs/metrics-server ./tmp/metrics-server
    cd ./tmp/metrics-server
    git checkout v0.3.6

Once the correct revision is checked out you can create the server in kubernetes
with the following command:

    kubectl create -f deploy/1.8+/



## Storage
### Deloying a Rook Ceph cluster

The instructions are at https://rook.io/docs/rook/v1.1/ceph-quickstart.html

In my experience the main thing that causes issues here is networking internally
to kubernetes.

The TL;DR for the rook quickstart is the following commands:


    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.2/cluster/examples/kubernetes/ceph/common.yaml
    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.2/cluster/examples/kubernetes/ceph/operator.yaml
    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.2/cluster/examples/kubernetes/ceph/cluster-test.yaml

These take a little while to get bootstrapped. You can check the state of the pods using:

    kubectl  get all -n rook-ceph

Once the Ceph cluster is running you can access it using the `kubectl proxy`:

- http://localhost:8001/api/v1/namespaces/rook-ceph/services/https:rook-ceph-mgr-dashboard:8443/proxy/

This is an example of application not working correctly using the proxy due to the way the Ceph
Dashboard redirects on login we should use the port-forward method instead

    kubectl --namespace rook-ceph port-forward service/rook-ceph-mgr-dashboard 8443
    
- http://127.0.0.1:8443

The default username is `admin` and the password is randomly generated and stored in
Kubernetes. It can be extracted using the following:

    kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo


### Rook Ceph Toolbox
[rook-ceph-toolbox]: https://rook.io/docs/rook/master/ceph-toolbox.html

To check the status of the Ceph cluster you can deploy the [rook-ceph-toolbox].
Remember to change to the page that matches the version you have installed.

Here is a TL;DR for rook-ceph-toolbox:

    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.2/cluster/examples/kubernetes/ceph/toolbox.yaml
    kubectl -n rook-ceph get pod -l "app=rook-ceph-tools"
    kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash

The quick commands to get an overview are:

    ceph status
    ceph osd status
    ceph df
    rados df

Once you are done you can delete the toolbox:

    kubectl -n rook-ceph delete deployment rook-ceph-tools


### Creating Ceph block devices

The following will create a `rook-ceph-block` StorageClass which can be used by
applications. These will be ext4 by default so if you want to change it then download the yamls and edit before creating. 

There are 2 different yamls available at https://github.com/rook/rook/tree/release-1.2/cluster/examples/kubernetes/ceph/csi/rbd `storageclass.yaml` which requres 3 replicas and `storageclass-test.yaml` which is designed for testing on a single node. Pick one and create it.

    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.2/cluster/examples/kubernetes/ceph/csi/rbd/storageclass-test.yaml
    
or

    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.2/cluster/examples/kubernetes/ceph/csi/rbd/storageclass.yaml


Some sample apps are provided with rook for testing.

    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.2/cluster/examples/kubernetes/mysql.yaml
    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.2/cluster/examples/kubernetes/wordpress.yaml


You can see the state of the apps using `kubectl get all` as these will be
deployed in the default namespace.

    $ kubectl get all

    NAME                                  READY   STATUS              RESTARTS   AGE
    pod/wordpress-5b886cf59b-97rfl        0/1     ContainerCreating   0          114s
    pod/wordpress-mysql-b9ddd6d4c-2sklp   1/1     Running             0          115s


    NAME                      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
    service/kubernetes        ClusterIP      10.96.0.1       <none>        443/TCP        126m
    service/wordpress         LoadBalancer   10.101.90.166   <pending>     80:30118/TCP   114s
    service/wordpress-mysql   ClusterIP      None            <none>        3306/TCP       116s


    NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/wordpress         0/1     1            0           114s
    deployment.apps/wordpress-mysql   1/1     1            1           115s

    NAME                                        DESIRED   CURRENT   READY   AGE
    replicaset.apps/wordpress-5b886cf59b        1         1         0       114s
    replicaset.apps/wordpress-mysql-b9ddd6d4c   1         1         1       115s


When everything `Running` you can access the WordPress install via `kubectl proxy`:

http://localhost:8001/api/v1/namespaces/default/services/http:wordpress:/proxy/


You may have notices with the WordPress example that the EXTERNAL-IP of the
`service/wordpress` is sitting in `<pending>` state. This is because it is setup
to use a load-balancer. The next section will cover creating a load balancer.


## Connecting from the outside

### Creating the LoadBalancer

Kubernetes doesn't supply a LoadBalancer for clusters that aren't running on
cloud platforms. The implementations they do have expect to be able to speak to
a cloud platform load-balancer. Luckily for us there is a project [MetalLB]
which provides just what we need.

There are a bunch of caveats with MetalLB which affect cloud providers and some
of the CNIs so checkout the [MetalLB compatibility docs] if you are following
these instructions but have made some changes. In this example the decision
earlier to use Calico works fine because we wont be using BGP.


Installation is a simple apply of a manifest:

    kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.1/manifests/metallb.yaml


MetalLB can use ARP or BGP to manage the address space. The simpler of the two
and one that suits this example is ARP.

We need to create a manifest and give MetalLB some addresses to manage. Create
the following file as ./tmp/metallb_address-pool.yaml. The address range is the
setup in the `Vagrantfile` as the private network `metallb`. We dont need to
actually bind the ipaddresses to the interfaces in the nodes because metallb
will respond to arp requests and do this all at layer2.

    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      namespace: metallb-system
      name: config
    data:
      config: |
        address-pools:
        - name: default
          protocol: layer2
          addresses:
          - 192.168.200.100-192.168.200.250

Next apply it using kubectl

     kubectl apply -f tmp/metallb_address-pool.yaml


Hopefully if everything has gone to plan the wordpress service should now have
an exteral IP address that can be accessed from your workstation.

You can check this by getting the services:

    $ kubectl get services -o wide
    NAME              TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE    SELECTOR
    kubernetes        ClusterIP      10.96.0.1     <none>           443/TCP        121m   <none>
    wordpress         LoadBalancer   10.98.86.15   172.28.128.101   80:30910/TCP   34m    app=wordpress,tier=frontend
    wordpress-mysql   ClusterIP      None          <none>           3306/TCP       92m    app=wordpress,tier=mysql


In this instance NAT has been used but it should be possible to use macvtap to
make this accessable from other machines in the network.


[MetalLB]: https://metallb.universe.tf
[MetalLB compatibility docs]: https://metallb.universe.tf/installation/



