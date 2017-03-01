# Persistence of Storage in Kubernetes

There are at least two types of storage in Kubernetes:

 * Volumes: describe mountable disk based resources in a pod. The pod spec can include 'emptyDir','hostPath', and 'secret' volumes.

 The most commonly used volume type that doesn't ensure persistence is emptyDir. This type is used to share information between containers in a pod, and is often populated on startup by one of the containers.
 host Path
 * Persistent Volumes: a resource that has life outside of the Pod, and the resource is made available to the location where the Pod gets scheduled.

## Non-persistent or Ephemeral
Let's look at a Pod with two containers that want to share data via disk. One container will create a set of documents to be displayed in some fashion (git pull/clone, programatic creation, unbundling of a tar file, etc.), and the second container will run an Nginx instance and display the data in the document.

These containers need to talk to each other, but once the pod is gone, there is no need for any data persistence. While it is possible to use a hostPath, and even see this data externally (this is actually a persistent volume), we don't need that, and instead can use an 'emptyDir' volume for this process.

An example of a pod description is as follows:

```
apiVersion: v1
kind: Pod
metadata:
  name: timestamp
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: empty-volume
  - image: rstarmer/timestamp
    imagePullPolicy: Always
    name: timer
    volumeMounts:
    - mountPath: /clone
      name: empty-volume
  volumes:
  - name: empty-volume
    emptyDir:
```

Note that in the spec, there are two sections, a containers section, and a volumes section. This volumes section tells Kuberntes to create an empty directory "in the Pod" and associate it to both containers.  In each case, we mount the volume to an appropriate location for that particular container.  In this case, we're mounting the Nginx html directory into the same location as the git-clone instance.  This lets the clone instance load a copy of the code, and the web server simply focuses on delivering the code.

The "clone" container in this example is made up of the following

Dockerfile:
```
FROM alpine:latest
MAINTAINER rstarmer@gmail.com
LABEL description='A git clone container that clones a repo passed in the HTTPS_REPO variable'
ENV HTTPS_REPO=${HTTPS_REPO:-https://github.com/kumulustech/startbootstrap-new-age}
RUN apk update && apk add git && mkdir /clone
COPY clone.sh /root/clone.sh
CMD /root/clone.sh
```

clone.sh
```
#!/bin/sh
git clone ${HTTPS_REPO} /clone
if [ $? ] ; then
cd /clone
while [ true ]
do
git pull
sleep 60
done
else
echo '<H1>Clone Operation Failed</H1>' > /clone/index.html
fi
```

Use the code above (or an equivalent container set) to deploy a two container environment.

1) Using the above information, launch the pod (you don't have to create your own image, just use the upstream image)
  - Can you see the volume that gets created when the Pod is started?
  - Are you able to validate the changing result with curl against the pod IP?
  - Log in to the two containers in the pod, and have a look at their mapped directories, is the content changing as expected?
  hint: ```kubectl exec -it {pod} -c {container} sh```

2) change from an emptyDir to a hostDir.
  Change the volume section in the Pod specification:
```
  volumes:
  - name: empty-volume
    hostPath:
      path: /tmp/timestamp
```

  - What do you have to do in order to ensure that the hostDirectory actually exists on the nodes where the pod may be scheduled?
  - Is the "local" version of the index file changing in correlation to the output of the curl command from the deployed instance?
  - How does this compare to what the containers internally see?

### Extra Credit
use the nodeSelector model to constrain the pod to node-1 only:
https://kubernetes.io/docs/user-guide/node-selection/

## Perstistence

As we've seen with the hostPath "persistent" model, we really need a way of describing volumes that are cross-node persistent, as the scheduler isn't going to always select the node we might want our Pods to land on. If you did the extra credit you saw a way where we can tweak the scheduler, but this doesn't resolve the issue of a node failure and the non-persistence of data in that failure case.

So we'd really prefer to launch nodes with a persistent storage model that is also based on some form of distributed storage so that wherever we end up scheduling our instance, we have our storage associated with that node as well.  The model for this in Kubernetes is broken into two segments: The Persistent Volume, and the Persistent Volume Claim.  First we create persistent volumes, and for some mechanisms there are auto-provisioning scripts avaialable so that we can just create a Persistent Volume model, and the provisioner will create volumes at the point where they are claimed. This works well with certain services including Google, Amazon, and Azure's Kubernetes service environments, and with a growing set of opensource and third party systems as well, including GlusterFS, Ceph, and the OpenStack Cinder service. In our environment we're going to use a simpler distributed storage service, NFS, and while there is work on the auto-provisioning service, we'll be using the static PV creation process to create a pool of volumes.

The vagrant service should have already deployed a multi-node NFS service that uses the master node as it's NFS server (exporting the /var/nfs/general directory).  There should also be four sub-directories that will be used as our persistent volumes for remote mounting (named one, two, three, four specifically).

In order to create our persistent volumes, we'll stick with the "simple" model and create them from spec files.

1) Create four spec files changing the names and nfs paths for each.

pv1.yml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
  annotations:
    volume.beta.kubernetes.io/storage-class: "slow"
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /var/nfs/general/one
    server: 192.168.56.10
```

2) Now create the persistent volumes with a command like:

kubectl create -f pv1.yml

Note: the storage class annotation is currently important, as it needs to match the Persistent Volume Claim that we'll create next.

3) Once these volumes have been created, explore them with the describe and get commands.

```
kubectl get pv
kubectl describe pv
```

Does the information match what one would expect based on the spec file?

Now we can create a persistent volume claim against the persistent volumes.

4) Again, we'll start with a spec file to avoid mis-typing:

pvc1.yml
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: claim1
  annotations:
    volume.beta.kubernetes.io/storage-class: "slow"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

5) Create claims against at least two of the PVs we created earlier.
  - Thoughts about how you'd do that?

6) Now we have a volume, and a volume claim.  Let's change our previous example to use a persistent location to store the shared information between our the containers in our Pod.

As before, update the volume section to call our our PersistentVolumeClaim:
```
volumes:
  - name: empty-volume
    persistentVolumeClaim:
      claimName: claim1
```

Note: We're ensuring that all of the names map to their resources (hence the empty-volume name in this volume definition).

7) Determine which node your pod was finally deployed to, log in, become root, and look through the volumes that are existent on the node. Can you find the NFS volume that was mounted for our instance?

8) remove and re-create the pod, or create a second pod (you'll have to manually select the second volume claim name, since there are no automated provisioners for this part of the process).
- Do you see the nfs mounts change as Pods are created and deleted?

###Extra Credit
- Find the master node, is the information in the nfs directory reflecting the running or non-running state of the pods we've deployed?
- How does the content faire when a pod is deleted/created.
