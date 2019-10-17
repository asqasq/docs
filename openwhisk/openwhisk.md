# OpenWhisk

## Scripts and configuration files used to install OpenWhisk as described here
All configuration files and scripts used to install OpenWhisk on Kubernetes can be found here:
https://github.com/asqasq/serverless/tree/master/openwhisk/deploy

## Commands
### Install Helm
Download helm from the [helm](https://github.com/helm/helm) homepage
(for example [release 2.12.13](https://github.com/helm/helm/releases/tag/v2.12.3)) and
install it as described also [here](https://github.com/apache/incubator-openwhisk-deploy-kube/blob/master/docs/helm.md)
using the following commands on the OpenWhisk master node:

    wget https://storage.googleapis.com/kubernetes-helm/helm-v2.12.3-linux-amd64.tar.gz
    tar xvzf helm-v2.12.3-linux-amd64.tar.gz
    cd linux-amd64/
    ./helm init
    sudo cp ./helm /usr/local/bin
    cd ../
    rm -rf linux-amd64/

Verify that the tiller service is running:

    kubectl get pods -n kube-system

Create clusterrolebinding:

    kubectl create clusterrolebinding tiller-cluster-admin --clusterrole=cluster-admin \
      --serviceaccount=kube-system:default

Update the helm repo

    helm repo update


### Download the wsk client
To manage the OpenWhisk cluster, install, modify and execute functions,
it is necessary to download the [wsk client](https://github.com/apache/incubator-openwhisk-cli).
There are [binary releases](https://github.com/apache/incubator-openwhisk-cli/releases) available
for download.

        mkdir -p ~/openwhisk/wsk_client
        cd ~/openwhisk/wsk_client
        wget https://github.com/apache/incubator-openwhisk-cli/releases/download/latest/OpenWhisk_CLI-latest-linux-amd64.tgz
        tar xvzf OpenWhisk_CLI-latest-linux-amd64.tgz
        sudo cp wsk /usr/local/bin

Then configure the wsk client with an API host and an authentication key. Without
configuring users, use the default key (not for production systems):

        wsk property set --apihost node01:31001
        wsk property set --auth \ 
            23bc46b1-71f6-4ed5-8c54-816aa4f8c502:123zO3xZCLrMN6v2BKK1dXYFpXlPkccOFqm12CdAsMgRU4VrNZ9lyGVCGuMDGIwP

### Configure PersistentVolume via NFS client
To serve OpenWhisk's PersistentVolumeClaims (PVC), we have to provide PersistentVolumes (PV). One
possible way is a service, whch mounts an external NFS server export and which provisions
PersistentVolumes based on the external NFS mount. Such an NFS client provisioner can be installed
as [Helm chart](https://github.com/helm/charts/tree/master/stable/nfs-client-provisioner). A small
configuration file specifies the NFS server and exported mount path with some additional options.
This is an example configuration file nfs_client.yaml:

        replicaCount: 1
        strategyType: Recreate

        nfs:
          server: nfsservername.domain.nil
          path: /data/kubernetes/pv
          mountOptions:

        # For creating the StorageClass automatically:
        storageClass:
          create: true
          defaultClass: false

          # Set a StorageClass name
          name: nfs-client

          # Allow volume to be expanded dynamically
          allowVolumeExpansion: true

          # Method used to reclaim an obsoleted volume
          reclaimPolicy: Delete

          # When set to false your PVs will not be archived by the provisioner upon deletion of the PVC.
          archiveOnDelete: true

          provisionerName: nfs-client-provisioner-from-nfsserver

        ## For RBAC support:
        rbac:
          create: true
        podSecurityPolicy:
          enabled: false

        serviceAccount:
          # Specifies whether a ServiceAccount should be created
          create: true

          # The name of the ServiceAccount to use.
          # If not set and create is true, a name is generated using the fullname template
          name:

The PersistenVolume service can be created by installing the Helm chart (linked above) with
this command:

        helm install stable/nfs-client-provisioner --name nfs-client \
            --namespace nfs-client -f nfs_client.yaml

The PersistentVoulme can be tested/used by creating a PersistentVolumeClaim with the following
configurationj stored in nfs-test-claim.yaml:

        kind: PersistentVolumeClaim
        apiVersion: v1
        metadata:
          name: nfsasq
        spec:
          accessModes:
            - ReadWriteMany
          resources:
            requests:
              storage: 1Mi
          storageClassName: nfs-client

The first command creates the PVC and the second and third commands inspect, if the creation succeeded:

        kubectl create -f nfs-test-claim.yaml
        kubectl get pvc
        kubectl describe pvc nfsasq

To delete the PVC, use the following command:

        kubectl delete -f nfs-test-claim.yaml


### Creating a cluster configuration file
The cluster needs to be configured and set up according to this configuration. A small
YAML file as below configures the API host name and port as well as the method to
start containers. In order to have kubernetes host and domain names created for
function docker containers, it is needed to start function containers using the
kubernetes mechanism, instead of the plain docker mechanism. To configure the invoker
to start function containers using kubernetes. For example clusterconf.yaml:

        whisk:
          ingress:
            apiHostName: node01
            apiHostPort: 31001
            type: NodePort
          loadbalancer:
            invokerUserMemory: "512m"
        nginx:
          httpsNodePort: 31001

        invoker:
          containerFactory:
            impl: "kubernetes"
            kubernetes:
              replicaCount: 2
              agent:
                enabled: true

        k8s:
          persistence:
            hasDefaultStorageClass: false
            explicitStorageClass: nfs-client


If you don't need or don't want persistence, you can turn it of by setting enabled to false
in the cluster configuration file:

        k8s:
          persistence:
            enabled: false

More configuration variables can be set and are listed in [this file](https://github.com/apache/openwhisk/blob/master/ansible/group_vars/all).

### Label invoker nodes
First of all, label all nodes, which should operate as invokers, with the
invoker label using the following command:

        kubectl label nodes node01 noe02 node03 node04 openwhisk-role=invoker

Or alternatively, if all nodes should be labelled:

        kubectl label nodes --all openwhisk-role=invoker


### Install OpenWhisk with Helm
The [OpenWhisk repository](https://github.com/apache/incubator-openwhisk-deploy-kube) needs to be cloned.

        git clone https://github.com/apache/incubator-openwhisk-deploy-kube.git
        
Then, install OpenWhisk with Helm and apply the configuration file created above:

        helm install incubator-openwhisk-deploy-kube/helm/openwhisk --namespace=openwhisk \
            --name=owdev -f clusterconf.yaml

### Checking, if OpenWhisk is running

        helm status owdev
        helm get owdev

### Uninstalling OpenWhisk completely
Should you want to remove OpenWhisk completely and remove its namespace from Kubernetes,
execute the following command:

        helm del --purge owdev


## Administration
OpenWhisk uses wskadmin to administer the OpenWhisk cluster. While earlier it was a standalon
program, it now runs as a POD on the Kubernetes cluster within the OpenWhisk namespace.
The [deployment page](https://github.com/apache/incubator-openwhisk-deploy-kube) shows how
to execute wskadmi and the
[Administrative Operations](https://github.com/apache/incubator-openwhisk/tree/master/tools/admin)
page explains the available commands.

To create a new user in a separate namespace, execute the following command:

        kubectl -n openwhisk -ti exec owdev-wskadmin -- wskadmin user create blabla -ns ns01

Then set the new AUTH key returned by the command above in the ~/.wskprops file.

To list the available users belonging to a given namespace, use the following command:

        kubectl -n openwhisk -ti exec owdev-wskadmin -- wskadmin user list ns01 -a

To increase the invocation limit per minute, execute the following command:

        kubectl -n openwhisk -ti exec owdev-wskadmin -- wskadmin limits get ns01
        kubectl -n openwhisk -ti exec owdev-wskadmin -- wskadmin limits set ns01 --invocationsPerMinute 1000

To increase the number of concurrent invocations, execute the following command:

        kubectl -n openwhisk -ti exec owdev-wskadmin -- wskadmin limits set ns01 --concurrentInvocations 1000

To increase the number of trigger fired per minute, execute the following command:

        kubectl -n openwhisk -ti exec owdev-wskadmin -- wskadmin limits set ns01 --firesPerMinute 1000

Setting the values to 999999 disables the thorrtling completely.


## Changing values and applying them
Sometimes it is necessary to change or add more customized values to a runnning OpenWhisk installaion.
An example could be that the maximum allowed memory for action should be increased, while OpenWhisk
is already installed an running (with a too low max value).
The new values can be prepared in the cluster configuration yaml file and applied the following way:

        helm upgrade --reuse-values -f clusterconf_new.yaml \
             owdev incubator-openwhisk-deploy-kube/helm/openwhisk

See also [this discussion](https://stackoverflow.com/questions/48927233/updating-kubernetes-helm-values)
and the [list of values](
https://github.com/apache/openwhisk/blob/d85a722b66980af8e7dd9a3d61918020ba45bcea/common/scala/src/main/resources/application.conf#L302), which can be adjusted.

## Measuring performance

This [tool](https://github.com/IBM/overhead) measures the performance of OpenWhisk.

## Advanced debugging

OpenWhisk containers can be run directly in docker and it is possible to interact with the
container using the REST API. Here is an [article](https://medium.com/openwhisk/advanced-debugging-of-openwhisk-actions-518414636932) explaining more details.

## Resources

### Installation
[OpenWhisk on Kubernetes](https://github.com/apache/incubator-openwhisk-deploy-kube)
[OpenWhisk on Kubernetes](https://github.com/IBM/OpenWhisk-on-Kubernetes)

### Configuration
[Use Kubernetes to start user functions instead of docker directly:](https://github.com/apache/incubator-openwhisk-deploy-kube/blob/master/docs/configurationChoices.md#invoker-container-factory)
[Values/keys:](https://github.com/apache/incubator-openwhisk-deploy-kube/blob/735e576f356500424d1a1e720064ca3182da644d/helm/openwhisk/values.yaml#L172)
[Alternatives for certain implementations:](https://github.com/apache/incubator-openwhisk/blob/master/common/scala/src/main/resources/reference.conf#L6-#L15)
[wskadmin](https://github.com/redhat-developer-demos/faas-tutorial/issues/16)
[Configuration choices](https://github.com/apache/incubator-openwhisk-deploy-kube/blob/master/docs/configurationChoices.md)

[Dynamic storage/PersistentVolume with NFS](https://github.com/apache/incubator-openwhisk-deploy-kube/blob/master/docs/k8s-nfs-dynamic-storage.md)

### Debugging
The [Whisk debugger wskdb](https://github.com/apache/incubator-openwhisk-debugger) is a debugger based on OpenWhisk functions and rules and wllows debugging OpenWhisk applications. A short overview can be found in these [slides](https://de.slideshare.net/StephenFink1/openwhisk-under-the-hood-london-oct-16-2016?next_slideshow=1).

### Tools
[Composer](https://github.com/ibm-functions/composer)

### Examples
Simple chat client on OpenWhisk: [1](https://www.youtube.com/watch?v=hGl__huStnc) [2](https://github.com/starpit/serverless-chat-demo)
[OpenWhisk and webfrontend:](https://horeaporutiu.github.io/blog/openwhisk-web-actions-and-rest-api-calls/)
[Serverless examples:](https://github.com/cfjedimaster/Serverless-Examples/tree/master/file_upload)
[Uploading a file to OpenWhisk:](https://www.raymondcamden.com/2017/06/09/uploading-files-to-an-openwhisk-action)

### Python libraries included by default
[Python 2 and 3 libraries:](https://github.com/apache/incubator-openwhisk/blob/master/docs/actions-python.md)

### Building native binaries for actions
Binaries have to be compiled for the specific target Apline OpenWhisk containers. They cannot just be
copied in binary form from Ubuntu or another distribution. In fact, this means, that the source files
have to be compiled in the same environment as they will be executed, when the action gets triggered.
Please follow [these instructions](https://github.com/apache/openwhisk/blob/master/docs/actions-docker.md)
to compile sources to a native binary able to execute within actions.

### Random other links
https://stackoverflow.com/questions/43827471/parsing-and-saving-multipart-form-data-from-a-file-using-werkzeug
[Flask multipart parser:](https://gist.github.com/tgwizard/95b82c98e17e72a4c3c0d75dda19eef4)
https://www.flaskapi.org/api-guide/parsers/
https://stackoverflow.com/questions/40414526/how-to-read-multipart-form-data-in-flask
https://www.sandtable.com/reduce-docker-image-sizes-using-alpine/
