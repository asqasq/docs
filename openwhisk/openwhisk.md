# OpenWhisk

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



## Resources

### Installation
[OpenWhisk on Kubernetes](https://github.com/apache/incubator-openwhisk-deploy-kube)
[OpenWhisk on Kubernetes](https://github.com/IBM/OpenWhisk-on-Kubernetes)

### Configuration
[Use Kubernetes to start user functions instead of docker directly:](https://github.com/apache/incubator-openwhisk-deploy-kube/blob/master/docs/configurationChoices.md#invoker-container-factory)
[Values/keys:](https://github.com/apache/incubator-openwhisk-deploy-kube/blob/735e576f356500424d1a1e720064ca3182da644d/helm/openwhisk/values.yaml#L172)
[Alternatives for certain implementations:](https://github.com/apache/incubator-openwhisk/blob/master/common/scala/src/main/resources/reference.conf#L6-#L15)
[wskadmin](https://github.com/redhat-developer-demos/faas-tutorial/issues/16)

### Tools
[Composer](https://github.com/ibm-functions/composer)

### Examples
Simple chat client on OpenWhisk: [1](https://www.youtube.com/watch?v=hGl__huStnc) [2](https://github.com/starpit/serverless-chat-demo)
[OpenWhisk and webfrontend:](https://horeaporutiu.github.io/blog/openwhisk-web-actions-and-rest-api-calls/)
[Serverless examples:](https://github.com/cfjedimaster/Serverless-Examples/tree/master/file_upload)
[Uploading a file to OpenWhisk:](https://www.raymondcamden.com/2017/06/09/uploading-files-to-an-openwhisk-action)

### Python libraries included by default
[Python 2 and 3 libraries:](https://github.com/apache/incubator-openwhisk/blob/master/docs/actions-python.md)

### Random other links
https://stackoverflow.com/questions/43827471/parsing-and-saving-multipart-form-data-from-a-file-using-werkzeug
[Flask multipart parser:](https://gist.github.com/tgwizard/95b82c98e17e72a4c3c0d75dda19eef4)
https://www.flaskapi.org/api-guide/parsers/
https://stackoverflow.com/questions/40414526/how-to-read-multipart-form-data-in-flask
https://www.sandtable.com/reduce-docker-image-sizes-using-alpine/
