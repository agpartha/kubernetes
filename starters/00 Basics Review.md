# Review the Basics of Kubernetes with Docker.
In this lab you'll refresh you memory about some of the basic Kubernetes operations you will want to be comfortable with. Once you have logged into your kubernetes environment, become root:
```
sudo su -
```
Kubernetes access:

use kubectl to list nodes, pods, etc.
```
kubectl get {nodes,pods,deployments,replicasets}
```

Deploy Pod

```
kubectl create -f pod.yml
```
Deploy Deployment - scale up/scale down
```
kubectl delete -f pod.yml
kubectl create -f deploy.yml
kubectl scale deploy/nginx --replicas 4
kubectl get deploy/nginx
kubectl describe pods | grep IP
curl
```
Deploy Service

```
kubectl delete -f deploy.yml
kubectl create -f nginx.yml
kubectl get svc/nginx -o yaml | grep "(IP\|Port)"
curl IP:Port
```

## Labels and Namespaces

###Namespaces

```
kubectl create namespace testspace
```

Change Kubectl namespace default to testspace
```
mkdir ~/.kube
cp /etc/kubernetes/admin.conf ~/.kube/config
kubectl config use-context admin@kubernetes
kubectl config set-context admin@kubernetes --namespace testspace
```

Deploy a pod
```
kubectl create -f pod.yml
```
check with explicit namespace:
```
kubectl get pods --namespace default
kubectl get pods --namespace testspace
kubectl get pods
```
###Extra Credit
- copy the admin.conf file to your local laptop, download the kubectl binary, and verify that you can "get pods" with the --kubeconfig admin.conf parameter from yoru laptop rather than from the 'master' node.
- create a ~/.kube/config file based on the admin.conf file above, and verify that you don't have to pass --kubeconfig to the kubectl command.

###Labels
Add label to pod(s)

First, we can edit the pod spec in an editor on the master (or our local laptop if we did the above Extra Credit)
```
kubectl edit pod/nginx
```

Add a labels section to the root metadata: section (there should be a metadata section a the top of the pod specification).  Leave the rest of the metadata as it is (or try changing it and see what happens...)
```
metadata:
  labels:
    env: test
    tier: web
```

Launch pods with different Labels (duplicate a default pod spec so you at least have multiple names, if not labels right in the pod spec)
  env: test
  tier: web
and
  env: prod
  tier: web


Then we can search either with specific labels:

kubectl get pods -l env=test

or set based

kubectl get pods -l 'env in (test,prod)'
kubectl get pods -l 'tier in (web),env notin (test)'

### Extra Credit
- Create a third label, select based on sets of the three labels

## Docker registry
Create an account at hub.docker.com

on the master node:
```
docker login
```

Create an example container based on the following Dockerfile

Dockerfile
```
FROM alpine:latest
MAINTAINER {your_email_in_docker_hub}
LABEL description='A simple python http.server based Hello World web server'
RUN apk update && apk add python3
RUN echo "Hello World!" > /root/index.html
WORKDIR /root
EXPOSE 8000
CMD python3 -m http.server 8000
```

Build an image based on the dockerfile (Dockerfile needs to be in the same directory as where you run the next command). Replace {username} with the username you registered with Docker Hub.
```
docker build . -t {username}/python-server:latest -t {username}/python-server:1.0
```

Now, we can push our newly created docker image to the hub.  We push both a "latest" tag and a version "1.0" tag, so that we can call for our image without a version or latest (latest being the default). In addition, when we update our image, we can re-set the latest tag, but we'll still have our 1.0 copy in the hub in case we need to revert to an older version.
```
docker push {username}/python-server:latest
docker push {username}/python-server:1.0
```

Now we need to clean up our local images so that we can verify our ability to download and launch from the remote Hub registry.
```
docker rmi {username}/python-server:latest
docker rmi {username}/python-server:1.0
```

And lastly, let's launch a deployment with our new image, and verify that it is indeed running.
```
kubectl run python-test --image=python-server:latest
kubectl describe deploy/python-test
```

###Extra Credit
 - Check to see if the image is now in the local docker image library
 - Check to see if the output is as you would expect (curl {ClusterIP})
