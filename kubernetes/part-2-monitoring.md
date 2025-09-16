# Part 2: Monitoring the Cluster
In part 2 of this series, we begin to add monitoring services to our cluster. This guide is based on the following versions:

- k3s version 1.33+
- kube-prometheus-stack version 0.85.0 (Helm Chart version 77.6.2)

Prometheus is a monitoring and alerting tookit for collecting and storing time-series metrics from the k8s cluster. It'll help with providing visibility into resource usage, application performance, and overall cluster health.

Why should we add it to our cluster?

- Simple answer: to gain insights.
- Longer answer: gain insights into cluster behavior and proactively identify and resolve issues.

**What makes it best suited for Kubernetes monitoring?**

Prometheus is a pull-based model which integrates perfectly with k8s service discovery to automatically find and monitor the dynamically changing services in the cluster.

## Step 1: Required Prerequisite Knowledge

Prior to getting started with our monitoring stack with Prometheus and Grafana, there are a few key points to understand. Unlike traditional Kubernetes cluster deployments, k3s by Rancher runs as a single binary that includes everything (kubelet, kube-apiserver, kube-controller-manager, kube-scheduler, kube-proxy). This is important because there are not going to be separate, standalone services in our k8s cluster for these. It is all wrapped up in the metrics provided by the single binary for k3s.

By default, k3s uses the following:

- **SQLite** as the datastore instead of Etcd.
- **containerd** as the container runtime environment.
- **Traefik** as the Ingress Controller
- **klipper-lb** as the Service Load Balancer (based on Klipper)
- **Flannel** as the default Container Network Interface (CNI)

Nearly all of these can of course be changed, but that's outside the scope of this guide. However, each of these defaults fits with the theme of k3s being "lightweight Kubernetes" solution.

Additionally, while the k3s binary runs all the control plane components, each still behaves as if it's a standalone service. This includes metrics behavior! For each of these control plane components, the metrics are only exposed on localhost (`127.0.0.1`) which does not help us with exposing them to Prometheus or Grafana.

With those important points out of the way, let's move on to how we can use this knowledge to get our monitoring and alerting setup with Prometheus and Grafana.

## Step 2: Cluster Configuration Updates

As pointed out in the previous section, there are important considerations for k3s kubernetes cluster and how best to monitor them with Prometheus and Grafana.

### Bind-Address Exposure

To get started, we need to specifically configure the bind-addresses for the `kube-controller-manager`, `kube-scheduler`, and `kube-proxy` controller plan components. Create and add the following to the file `/etc/rancher/k3s/config.yaml`:

```yaml
# Enable control plan metrics outside of localhost
kube-controller-manager-arg:
  - bind-address=0.0.0.0
kube-scheduler-arg:
  - bind-address=0.0.0.0
kube-proxy-arg:
  - metrics-bind-address=0.0.0.0
```

Then restart the k3s service with `sudo systemctl restart k3s.service` so it sees and reads this new `config.yaml` file.

You can confirm these changes are in place with:

```bash
ss -tuln | grep -E ':10257|:10259|:10249`

# Should show the following:
tcp   LISTEN 0      4096               *:10259            *:*
tcp   LISTEN 0      4096               *:10257            *:*
tcp   LISTEN 0      4096               *:10249            *:*
```

Without this `config.yaml` change, instead of a * (star) you would see `127.0.0.1` since that is the default bind-address for these control plane components.

Another check is for the control plane metrics endpoints:

```bash
curl -k https://<NODE_IP>:10257/metrics     # KubeControllerManager
curl -k https://<NODE_IP>:10259/metrics     # KubeScheduler
curl -k https://<NODE_IP>:10249/metrics     # KubeProxy

# Check ServiceMonitor discovery
kubectl get servicemonitor [-n monitoring]
```

### Essential Adaptations to k3s

There is going to be a lot of unpacking here. Let me first show you the values file you'll need to created along with the content inside it. Then after I'll explain what each part does.

Create a `k3s-prom-values.yaml` file and add all of the following to it:

```yaml
# Disable incompatible default components
grafana:
  persistence:
    enabled: true
    size: 10Gi
    # Use your storage class (e.g., longhorn, nfs-client)
    storageClassName: "local-path"  # Adjust to your setup

# Optional to disable default rules
#defaultRules:
#  create: false

# K3s-specific service monitor configuration
# Only scrape kubelet to avoid duplicate metrics
kubelet:
  enabled: true

# Disable separate scraping since k3s combines all metrics
kubeApiServer:
  enabled: false

kubeControllerManager:
  enabled: true
  endpoints: ["YOUR_MASTER_NODE_IP"]  # Replace with your master IP
  service:
    enabled: true
    port: 10257
    targetPort: 10257
  serviceMonitor:
    enabled: true
    https: true # Enable HTTPS for k3s v1.22+
    insecureSkipVerify: true # Skip certificate verification

