# Kubernetes Package Manager (Helm) – Complete Hands-on Practice

Helm is the most widely used package manager for Kubernetes. It helps you install, configure, upgrade, roll back, and uninstall Kubernetes applications using reusable packages called Helm Charts.

This lab works with Amazon EKS, kubeadm on EC2 (Amazon Linux), Minikube, and AKS.

## Quick start from this repository

```bash
git clone https://github.com/atulkamble/helm.git
cd helm
kubectl get nodes
helm lint ./mywebapp
helm template mynginx ./mywebapp
helm install mynginx ./mywebapp -n webapp --create-namespace --wait --timeout 5m
kubectl get all -n webapp
kubectl port-forward -n webapp svc/mynginx-mywebapp 8080:80
```

Open `http://localhost:8080` on the computer running port forwarding. Run the `port-forward` command in a separate terminal. If Kubernetes is on EC2, use an SSH tunnel or port-forward locally with a working kubeconfig.

### Included project files

| Path | Purpose |
| --- | --- |
| `mywebapp/Chart.yaml` | Chart metadata |
| `mywebapp/values.yaml` | Default NGINX image, two replicas, and ClusterIP service |
| `mywebapp/custom-values.yaml` | Override with four replicas |
| `mywebapp/templates/deployment.yaml` | Templated Kubernetes Deployment |
| `mywebapp/templates/service.yaml` | Templated Kubernetes Service |

> The chart sources are already committed. The `helm create` exercise below is optional for learning scaffolding. Deployments require a working Kubernetes cluster and network access to pull the NGINX image.

## 1. Helm architecture

Helm CLI → Kubernetes API server → rendered Kubernetes Deployment and Service → NGINX Pods. Helm stores release metadata in the target namespace, enabling upgrades and rollbacks.

## 2. Prerequisites

Verify that your Kubernetes cluster is running.

```
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
kubectl config current-context
```

All nodes should ideally show `Ready`.

## 3. Install Helm on Amazon Linux EC2

Run these commands on the Kubernetes control-plane node or any machine with working `kubectl` access.

```
sudo yum update -y

curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version
helm help
```

This installs Helm 3. For a production environment, review installation scripts and verify their authenticity before executing them.

## 4. Practice 1 – Add a Helm repository

```
helm repo list

helm repo add bitnami https://charts.bitnami.com/bitnami

helm repo update

helm repo list
```

Search available charts:

```
helm search repo nginx

helm search repo apache

helm search hub wordpress
```

Inspect a chart:

```
helm show chart bitnami/nginx

helm show values bitnami/nginx

helm show readme bitnami/nginx
```

## 5. Practice 2 – Install NGINX using Helm

For this exercise, use a lightweight training chart that creates an NGINX Deployment and Service without depending on an external chart's changing image requirements.

The ready-to-run chart is included in this repository under `mywebapp/` (the source for Practice 5). Clone this repository first, then install it with:

```
helm install mynginx ./mywebapp \
  --namespace webapp \
  --create-namespace
```

Verify installation:

```
helm list -A

helm status mynginx -n webapp

kubectl get all -n webapp
```

Access the application:

```
kubectl port-forward -n webapp svc/mynginx-mywebapp 8080:80
```

Open `http://localhost:8080` from the machine running port forwarding. For a remote EC2 server, use SSH port forwarding to access it from your laptop.

## 6. Practice 3 – Helm release management

```
helm list -A

helm status mynginx -n webapp

helm history mynginx -n webapp

helm get values mynginx -n webapp

helm get manifest mynginx -n webapp
```

## 7. Practice 4 – Create a namespace

```
kubectl create namespace development

kubectl get namespaces
```

Helm can manage releases separately in each namespace.

## 8. Practice 5 – Create a custom Helm Chart

Create a new chart:

```
# If following along with the existing repository:
cd mywebapp
ls -R

# To scaffold a new chart from scratch in a separate directory:
# helm create mywebapp
```

A Helm chart has this structure:

```
mywebapp/
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── serviceaccount.yaml
    ├── _helpers.tpl
    └── ...
```

When creating your own chart from scratch, remove the generated templates before adding the files shown below. Do not run this in the cloned project, because its templates are already provided:

```
rm -rf templates/*
```

Update `Chart.yaml`:

```
apiVersion: v2
name: mywebapp
description: Basic NGINX Helm practice
type: application
version: 0.1.0
appVersion: "1.27"
```

Update `values.yaml`:

```
replicaCount: 2

image:
  repository: nginx
  tag: "1.27"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
```

