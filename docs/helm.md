
Install

    wget https://get.helm.sh/helm-v3.3.0-rc.1-linux-amd64.tar.gz
    tar -xvf helm-v3.3.0-rc.1-linux-amd64.tar.gz
    mv linux-amd64/helm bin/
    rm -rf linux-amd64/ helm-v3.3.0-rc.1-linux-amd64.tar.gz
    helm version

Add a repo and update it

    helm repo add stable https://kubernetes-charts.storage.googleapis.com/
    helm repo update
    helm search repo stable

Install metallb

    helm show all stable/metallb
    helm install --create-namespace --namespace metallb --set arpAddresses=192.168.200.100-192.168.200.250 metallb stable/metallb

Show the installations
    helm ls