kubeScheduler:
  enabled: true  
  endpoints: ["YOUR_MASTER_NODE_IP"]  # Replace with your master IP
  service:
    enabled: true
    port: 10259
    targetPort: 10259
  serviceMonitor:
    enabled: true
    https: true # Enable HTTPS for k3s v1.22+
    insecureSkipVerify: true # Skip certificate verification

kubeProxy:
  enabled: true
  endpoints: ["YOUR_MASTER_NODE_IP"]  # Replace with your master IP  
  service:
    enabled: true
    port: 10249
    targetPort: 10249
  serviceMonitor:
    enabled: true
    https: false

# Disable etcd if using SQLite (default k3s)
kubeEtcd:
  enabled: false

# Storage configuration for persistence
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: "local-path"  # Adjust to your setup
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    # K3s memory requirements
    resources:
      requests:
        memory: 2500Mi
        cpu: 500m
      limits:
        memory: 4Gi
        cpu: 2000m

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: "local-path"  # Adjust to your setup
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
```
This will be broken down by the main section headings, so `grafana`, `defaultRles`, `kubelet`, etc...

- First section is `grafana` configures persistent storage for Grafana dashboards, datasources, and user settings.
- Optionally, we can disable the default rules to avoid frustrations with what actually works for k3s and what doesn't.
  - The default rules assume separate control plane pods, which don't exist by default in k3s's unified architecture.
  - Optional because we created endpoints for 3 of these control plane resources.
- We want to scrape `kubelet` and disable the `kubeApiServer` scraping to avoid duplicate metrics.
- The next 3 sections configure the controller planes for *Controller Manager*, *Scheduler*, and *Proxy*, since we previously configured the bind-address to point outside of localhost.
  - The configuration sets the endpoint IP and ports and enables the service.
  - k3s v1.22+ uses HTTPS by default, so we skip certificate verification (just for Prometheus).
    - This is for simplicity in a lab environment. For production consider proper certificate management.
- We also disable the `etcd` scraping since k3s uses SQLite by default.
- The final sections for `prometheus` and `alertmanager` set retention, resource, and storage options.

Feel free to paste the above yaml file contents into your favorite LLM to ask more specific questions or gain a deeper explanation for each line. I'd suggest learning about the `requests` and `limits` parts as those are great to understand for resource utilization.

## Step 3: Deploy Prometheus

Finally, we get to deploy our monitoring stack to our Kubernetes cluster!

To recap the what and why for Prometheus:

- It's pull-based model integrates well with k8s service discovery.
- It's a monitoring and alerting tookit.
- It collects and stores time-series metrics from the k8s cluster. 
- It provides visibility from the k8s cluster
- Gives insight into k8s cluster behavior
- Empowers proactive issue identification and resolution.

Optional: Setup a namespace in the k8s cluster for *monitoring* which helps later with administration and maintenance, as well as good practice to group deployments in namespaces:

```bash
kubectl create namespace monitoring
```

First we need to add the Helm Chart:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Then install Prometheus using Helm and specifying our values YAML file:

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values k3s-prom-values.yaml
```

Check installation with the following commands:

```bash
# Check pods [optional namespace if setup]
kubectl get pods [-n monitoring]

# Check services [optional namespace if setup]
kubectl get services [-n monitoring]
```

If you see multiple `prometheus-*` Pods and Services then congratulations are in order! You've got your monitoring stack installed!

But the fun doesn't just end there, much more to do! Before we get into any of that, let's run some temporary checks to verify access and that the Pods and Services are working correctly.

**Verify Access**

Simplest way is with the `port-forward` option:

```bash
# First check the Prometheus dashboard:
kubectl port-forward [-n monitoring] services/prometheus-kube-prometheus-prometheus 9091:9090 --address 0.0.0.0
```

This enables access to the prometheus dashboard at your VM IP on port 9091. We need to use port 9091 instead of 9090 because with Fedora, you'll get Cockpit by default (unless you've disabled it), and that by default uses port 9090.

While that command is waiting, point a browser to `http://<VM_IP>:9091` and you should see the Prometheus Dashboard. If not, then you'll be diving deep into some networking troubleshooting. (A great place to start with would be `firewalld`).

Next we want to check for Grafana Dashboard access. so `ctrl+C` the current running `kubectl port-forward...` command and run the following:

```bash
kubectl port-forward [-n monitoring] services/prometheus-grafana 9091:80 --address 0.0.0.0
```

Like the first `port-foward` verification we did, this one enables access to the Grafana Dashboard at your VM IP on port 9091. While this command is waiting, open your browser again to `http://<VM_IP>:9091` and you should have the Grafana Dashboard loaded.

## Next Steps:

Well done getting your monitoring stack up and running! In part 3 of this series I'll be detailing troubleshooting steps to common issues up to this point.

If you ran into any issues following along, make sure you check out part 3 for a deeper dive into common issues and how to resolve them.