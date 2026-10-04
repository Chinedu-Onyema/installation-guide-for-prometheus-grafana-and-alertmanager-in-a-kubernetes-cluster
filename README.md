# Kubernetes Monitoring Stack (Prometheus, Grafana, Alertmanager) Installation Guide

This repository contains step-by-step instructions and configuration files to deploy a complete monitoring solution on a Kubernetes cluster using the kube-prometheus-stack Helm chart. This stack includes Prometheus for metrics collection, Grafana for visualization, and Alertmanager for handling alerts.

### PDF GUIDE: [STEPS TO INSTALL PROMETHEUS, GRAFANA AND ALERTMANAGER.pdf](https://github.com/user-attachments/files/33024144/STEPS.TO.INSTALL.PROMETHEUS.GRAFANA.AND.ALERTMANAGER.pdf)

### WATCH VIDEO WALKTHROUGH HERE: https://youtu.be/19AcXcnNBjQ

## PREREQUISITE
Before beginning, ensure you have the following:

   1) A Kubernetes Cluster: This can be a production cluster (EKS, OpenShift, GKE, AKS, KOPS) or a local development cluster (Minikube, Kind, Docker Desktop).
      kubectl: Installed and configured to communicate with your cluster.
      Helm 3: Installed locally to manage chart installations.
      
      If using Minikube, ensure Docker Desktop is running and start your cluster

      <PRE>minikube start</PRE>


## INSTALLATION STEPS
Follow these steps sequentially to install and access the monitoring stack.

### Step 1: Install Helm and Add Prometheus Repository

Install the latest version of Helm on your local machine or server, then add the official Prometheus community Helm repository.

#### Download and install Helm script (Linux/macOS/WSL)
<PRE>curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3</PRE>

#### Grant execute permissions
<PRE>chmod 700 get_helm.sh</PRE>

#### Run the installer
<PRE>./get_helm.sh</PRE>

#### Verify installation
<PRE>helm version</PRE>

#### Add Prometheus community repo
<PRE>helm repo add prometheus-community https://prometheus-community.github.io/helm-charts</PRE>

#### Update repositories to get latest charts
<PRE>helm repo update</PRE>


### Step 2: Create Namespace and Custom Configuration

Create a dedicated namespace for monitoring resources and prepare a configuration file to set initial Grafana credentials and Alertmanager replicas.

1) Create the namespace:

   <PRE>kubectl create ns monitoring</PRE>

2) Create configuration file:
   Create a file named prometheus-alert-manager.yml with the following content.
   This sets the default Grafana admin password and configures Alertmanager for high availability.
```
   grafana:
  adminPassword: prom-operator
alertmanager:
  alertmanagerSpec:
    alertmanagerConfigSelector:
      matchLabels:
        release: monitoring
    replicas: 2
  alertmanagerConfigMatcherStrategy:
    type: None
  ```

### Step 3: Install the Prometheus Stack Helm Chart

1) Install the kube-prometheus-stack chart using Helm in the monitoring namespace, applying the custom configuration created in Step 2.

   ```
   helm install monitoring prometheus-community/kube-prometheus-stack \
     -n monitoring \
     -f ./prometheus-alert-manager.yml
   ```
Note: This installation may take several minutes to complete as it pulls container images and starts various pods.


2) Verify the installation by checking the running pods in the namespace:

   <PRE>kubectl get pods -n monitoring</PRE>
   You should see pods running for Prometheus operator, Alertmanager, Grafana, and exporters (node-exporter and kube-state-metrics).


### Step 4: Access the User Interfaces
To access the UIs of Prometheus, Grafana, and Alertmanager, we will use kubectl port-forward

1) Access Prometheus UI
   Run the following command to forward Prometheus traffic to your local machine.

   <PRE>kubectl port-forward service/prometheus-operated -n monitoring 9090:9090</PRE>

   Access: Open a web browser and navigate to [http://127.0.0.1:9090](http://127.0.0.1:9090).
   Note for Cloud VMs/WSL: If running remotely, add --address 0.0.0.0 to the end of the command to allow external access, and access via http://<vm-ip>:9090.

2) Access Grafana UI
   Forward Grafana traffic to your local machine.

   <PRE>kubectl port-forward service/monitoring-grafana -n monitoring 8080:80</PRE>

   Access: Open a web browser and navigate to [http://127.0.0.1:8080](http://127.0.0.1:8080).
   Credentials:
   Username: admin
   Password: prom-operator (as defined in prometheus-alert-manager.yml)

3) Access Alertmanager UI
   Forward Alertmanager traffic to your local machine.

   <PRE>kubectl port-forward service/alertmanager-operated -n monitoring 9093:9093</PRE>

   Access: Open a web browser and navigate to [http://127.0.0.1:9093](http://127.0.0.1:9093)


## HOW MONITORING WORKS

This stack utilizes several methods to gather metrics:

1) Node Exporter: Runs as a DaemonSet on every node to collect hardware and OS metrics (CPU, RAM, disk usage).
   Identified as pods named monitoring-prometheus-node-exporter-....

2) Kube State Metrics: Listens to the Kubernetes API server to generate metrics about the state of Kubernetes objects (deployments, pods, resources).
   Identified as pods named monitoring-kube-state-metrics-....

3) Application Metrics: Prometheus uses service discovery to scrape custom application metrics exposed via a /metrics endpoint within your applications.


## CLEANUP (UNINSTALL)
To remove the monitoring stack and associated resources from your cluster, run the following commands

#### Uninstall the Helm release
<PRE>helm uninstall monitoring --namespace monitoring</PRE>

#### Delete the monitoring namespace
<PRE>kubectl delete ns monitoring</PRE>

