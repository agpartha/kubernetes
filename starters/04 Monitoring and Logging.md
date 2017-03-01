# Monitoring with Prometheus

The first step of course is to get Prometheus running. In order to do this, we leverage the work done by the CoreOS team to help enable Prometheus usage.  There is a deployment that describes a Deployment of Prometheus, along with a "configMap" volume that provides the configuration data needed to launch the service.  The only thing missing from this by default is a service definition is to allow "external" access to the Prometheus service via web-browser (an important part of leveraging the Prometheus infrastructure for our purposes). We've enhnaced the default service by adding a service definition to expose the Prometheus service on port 30000.  So once we create the spec, we'll be able to connect to "PROMDASH" the default web interface view into the Prometheus data service.

Since we want data to be collected, we'll also want some containers running, so we might start a deployment of something and then scale it up or down to create some change for the system to register.

## ConfigMaps

Before we get too far into Prometheus, let's look at configMaps.
In this case, we see in the deployment a volume is described:

```
  .....
    containers:
    - name: prometheus
      - mountPath: "/etc/prometheus"
        name: config-volume
  .....
    volumes:
    - emptyDir: {}
      name: data
    - configMap:
        name: prometheus-config
      name: config-volume
```

The config map in this case is a single YAML description, presented as follows:

```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 30s
      scrape_timeout: 30s
      .....
```

Trace the config map data (which becomes a file name), to the volume name, to the deployment configMap to the container mount point.  Effectively, we've described a yaml config file and mapped it through to a specific file an path in the container.

Luckily for us, we don't have to address the configuration beyond the defaults defined here, and certainly there's plenty to configure beyond these basics, but we can start with this config.

###Extra Credit
- Using the nginx Pod (pod.yml), use a configMap and a create a file called index.html, mapped to /usr/share/nginx/html in the container, to create a web page that expresses your level of excitement over Kubernetes.
- Using the nginx Pod, create a resource definition, and see how little resource you can allocate to the container without it failing due to lack of resources.

## Launching Prometheus
The actual deployment is also fairly straight forward, and the only topic we've not discussed in detail is the resources section, which defines what portion of compute and memory resources are being requested for this container/pod deployment.

The full code, with deployment, configMap, and our added service mapping is as follows:

prometheus.yml
```
---
# Originally from https://raw.githubusercontent.com/coreos/blog-examples/master/monitoring-kubernetes-with-prometheus/prometheus.yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: kube-system
  labels:
    name: prometheus-deployment
  name: prometheus
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - image: quay.io/prometheus/prometheus:v1.0.1
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "-config.file=/etc/prometheus/prometheus.yml"
        - "-storage.local.path=/prometheus"
        - "-storage.local.retention=24h"
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: "/prometheus"
          name: data
        - mountPath: "/etc/prometheus"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 500m
            memory: 2500Mi
      volumes:
      - emptyDir: {}
        name: data
      - configMap:
          name: prometheus-config
        name: config-volume
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: kube-system
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 30s
      scrape_timeout: 30s
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
        - targets: ['localhost:9090']
    - job_name: 'kubernetes-cluster'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - api_servers:
        - 'https://kubernetes.default.svc'
        in_cluster: true
        role: apiserver
    - job_name: 'kubernetes-nodes'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - api_servers:
        - 'https://kubernetes.default.svc'
        in_cluster: true
        role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    - job_name: 'kubernetes-service-endpoints'
      scheme: https
      kubernetes_sd_configs:
      - api_servers:
        - 'https://kubernetes.default.svc'
        in_cluster: true
        role: endpoint
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)(?::\d+);(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_service_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name
    - job_name: 'kubernetes-services'
      scheme: https
      metrics_path: /probe
      params:
        module: [http_2xx]
      kubernetes_sd_configs:
      - api_servers:
        - 'https://kubernetes.default.svc'
        in_cluster: true
        role: service
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_service_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_name
    - job_name: 'kubernetes-pods'
      scheme: https
      kubernetes_sd_configs:
      - api_servers:
        - 'https://kubernetes.default.svc'
        in_cluster: true
        role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: (.+):(?:\d+);(\d+)
        replacement: ${1}:${2}
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_pod_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
---
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  labels:
    run: prometheus
  name: prometheus
spec:
  ports:
  - nodePort: 30000
    port: 9090
    protocol: TCP
    targetPort: 9090
  selector:
    app: prometheus
  type: NodePort
```

Create a file called prometheus.yml, paste the code above into it (or find it in the examples directory in the vagrant-kubeadm project), and launch the prometheus service:

```
kubectl create -f prometheus.yml
```

Data is not all captured instantaneously, so we'll want to give Prometheus a few minutes to capture some inital data, but we can validate that Prometheus is up and running:

## System logging information
Open a web browser on your laptop, and connect to http://192.168.56.10:30000
You should see the default Prometheus page waiting for input. First, select a metric (for example: container_memory_usage_bytes), and hit execute.

You'll be given a list that looks something like:
```
container_memory_usage_bytes{beta_kubernetes_io_arch="amd64",beta_kubernetes_io_os="linux",id="/docker",instance="node-1",job="kubernetes-nodes",kubernetes_io_hostname="node-1"}	79261696
container_memory_usage_bytes{beta_kubernetes_io_arch="amd64",beta_kubernetes_io_os="linux",id="/system.slice/lxcfs.service",instance="node-1",job="kubernetes-nodes",kubernetes_io_hostname="node-1"}	3186688
```

We can drill down into this information by adding subsets of the selector key=value pairs.  For example:

```
container_memory_usage_bytes{image="nginx"}
```
or
```
scrape_duration_seconds{job="prometheus"}
```

We'll use this model to explore the features and data being collected, look at the graph of the data itself.

Now, launch the nginx deployment, and ensure first 0 replicas are running (scale --replicas=0) and then scale up to 3 replicas running.
```
kubectl create -f /vagrant/nginx.yml
kubectl scale deployment/nginx --replicas=0
kubectl scale deployment/nginx --replicas=3
```
next, we can create some load against the system:

```
while true; curl -so /dev/null http://localhost:32501 ; sleep 1; done
```

Let's see what we might be able to learn about our pods.

Firstly, we can look at the available metrics, and find something interesting like:

```
container_cpu_usage_seconds_total
```
or
```
container_memory_usage_bytes
```

we can then filter to a subset as we're really interested in our nginx Pods.  A few labels that may help us tune in to our specific containers:

```
pod_name=~"nginx-.*"
image=~"nginx"
namespace="default"
```

We might use them like:
```
container_memory_usage_bytes{pod_name=~"nginx-.*"}
container_memory_usage_bytes{pod_name=~"nginx-.*",image=~"nginx"}
```

Since we've not pinned our containers can be scheduled onto any available core, we may see more than one cpu defined. In order to really get each pod's data rather than have the data split across multiple cores, we can concatenate data, by addition:

```
container_cpu_usage_seconds_total{pod_name=~"nginx-.*",image=~"nginx.*",cpu="cpu00"} +  ignoring(cpu) container_cpu_usage_seconds_total{pod_name=~"nginx-.*",image=~"nginx.*",cpu="cpu01"}
```

###Extra Credit
- capture metrics that look interesting, and share with your colleagues.  One of the biggest challenges with a metrics tool is that there are often so many metrics to look at, it's difficult to figure out where to start.