Create `templates/deployment.yaml`:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-{{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-{{ .Chart.Name }}
    spec:
      containers:
        - name: nginx
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 80
```

Create `templates/service.yaml`:

```
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}-{{ .Chart.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
```

Validate the chart:

```
helm lint .

helm template mynginx .

helm install mynginx . \
  -n webapp \
  --create-namespace \
  --wait \
  --timeout 5m
```

Run the last command only if you haven't already installed the `mynginx` release in Practice 2. For an existing release, use `helm upgrade mynginx . -n webapp --wait` instead.

## 9. Practice 6 – Upgrade an application

Update the replica count:

```
helm upgrade mynginx . \
  -n webapp \
  --set replicaCount=3 \
  --wait
```

Verify:

```
kubectl get deployments -n webapp

kubectl get pods -n webapp

helm history mynginx -n webapp
```

## 10. Practice 7 – Helm rollback

View release history:

```
helm history mynginx -n webapp
```

Rollback to revision 1:

```
helm rollback mynginx 1 -n webapp --wait
```

Verify:

```
kubectl get pods -n webapp

helm status mynginx -n webapp

helm history mynginx -n webapp
```

## 11. Practice 8 – Override Helm values

Create `custom-values.yaml` inside `mywebapp`:

```
replicaCount: 4

service:
  type: ClusterIP
  port: 80
```

Apply the configuration:

```
helm upgrade mynginx . \
  -n webapp \
  -f custom-values.yaml \
  --wait
```

Verify:

```
helm get values mynginx -n webapp

kubectl get deployments -n webapp
```

## 12. Practice 9 – Dry run and template validation

```
helm lint .

helm template mynginx .

helm upgrade mynginx . \
  -n webapp \
  --dry-run=client \
  --debug
```

These commands help detect template errors before deploying.

## 13. Practice 10 – Package a Helm Chart

Run from inside `mywebapp`:

```
cd ..

helm package mywebapp
```

Expected package name:

```
mywebapp-0.1.0.tgz
```

Install from the package:

```
helm install packaged-app ./mywebapp-0.1.0.tgz \
  -n development \
  --create-namespace \
  --wait
```

Verify:

```
helm list -A

kubectl get pods -n development
```

## 14. Practice 11 – Deploy multiple releases

Install the same chart under different release names:

```
helm install dev-app ./mywebapp \
  -n development \
  --create-namespace \
  --set replicaCount=1

helm install prod-app ./mywebapp \
  -n production \
  --create-namespace \
  --set replicaCount=3
```

Verify:

```
helm list -A

kubectl get pods -n development

kubectl get pods -n production
```

## 15. Practice 12 – Troubleshoot Helm deployments

```
helm status mynginx -n webapp

helm history mynginx -n webapp

kubectl describe deployment mynginx-mywebapp -n webapp

kubectl get events -n webapp --sort-by=.lastTimestamp

kubectl logs deployment/mynginx-mywebapp -n webapp
```

Common problems:

| Issue              | Possible cause                                            |
| ------------------ | --------------------------------------------------------- |
| `ImagePullBackOff` | Image unavailable or registry authentication issue        |
| `CrashLoopBackOff` | Container crashes repeatedly                              |
| `Pending`          | Insufficient resources or scheduling constraints          |
| `ErrImagePull`     | Invalid image name, tag, or registry access               |
| `UPGRADE FAILED`   | Invalid manifest, conflict, or resource readiness failure |

## 16. Practice 13 – Uninstall Helm releases

```
helm uninstall mynginx -n webapp

helm uninstall packaged-app -n development

helm uninstall dev-app -n development

helm uninstall prod-app -n production
```

Verify:

```
helm list -A

kubectl get all -n webapp
```

Optionally delete the lab namespaces:

```
kubectl delete namespace webapp development production
```

Only delete these namespaces if they were created exclusively for this practice.

## 17. Helm command cheat sheet

| Command            | Purpose                      |
| ------------------ | ---------------------------- |
| `helm version`     | Display Helm version         |
| `helm repo add`    | Add chart repository         |
| `helm repo update` | Refresh repository index     |
| `helm search repo` | Search repository charts     |
| `helm show values` | Inspect default chart values |
| `helm create`      | Generate chart structure     |
| `helm lint`        | Validate chart structure     |
| `helm template`    | Render Kubernetes YAML       |
| `helm install`     | Install application          |
| `helm list -A`     | List releases                |
| `helm status`      | Check release status         |
| `helm upgrade`     | Update application           |
| `helm history`     | View release revisions       |
| `helm rollback`    | Restore previous release     |
| `helm package`     | Package a chart              |
| `helm uninstall`   | Remove a release             |

## 18. Final practical assignment

Project: Deploy and Manage an NGINX Application Using Helm on Kubernetes

Complete these activities:

- Install Helm on an Amazon Linux EC2 instance.
- Verify connectivity to a kubeadm or EKS cluster.
- Add and search a Helm chart repository.
- Create a custom NGINX Helm chart.
- Deploy NGINX with two replicas.
- Access NGINX through port forwarding.
- Upgrade the deployment to four replicas.
- Use a custom values file.
- Roll back to an earlier revision.
- Package the chart as `.tgz`.
- Deploy separate development and production releases.
- Troubleshoot a deliberately incorrect image tag.
- Uninstall all lab releases.

Expected outcome: You will understand Helm chart creation, templating, configuration management, application deployment, upgrades, rollback, packaging, and release lifecycle management.

Official reference: [Helm Documentation](https://helm.sh/docs/?utm_source=chatgpt.com).
