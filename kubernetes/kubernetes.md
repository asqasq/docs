# Kubernetes

## Scripts and configuration files used to install Kubernetes
All configuration files and scripts used to install Kubernetes can be found here:
https://github.com/asqasq/serverless/tree/master/kubernetes/deploy


## Issues to be solved *before* installing Kubernetes:
OOM kill of coredns on Ubuntu, if systemd-resolved runs and /etc/resolv.conf contains 127.0.0.53 as nameserver: [1](https://github.com/kubernetes/kops/issues/5652) [2](https://github.com/kubernetes/kubeadm/issues/1037) [3](https://kubernetes.io/docs/setup/independent/kubelet-integration/)
[4](https://www.ctrl.blog/entry/resolvconf-tutorial)
Solve OOM kill of coredns by disabling systemd-resolved on Ubuntu 18.04 [4](https://askubuntu.com/questions/907246/how-to-disable-systemd-resolved-in-ubuntu)

    sudo systemctl disable systemd-resolved.service
    sudo systemctl stop systemd-resolved

Also disable the management of /etc/resolv.conf by the NetworkManager by creating the file
/etc/NetworkManager/conf.d/no-dns.conf with this content:

    [main]
    dns=none

### *Don't forget to set a DNS IP address*!
After disabling systemd-resolved, it is necessary to manually set the IP address of a DNS server.
Edit the /etc/resolved.conf file and set the DNS IP:

    nameserver 10.11.12.13
    search your.domain.tld

## Commands
### All nodes

docker.io became alive again and can be used instead of docker-ce (which does not have packages for Ubuntu eoan),
so we can install docker.io and get a up-to-date version, which works with Kubernetes.

As root, enter the following commands:

    aptitude install docker.io
    aptitude install apt-transport-https
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
    aptitude update
    aptitude install kubelet kubeadm kubernetes-cni

For Ubuntu 18.04, I used to install docker-ce the following way (not needed anymore, if you install docker.io):

    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    aptitude update
    apt-cache policy docker-ce
    aptitude install docker-ce


### On the Kubernetes master node

The default configuration uses 10.244.0.0/16 for POD networks and 10.96.0.1 for the service network.
If this conflicts with IPs used on hardware network interfaces, change the range with --pod-network-cidr
and --service-cidr as well as in the kube-flannel.yaml configuration file (see below).

As root, enter the following commands (kubernetesuser is a regular Unix username or your perosnal user id):

    kubeadm init --pod-network-cidr=192.168.128.0/20 --service-cidr=192.168.192.0/20 \
        --apiserver-advertise-address=192.144.144.13
    su kubernetesuser
    mkdir -p $HOME/.kube
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    chown $(id -u):$(id -g) $HOME/.kube/config
    exit

### On the kubernetes slave nodes

Join the cluster:

    kubeadm join 192.144.144.13:6443 --token 123456.kjdjdhj \
        --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef


### Again on the Kubernetes master node
Install the flannel network:

    kubectl get nodes
    kubectl apply -f  \
        https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

### Optional

    kubectl taint nodes --all
    kubectl taint nodes --all dedicated=

### Install SSL certificate
To run docker containers over SSL it is necessary that the container has an SSL certificate.
All clients need to have the public key available.

    mkdir /root/certs
    cd /root/certs
    openssl req -newkey rsa:4096 -nodes -sha256 -keyout domain.key -x509 -days 3650 -out domain.crt
    for k in kubernetesnodes; do
        scp certs/domain.crt $k:/etc/docker/certs.d/dockerservernode\:dockerserverport/ca.crt;
    done

### Run a service in a docker container, which exports a socket over SSL

Then on dockerservernode, start the container the following way:

    docker run -d -p dockerserverport:dockerserverport --restart always \
    -v /root/certs:/certs \
    -e REGISTRY_HTTP_ADDR=0.0.0.0:dockerserverport \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
    --name bla bla:2

### Run a docker image registry on the kubernetes cluster on a specific node (not ideal, but works) on port 5000:

    docker run -d -p 5000:5000 --restart always \
    -v /root/certs:/certs \
    -v /root/registrydata:/var/lib/registry \
    -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
    --name registry registry:2




## Upgrading the cluster
There is an official [documentation](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade-1-14/) on hot to upgrade the cluster.

The simplified variant of just upgrading everything and restarting the cluster (only for experimental clusters, where all physical host can be rebooted and availability of Kubernetes is not necessary) follows below:

### Upgrade the binaries on _all nodes_

    apt update
    apt upgrade

Then check for the currently installed version (binary version, not yet the version the cluster runs at):

    kubeadm version

### Upgrade the cluster on the _master node only_

    sudo kubeadm upgrade plan
    sudo kubeadm upgrade apply v1.15.0

### Upgrade config on _all nodes_

In this simplified version, all node configs get upgraded simultaneously without putting the nodes in cordon state and therefore without draining work assigned to the nodes.

    sudo kubeadm upgrade node config --kubelet-version v1.15.0

Finally restart the nodes.

These simplified instructions work fine for an experimental cluster, but not for a cluster in production!


## Performance monitoring
Prometheus can be deployed directly using helm to the kubernetes cluster.

Forward three ports to the local machine, so that you can use http://localhost:<forwarded port>

    kubectl port-forward -n monitoring prometheus-prometheus-operator-prometheus-0 9090

Grafana:
    kubectl port-forward $(kubectl get  pods --selector=app=grafana -n  monitoring --output=jsonpath="{.items..metadata.name}") -n monitoring  3000

Alerts:
    kubectl port-forward -n monitoring alertmanager-prometheus-operator-alertmanager-0 9093


## Resources

### Persistent volumes/storage
[Persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
[Configure Pod to use persistent storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)
[Example for persistent storage](https://developer.ibm.com/tutorials/add-nfs-provisioner-to-icp/)

### RDMA support within containers / PODs
[RDMA support](https://medium.com/@Alibaba_Cloud/using-rdma-on-container-service-for-kubernetes-c7a4484c22b5)
[2](https://www.alibabacloud.com/blog/using-rdma-on-container-service-for-kubernetes_594462?spm=a2c41.12560487.0.0)

### Installation


## Applications
WebShop on Kubernetes [1](https://gist.github.com/TRoetz/763c280f8216f7ece56310fb68788de3) [2](https://www.mirantis.com/blog/how-install-kubernetes-kubeadm/)

### Examples

[Sock shop microservice on Kubernetes](https://github.com/microservices-demo/microservices-demo)


## Extra
[Visualize app](https://www.weave.works/docs/scope/latest/installing/#k8s)

