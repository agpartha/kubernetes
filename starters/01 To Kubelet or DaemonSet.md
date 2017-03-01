# DaemonSets - Managing persistence

DaemonSets are tools that support launching Pods/Deployments/Services where there is a requirement for a Pod/'physical' resource.  And the desire is to have that state maintained regardless of how many physical resources, and wether they come or go over time.

## DaemonSets
The key value to the daemonset component is that it takes a pod definition and ensures that the pod is running on all systems.  When we get to discussing persistent storage, we'll talk about the hostPath storage option, which mounts a "local" (e.g. physical node local) storage directory to our pod, from which we can then associate the storage to our individual containers in much the same way as with a host path volume in Docker. This is normally a bad idea, as there's no guarantee that the host on which the path is created is in fact the same as the host where the pod is scheduled.  With the daemonSet model, we're ensured to have a pod on every host, and with that we can now also ensure that our hostPath volumes will always match up with a Pod.  This means that we can associate node specific configuration with our services in a fairly simple fashion.

Let's build our own DaemonSet instantiatoin with our simple web server models.  First, let's create a consistent path on each of our three machines.  Log in to each (master, node-1, node-2) via vagrant ssh {node} and then create a directory (our container will automatically create the index file for our web based service):

```
sudo mkdir /home/daemon ; sudo chmod 755 /home/daemon
```

Now, on the master, let's create a our daemonSet spec file:

time-daemon.yml
```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: timecheck
  namespace: default
spec:
  template:
    metadata:
      labels:
        app: timecheck
    spec:
      hostNetwork: true
      containers:
      - name: nginx
        image: nginx
        imagePullPolicy: Always
        ports:
        - containerPort: 80
          hostPort: 80
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: host-path-volume
      - name: timestamp
        image: rstarmer/timestamp
        imagePullPolicy: Always
        volumeMounts:
        - mountPath: /clone
          name: host-path-volume
      volumes:
        - name: host-path-volume
          hostPath:
            path: /home/daemon/
```

and then we can launch this as we would any other container:

```
kubectl create -f time-daemon.yml
```

## Alternatives to DaemonSets
Kubelet is the control agent on each node that is going to run a Pod for Kubernetes, and is tightly coupled with the rest of the deployment services (including things like DaemonSets and ReplicaSets).  By default it has a built in mechanism for "bootstrap" operations in the Kubernetes environment, allowing a system manager to run some of the Kubernetes components under the watchful eye of the Kubelet engine.  This means that a manifest specification that is placed under Kubelet control directly will gain the benefits of auto-restart/expected update that would normally require a replicaSet instantiation.

This is really the pre-cursor to the daemonset model, but it is still used as a part of the bootstrapping process.

Let's re-work our daemonSet example as a kubelet controlled resource:

Firstly, let's clean up from our last operations:

```
kubectl delete -f time-daemon.yml
```

Now let's create our kubelet based resource. In this case, we're just going to define a pod spec, and then we need to copy that to each of our hosts in turn.  In the vagrant environment, this is a bit easier if we create this in the /vagrant directory on one of our machines (or in our vagrant launch directory on the underlying host), as we can then simply copy that code into place on the various systems.

time-kubelet.yml
```
apiVersion: v1
kind: Pod
apiVersion: v1
kind: Pod
metadata:
  name: timecheck
  namespace: default
  labels:
    app: timecheck
spec:
  hostNetwork: true
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: Always
    ports:
    - containerPort: 80
      hostPort: 80
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: host-path-volume
  - name: timestamp
    image: rstarmer/timestamp
    imagePullPolicy: Always
    volumeMounts:
    - mountPath: /clone
      name: host-path-volume
  volumes:
    - name: host-path-volume
      hostPath:
        path: /home/daemon/
```

In this case, we're going from a "service" model with the daemonSet (closer to a replicaSet or deployment) back to just a pod spec.

Let's load this file into our environment(s) from our laptop environment, we need to log into each machine and copy the manifest into each machine's /etc/kubernetes/manifests/ directory. Kubelet will take it from there.  While we're on the host, let's make sure we have a /home/daemon directory, and that no-one owns it so that the container environment can set permissions appropriately. On systems with a shell environment we can script this as follows (again, from our laptop):

```
for n in master node-1 node-2
do
vagrant ssh ${n} -c 'sudo mkdir -p /home/daemon; sudo chown -R nobody.nogroup /home/daemon; sudo cp /vagrant/time-kubelet.yml /etc/kubernetes/manifests/'
done
```

Once that's done, again on the latop, we can verify that all of the pods are running (may take a minute or two to launch) by curl'ing the external address (in our case, we are cheating and overloading port 80 on all of the node VMs):

```
for n in 192.168.56.10 192.168.56.20 192.168.56.21
do
curl ${n}
done
```
We should get three timestamps that are all identical.

Finally, let's clean up this environment.  In the same way that deleting a deployment, or daemonSet, etc. cleans up the resources, if we remove our pod spec from the /etc/kubernets/manifests directory, we'll find that the node Kubelet engines remove the locally managed containers as well.  We'll also want to remove the index file that our container creates in it's host directory (again, just good system stewardship):

```
for n in master node-1 node-2
do
vagrant ssh ${n} -c 'sudo rm /home/daemon/index.html ; sudo rm /etc/kubernetes/manifests/time-kubelet.yml'
done
```

###Extra Credit:
- Investigate the other pods that are created in the /etc/manifests directory. What is their function and what are they trying to accomplish.  Also, are there resources that are managed like this on all of the nodes?
- Our example is really not doing quite the right thing. Change it from using a hostPort to using a nodePort and add a service definition to expose the process on a higher order port (32000 or greater as a service will do by default).
