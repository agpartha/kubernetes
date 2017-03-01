#Working with Networks and Policy

Namespaces and Labels give us the beginnings of tenancy. Networks are one of the two key areas where this model becomes almost immediately useful, with the other being multi-project name collision segregation (and possibly a modicum of security).

##Creating and Using Multiple Networks

In order to create networks, we'll be using the CNI model for network segregation. This uses selectors as a way of segregating resources and defining our network segments. In order to reduce the amount of typing, for the most part, we have spec manifests for the different resources.

###Note, the manifests are all in the vagrant-kubeadm/net_lab directory (may have to do a 'git pull' in the vagrant-kubeadm directory if you don't see them.)

To start with, we'll create a namespace to separate our resources for everyone elase.  We'll be adding "segments" to this namespace in order to provide our isolation (and these are really just labels we use to select resources).  So first, create a tenant that we can use for the rest of these resources. Note that unlike the namespace we created earlier, this namespace also has a label of the same name, this is used to allow us to filter/select against this namespace.

```
kubectl create -f namespace-tenant-a.yaml
```

The actual file is:
```
kind: Namespace
apiVersion: v1
metadata:
  name: tenant-a
  labels:
    name: tenant-a
```

Next we'll create a service.  This is perhaps backwards from how we've done it before, adding a service to a pod/depoyment/replicaset, but it's part of how Kubenetes works, and again, we'll be using a selector (special label) to "select" resources to be connected. In this case, it's also a special resource, "romanaSegment" which actually triggers a new network to be created.

```
kubectl create -f backend-service.yaml
```
This is the file:
```
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: tenant-a
spec:
  selector: {romanaSegment: backend}
  ports: [{protocol: TCP, port: 80, targetPort: 80}]
```

We can see that the selector is now defined on the service, and that it's ready to bind port 80 for us:

```
kubectl describe svc -n tenant-a
```

Now, let's create a pod a frontend pod and a backend pod.  We're going to allocate each pod to their own segment (and connecting the backend pod to the backend selector, and therefore service in the process)

```
kubectl create -f pod-frontend.yaml
kubectl create -f pod-backend.yaml
```

See if you can see where we've defined the other romanaSegment in the two pod definitions:

pod-frontend.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-frontend
  namespace: tenant-a
  labels:
    app: nginx
    romanaSegment: frontend
spec:
  containers:
  - name: nginx
    image: rstarmer/nginx-curl
    ports:
    - containerPort: 80
```

pod-backend.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-backend
  namespace: tenant-a
  labels:
    app: nginx
    romanaSegment: backend
spec:
  containers:
  - name: nginx
    image: rstarmer/nginx-curl
    ports:
    - containerPort: 80
```

Let's now see if we can communicate between these two segments and pods.  They both have clusterIPs which we can find in the normal way:

```
kubectl describe -n tenant-a pod/nginx-frontend | grep IP
kubectl describe -n tenant-a pod/nginx-backend | grep IP
```
and we can execute a ping between them:

```
kubectl -n tenant-a exec nginx-frontend -- ping -c 3 -W 1 $(kubectl describe -n tenant-a pod/nginx-backend | awk '/IP:/ {print $2}')
```

And just for completeness, let's see if our nodes were scheduled onto different or the same physical nodes (so that our policy can be sure to apply later)

```
kubectl get pods -o wide -n tenant-a
```
Before we apply our baseline policy model, everything should be able to talk (as we've actually just shown via pings between services). But for completeness, let's do an actual TCP session:

```
kubectl -n tenant-a exec nginx-frontend -- curl $(kubectl describe -n tenant-a svc/my-service | awk '/IP:/ {print $2}') --connect-timeout 5
```

This shows that our session is actually connected, and that we are able to talk to the backend from the frontend pod. Isolation is applied at the namespace level, so we'll do this to our tenant-a namespace.

## Policy on the Network
```
kubectl annotate --overwrite namespaces 'tenant-a' 'net.beta.kubernetes.io/networkpolicy={"ingress": {"isolation": "DefaultDeny"}}'
```

We can see the annotation applied with:

```
kubectl get ns tenant-a -o yaml
```

If we try our curl "web request" or pings, they will all now fail, because we've applied a "DefaultDeny" policy.  This is the only policy that's available currently, and it may be the only one that's usually needed...

a curl:
```
kubectl -n tenant-a exec nginx-frontend -- curl $(kubectl describe -n tenant-a svc/my-service | awk '/IP:/ {print $2}') --connect-timeout 5
```
or a ping:
```
kubectl -n tenant-a exec nginx-frontend -- ping -c 3 -W 1 $(kubectl describe -n tenant-a pod/nginx-backend | awk '/IP:/ {print $2}')
```
###Extra Credit
- Does the inverse set of operations also fail?

Let's add the policy to allow our frontend to talk to the backend:

This is the policy we'll apply:

romana-np-frontend-to-backend.yml
```
apiVersion: extensions/v1beta1
kind: NetworkPolicy
metadata:
 name: pol1
 namespace: tenant-a
spec:
 podSelector:
  matchLabels:
   romanaSegment: backend
 ingress:
 - from:
   - podSelector:
      matchLabels:
       romanaSegment: frontend
   ports:
    - protocol: tcp
      port: 80
```

We'll apply the policy, and then we can check and see if the policy now allows frontend to backend communication:

```
kubectl create -f romana-np-frontend-to-backend.yml
```

Again, we can test our curl and ping commands to verify that they work again:

a curl:
```
kubectl -n tenant-a exec nginx-frontend -- curl $(kubectl describe -n tenant-a svc/my-service | awk '/IP:/ {print $2}') --connect-timeout 5
```
or a ping:
```
kubectl -n tenant-a exec nginx-frontend -- ping -c 3 -W 1 $(kubectl describe -n tenant-a pod/nginx-backend | awk '/IP:/ {print $2}')
```

HMMM.  Well, something still doesn't work.  Review the policy, any idea why Curl works, but not ping?


###Extra Credit
- what would we have to add to the policy to allow ping to work?
- does the inverse curl (and if updated ping) work as well?
- create another namespace, and a front and backend pod, and verify that the application of policy only impacts the namespace to which the 'annotation' is applied.
- log in to the underlying hosts and look at the routing tables as you create resources in a new pod, that create new romanaSegments.  What do you see happening (hint: ip route will show you the route table)


## Cleanup
For the most part cleanup is consistent with the rest of our service/pod/etc. management, and we can delete with the inverse of our create commands:

```
kubectl delete -f romana-np-frontend-to-backend.yml
kubectl delete -f pod-backend.yaml
kubectl delete -f pod-frontend.yaml
kubectl delete -f backend-service.yaml
kubectl delete -f namespace-tenant-a.yaml
```

we also need to cleanup the romana datastore, but this is a bit trickier, and in this environment is not necessary.
