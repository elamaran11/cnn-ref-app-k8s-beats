# GKE-On-Prem

### Grab the configuration
`git clone https://git.keybank.com/scm/gcp00/gke-kubernetes-cluster-beats-monitoring.git`

### Set the cluster-admin-binding
Logging and metrics tools like Filebeat, Fluentd, Metricbeat, Prometheus, etc. run as DameonSets.  To deploy DaemonSets you need the cluster role binding `cluster-admin-binding`.  Create it now:

```
kubectl create clusterrolebinding cluster-admin-binding  \
  --clusterrole=cluster-admin --user=$(gcloud config get-value account)
```
### Set the cluster-admin-binding
In a terminal configured to access your GKE On-Prem environment, check to see if kube-state-metrics is already running:

```
kubectl get pods --namespace=kube-system | grep kube-state
```
If kube-state-metrics is already running, upgrade to a current version of GKE On-Prem that runs kube-state-metrics in a separate namespace before continuing.

Install kube-state-metrics:
```
git clone \
    https://github.com/kubernetes/kube-state-metrics.git
kubectl create -f kube-state-metrics/kubernetes
kubectl get pods --namespace=kube-system | grep kube-state
```

### Deploy example application
This uses the Guestbook app from the Kubernetes docs.  The YAML has been concatenated into a single manifest, and Apache HTTP mod Status has been enabled for metrics gathering.

Before you deploy the manifest have a look at the frontend service.  You may need to edit this service so that the service is exposed to your internal network.  The network topology of the lab where this example was developed has a load balancer in front of the GKE On-Prem environment, and so the service specifies an IP Address associated with the load balancer.  Your configuration will likely be different.

Note : Remove loadBalancerIP for GKE on GCP and keep it if its GKE On Prem

```
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  type: LoadBalancer
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: guestbook
    tier: frontend
  loadBalancerIP: 10.0.10.42
---
```

Edit the guestbook.yaml manifest as appropriate and then deploy it.

`kubectl create -f guestbook.yaml`

### Verify
Check to see that the application is deployed and reachable on your network:

`kubectl get pods -n default`

`kubectl get services -n default`

Open a browser to the IP Address associated with the `frontend` service at port 80.
### Create secrets
Rather than putting the Kafka endpoints into the manifest files they are provided to the Filebeat pods as k8s secrets.  Edit the files `kafka-hosts-ports` and then create the secret:

```
kubectl create secret generic kafka-host \
  --from-file=./kafka-hosts-ports --namespace=kube-system
```

### Deploy index patterns, visualizations, dashboards, and machine learning jobs
Filebeat and Metricbeat provide the configuration for things like web servers, caches, proxies, operating systems, container environments, databases, etc.  These are referred to as *Beats modules*.  By deploying these configurations you will be populating Kafka on respective kafka topics.  

```
kubectl create -f filebeat-setup.yaml
kubectl create -f metricbeat-setup.yaml
```

### Verify
`kubectl get pods -n kube-system | grep beat`

Verify that the setup pods complete
Check the logs for the setup pods to ensure that they connected to Elasticsearch and Kibana (the setup pod connects to both)

### Deploy the Beat DaemonSets
```
kubectl create -f filebeat-kubernetes.yaml
kubectl create -f metricbeat-kubernetes.yaml
```
#### Note: Depending on your k8s Node configuration, you may not need to deploy Jounalbeat.  If your Nodes use journald for logging, then deploy Journalbeat, otherwise Filebeat will get the logs
`kubectl create -f journalbeat-kubernetes.yaml`

### Verify
`kubectl get pods -n kube-system | grep beat`

Verify that there is one filebeat, metricbeat, and journalbeat pod per k8s Node running.

Check the logs for and one of the DaemonSet pods to ensure that they connected to kafka.
