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

## Monitoring


### Installing the Metrics Server

Metrics Server is a cluster-wide aggregator of resource usage data.

[Metrics Server]: https://github.com/kubernetes-incubator/metrics-server/

To install the metrics server, clone the git repo and then checkout the latest
revision. To list the available revisions use `git tag -l`.

    git clone https://github.com/kubernetes-incubator/metrics-server ./tmp/metrics-server
    cd ./tmp/metrics-server
    git checkout v0.3.6

Once the correct revision is checked out you can create the server in kubernetes
with the following command:

    kubectl create -f deploy/1.8+/



### Deploying the Dashboard.

The dashboard is a pretty interface with some monitoring in it.

The main instructions are
[here](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)

    kubectl apply -f kubernetes/dashboard.yaml
    kubectl apply -f kubernetes/dashboard-adminuser.yaml

    kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')

Kubectl has a proxy in it to make accessing the web interface of the cluster
possible in a secure way. Most of it is the API but services can quite often be
accessed in this way.

    kubectl proxy

- http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/


## Storage
### Deloying a Rook Ceph cluster

The instructions are at https://rook.io/docs/rook/v1.1/ceph-quickstart.html

In my experience the main thing that causes issues here is networking internally
to kubernetes.

The TL;DR for the rook quickstart is the following commands:


    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.1/cluster/examples/kubernetes/ceph/common.yaml
    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.1/cluster/examples/kubernetes/ceph/operator.yaml
    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.1/cluster/examples/kubernetes/ceph/cluster-test.yaml


Once the Ceph cluster is running you can access it using the `kubectl proxy`:

- http://localhost:8001/api/v1/namespaces/rook-ceph/services/https:rook-ceph-mgr-dashboard:8443/proxy/

I don't think you can log in correctly using the proxy due to the way the Ceph
Dashboard redirects on login so we can set it up with a
NodePort which is just a port-forward from the external interface of the
Kubernetes nodes to the service.

    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.1/cluster/examples/kubernetes/ceph/dashboard-external-https.yaml

You can then find the port using the following command. The port you want is the
one after `8443:` and is probably a high number:

    kubectl -n rook-ceph get service rook-ceph-mgr-dashboard-external-https

This port on all the kubernetes nodes will be forwarded to your service so we
can grab any external ip in the cluster. This command will show you all the IP
addresses available.

    kubectl -n rook-ceph get nodes -o wide

So combine the ip address and port and using https you should get a url similar
to https://192.168.121.53:31158

The default username is `admin` and the password is randomly generated and stored in
Kubernetes. It can be extracted using the following:

    kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo


### Rook Ceph Toolbox

To check the status of the Ceph cluster you can deploy the
[rook-ceph-toolbox](https://rook.io/docs/rook/master/ceph-toolbox.html).
Remember to change to the page that matches the version you have installed.

Here is a TL;DR for rook-ceph-toolbox:

    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.1/cluster/examples/kubernetes/ceph/toolbox.yaml
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
applications.

    wget -O ./tmp/storageclass.yaml https://raw.githubusercontent.com/rook/rook/release-1.1/cluster/examples/kubernetes/ceph/csi/rbd/storageclass.yaml
    sed -i 's|csi.storage.k8s.io/fstype: ext4|csi.storage.k8s.io/fstype: xfs|g' ./tmp/storageclass.yaml
    kubectl create -f ./tmp/storageclass.yaml


Some sample apps are provided with rook for testing.

    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.1/cluster/examples/kubernetes/mysql.yaml
    kubectl create -f https://raw.githubusercontent.com/rook/rook/release-1.1/cluster/examples/kubernetes/wordpress.yaml


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


