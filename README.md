# Complete-_devsecops_gitops_CICD_project_gitops_config
CI/CD DevSecOps Project (Part 2: GitOps and Monitoring)

This repo is the Continuous Deployment and Gitops part of the earlier https://github.com/MalickReborn/Complete-_devsecops_gitops_CICD_project


This documentation will be shape following this structure:

Table of Contents
- Introduction (#introduction)

- GitOps with ArgoCD (#gitops-with-argocd)

- ArgoCD Image Updater (#argocd-image-updater)

- Monitoring with Prometheus and Grafana (#monitoring-with-prometheus-and-grafana)

- Next Steps (#next-steps)
- Notes

## Introduction
This section of the README outlines the implementation of a GitOps approach using ArgoCD to manage deployments of a Flask application on a Kubernetes cluster . 

ArgoCD Image Updater monitors container image updates on Docker Hub and triggers automatic synchronizations. Prometheus and Grafana provide monitoring for the cluster and the application. The CI pipeline (GitHub → Jenkins → tests, scans, build, push to Docker Hub) is already in place and populates the container registry.
ArgoCD synchronizes Kubernetes manifests stored in the K8s_manifests/ directory of the GitHub repository with the Kubernetes cluster, following a declarative GitOps approach.

##Prerequisites
Operational Kubernetes cluster (for us 1 master node, 1 worker node on VMware VMs . You could use ay cluster setup , EKS, AKs, microk8s, minikube, itw).

K8s_manifests/ directory containing kustomization.yaml, flaskforCICD.yaml (deployment), and service.yaml (service) on your SCM platform.

Access to Docker Hub for container images.

## Installing ArgoCD
 Create the argocd namespace and install ArgoCD:
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
 Access the ArgoCD UI via port-forwarding:
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

 Retrieve the initial password for the admin user:
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Configuring the ArgoCD Application
An ArgoCD application file is created to monitor the K8s_manifests/ directory and apply manifests using Kustomize.
File: argocd-application.yaml
          
          apiVersion: argoproj.io/v1alpha1
          kind: Application
          metadata:
            name: flask-app
            namespace: argocd
          spec:
            project: default
            source:
              repoURL: https://github.com/<your-repo>/<your-project>
              targetRevision: main
              path: K8s_manifests
              kustomize:
                namePrefix: flask-
            destination:
              server: https://kubernetes.default.svc
              namespace: default
            syncPolicy:
              automated:
                prune: true
                selfHeal: true

Explanation:
path: K8s_manifests: Points to the directory containing kustomization.yaml.

kustomize: Uses Kustomize to apply manifests.

namePrefix: flask-: Adds a prefix to resources for clarity.

syncPolicy.automated: Enables automatic synchronization with pruning and self-healing.

Apply the file:
```
kubectl apply -f argocd-application.yaml
```


**_Kubernetes Manifests_**
The manifests are organized in K8s_manifests/ and use Kustomize for customization.
  File: K8s_manifests/kustomization.yaml
      
      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      resources:
        - flaskforCICD.yaml
        - service.yaml
      namespace: default

  File: K8s_manifests/flaskforCICD.yaml
      
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: flask-app
        namespace: default
      spec:
        replicas: 2
        selector:
          matchLabels:
            app: flask-app
        template:
          metadata:
            labels:
              app: flask-app
          spec:
            containers:
            - name: flask-app
              image: docker.io/<your-user>/flask-app:latest
              ports:
              - containerPort: 5000
              resources:
                limits:
                  cpu: "500m"
                  memory: "512Mi"
                requests:
                  cpu: "200m"
                  memory: "256Mi"

  File: K8s_manifests/service.yaml
      
      apiVersion: v1
      kind: Service
      metadata:
        name: flask-app
        namespace: default
      spec:
        selector:
          app: flask-app
        ports:
        - port: 5000
          targetPort: 5000
        type: ClusterIP

Notes:
The deployment uses the flask-app:latest image (You can replace with your Docker Hub image).

The service exposes port 5000 (standard for Flask) on port 5000 internally.

Kustomize groups the manifests for consistent application.


**_ArgoCD Image Updater_**
  ArgoCD Image Updater monitors new versions of the flask-app image on Docker Hub, updates the manifests in the GitHub repository, and triggers synchronization via ArgoCD.

  Installation
Install ArgoCD Image Updater in the argocd namespace:
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```
  Configuration
Add annotations to the deployment manifest to specify the image to monitor:
Modified File: K8s_manifests/flaskforCICD.yaml
yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  namespace: default
  annotations:
    _argocd-image-updater.argoproj.io/image-list: flask-app=docker.io/<your-user>/flask-app_
    _argocd-image-updater.argoproj.io/update-strategy: latest_
    _argocd-image-updater.argoproj.io/write-back-method: git_
    _argocd-image-updater.argoproj.io/git-branch: main_
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        image: docker.io/<your-user>/flask-app:latest
        ports:
        - containerPort: 5000
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "200m"
            memory: "256Mi"

  Annotations Explained:
  image-list: Maps the flask-app alias to the Docker Hub image.
  
  update-strategy: latest: Uses the latest image version.
  
  write-back-method: git: Updates the manifest in the GitHub repository.
  
  git-branch: main: Specifies the target branch for commits.

  Registry Configuration
Create a secret to allow Image Updater to access Docker Hub:
bash

kubectl create secret docker-registry dockerhub-secret -n argocd \
  --docker-username=<your-user> \
  --docker-password=<your-password> \
  --docker-server=https://index.docker.io/v1/

Update the ArgoCD Image Updater ConfigMap to use the secret:
File: argocd-image-updater-config.yaml

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: argocd-image-updater-config
      namespace: argocd
    data:
      registries: |
        - name: dockerhub
          api_url: https://registry-1.docker.io
          prefix: docker.io
          credentials: secret:argocd/dockerhub-secret

Apply:
```
kubectl apply -f argocd-image-updater-config.yaml
```

#### Workflow
The CI pipeline pushes a new image (e.g., <your-user>/flask-app:<new-tag>) to Docker Hub.

ArgoCD Image Updater detects the new image.

It updates K8s_manifests/flaskforCICD.yaml in the GitHub repository (e.g., changes image: ...:latest to image: ...:<new-tag>) and commits the change.

ArgoCD detects the repository change and synchronizes the cluster with the new version.


### Monitoring with Prometheus and Grafana
Prometheus collects metrics from the Kubernetes cluster and the Flask application, while Grafana provides visualizations through dashboards.

  Installation
Install Prometheus and Grafana using the kube-prometheus-stack Helm chart:
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
```
 Configuration
   _Prometheus_:
Automatically collects metrics from nodes, pods, and Kubernetes services.

- For the Flask application, expose a /metrics endpoint (e.g., using prometheus-flask-exporter in your Python code):
```
#python
from flask import Flask
from prometheus_flask_exporter import PrometheusMetrics

app = Flask(__name__)
metrics = PrometheusMetrics(app)

@app.route('/')
def hello():
    return 'Hello, World!'
```
Expose this endpoint in the manifest (e.g., via a dedicated port or annotation).

   _Grafana_:
Access Grafana via port-forwarding:
```
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
```
Default credentials: admin / prom-operator (change after first login).

- Import dashboards:
  Cluster: ID 6417 (node metrics).
  
  Pods: ID 15760 (pod metrics).

- Flask: Create a custom dashboard for /metrics (e.g., HTTP requests, latency).

  _Alerts_:
Configure alerts for critical thresholds (e.g., pod failures or high latency).

Example rule:
```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: flask-app-alerts
  namespace: monitoring
spec:
  groups:
  - name: flask-app
    rules:
    - alert: HighRequestLatency
      expr: rate(http_request_duration_seconds_sum{job="flask-app"}[5m]) / rate(http_request_duration_seconds_count{job="flask-app"}[5m]) > 0.5
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High request latency for flask-app"
    - alert: PodNotRunning
      expr: kube_pod_status_phase{namespace="default",pod=~"flask-app.*",phase!="Running"} > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Flask-app pod not running"
```
Apply: `kubectl apply -f prometheus-rules.yaml.`

## Next Steps

- Configure notifications for Prometheus alerts (e.g., via Slack or Alertmanager).

- Implement custom metrics for the Flask application (e.g., error counters or business metrics).

- Set up advanced deployment strategies (canary, blue-green) with ArgoCD.

- Secure manifests with Sealed Secrets or a secret manager like Vault.

##Notes

- Replace <your-user> and <your-repo> with your actual Docker Hub username and GitHub repository.
- Verify that the dockerhub-secret is correctly configured for ArgoCD Image Updater.
- Validate manifests with kubeval or kustomize build K8s_manifests/ | kubectl apply --dry-run=client before pushing.
- Back up Grafana dashboards as JSON files for re-importing if needed.


