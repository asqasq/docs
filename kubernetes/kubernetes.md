# Kubernetes

## Resources


### Installation


## Applications
WebShop on Kubernetes [1](https://gist.github.com/TRoetz/763c280f8216f7ece56310fb68788de3) [2](https://www.mirantis.com/blog/how-install-kubernetes-kubeadm/)

### Examples

[Sock shop microservice on Kubernetes](https://github.com/microservices-demo/microservices-demo)


## Issues:
OOM kill of coredns on Ubuntu, if systemd-resolved runs and /etc/resolv.conf contains 127.0.0.53 as nameserver: [1](https://github.com/kubernetes/kops/issues/5652) [2](https://github.com/kubernetes/kubeadm/issues/1037) [3](https://kubernetes.io/docs/setup/independent/kubelet-integration/)
[4](https://www.ctrl.blog/entry/resolvconf-tutorial)
Solve OOM kill of coredns by disabling systemd-resolved on Ubuntu 18.04 [4](https://askubuntu.com/questions/907246/how-to-disable-systemd-resolved-in-ubuntu)

    sudo systemctl disable systemd-resolved.service
    sudo systemctl stop systemd-resolved

Also disable the management of /etc/resolv.conf by the NetworkManager by creating the file
/etc/NetworkManager/conf.d/no-dns.conf with this content:

    [main]
    dns=none

## Extra
[Visualize app](https://www.weave.works/docs/scope/latest/installing/#k8s)

## Commands
### All nodes

As root, enter the following commands:

    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    aptitude update
    apt-cache policy docker-ce
    aptitude install docker-ce
    aptitude install apt-transport-https
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
    aptitude update
    aptitude install kubelet kubeadm kubernetes-cni

### On the Kubernetes master node

As root, enter the following commands (kubernetesuser is a regular Unix username or your perosnal user id):

    kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.144.144.13
    su kubernetesuser
    mkdir -p $HOME/.kube
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    chown $(id -u):$(id -g) $HOME/.kube/config
    exit
    kubectl apply -f  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

### On the kubernetes slave nodes

Join the cluster:

    kubeadm join 192.144.144.13:6443 --token 123456.kjdjdhj --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef


### Again on the Kubernetes master node
Install the flannel network:

    kubectl get nodes
    kubectl apply -f  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

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
