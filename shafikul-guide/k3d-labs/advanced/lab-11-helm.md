# 🔴 Lab 11 — Package Your App as a Helm Chart

**Level:** Advanced
**Time:** ~45 minutes

---

## 🎬 Scenario

Your company runs 12 microservices. Each one has nearly identical K8s YAML — just different names, images, and port numbers.

Your senior engineer says:

> *"We're copy-pasting the same deployment YAML 12 times. If we need to change something global — like add a new label or change the resource limits policy — we have to edit 12 files. This is unmaintainable. Create a Helm chart so we have one template, deployed with different values per service."*

**Your task:** Create a reusable Helm chart for a generic web service, then deploy 3 different services from the same chart with different values.

---

## 📋 Prerequisites

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

---

## 💡 What Helm Solves

```
Without Helm:
  service-a/deployment.yaml   (copy of base, different image)
  service-b/deployment.yaml   (copy of base, different image)
  service-c/deployment.yaml   (copy of base, different image)
  → Change resource policy = edit 12 files

With Helm:
  webservice-chart/            (one template)
  values-service-a.yaml        (just the overrides)
  values-service-b.yaml
  values-service-c.yaml
  → Change resource policy = edit chart once
```

---

## ✅ Full Solution

### Step 1: Create the Helm Chart

```bash
# Create chart scaffold
helm create webservice

# Look at what was generated
ls webservice/
# Chart.yaml    ← chart metadata
# values.yaml   ← default values
# templates/    ← your YAML templates with {{ }} placeholders
# charts/       ← dependencies
```

### Step 2: Simplify the Chart Templates

Replace `webservice/values.yaml` with:

```yaml
# webservice/values.yaml — sensible defaults
replicaCount: 2

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 80

resources:
  requests:
    cpu: "100m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env: []
#  - name: APP_ENV
#    value: production

labels: {}

healthCheck:
  enabled: true
  path: /
  port: 80
```

Replace `webservice/templates/deployment.yaml` with:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "webservice.fullname" . }}
  labels:
    {{- include "webservice.labels" . | nindent 4 }}
    {{- with .Values.labels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "webservice.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "webservice.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        {{- if .Values.env }}
        env:
          {{- toYaml .Values.env | nindent 10 }}
        {{- end }}
        {{- if .Values.healthCheck.enabled }}
        livenessProbe:
          httpGet:
            path: {{ .Values.healthCheck.path }}
            port: {{ .Values.healthCheck.port }}
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: {{ .Values.healthCheck.path }}
            port: {{ .Values.healthCheck.port }}
          initialDelaySeconds: 5
          periodSeconds: 10
        {{- end }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

### Step 3: Create Values Files Per Service

```yaml
# values-frontend.yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.25"
service:
  port: 80
  targetPort: 80
env:
  - name: APP_NAME
    value: "frontend"
  - name: APP_LOG_LEVEL
    value: "INFO"
labels:
  team: frontend
  tier: web
```

```yaml
# values-api.yaml
replicaCount: 4
image:
  repository: hashicorp/http-echo
  tag: "latest"
service:
  port: 80
  targetPort: 5678
healthCheck:
  enabled: false     # http-echo doesn't have /health
env:
  - name: APP_NAME
    value: "api"
labels:
  team: backend
  tier: api
```

```yaml
# values-worker.yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.24"    # worker still on older version
service:
  port: 80
  targetPort: 80
resources:
  requests:
    cpu: "200m"      # workers need more CPU
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
labels:
  team: backend
  tier: worker
```

### Step 4: Deploy All Three Services

```bash
kubectl create namespace production

# Deploy frontend
helm install frontend ./webservice \
  -f values-frontend.yaml \
  -n production

# Deploy API
helm install api ./webservice \
  -f values-api.yaml \
  -n production

# Deploy worker
helm install worker ./webservice \
  -f values-worker.yaml \
  -n production

# Check all services deployed
kubectl get deployments -n production
# NAME       READY   UP-TO-DATE   AVAILABLE
# frontend   3/3     3            3
# api        4/4     4            4
# worker     2/2     2            2

# List Helm releases
helm list -n production
# NAME      CHART           STATUS
# frontend  webservice-0.1  deployed
# api       webservice-0.1  deployed
# worker    webservice-0.1  deployed
```

### Step 5: Upgrade One Service

```bash
# Upgrade frontend to 5 replicas
helm upgrade frontend ./webservice \
  -f values-frontend.yaml \
  --set replicaCount=5 \
  -n production

# Or edit values-frontend.yaml and upgrade:
helm upgrade frontend ./webservice \
  -f values-frontend.yaml \
  -n production

# Check upgrade history
helm history frontend -n production
# REVISION  STATUS     CHART           DESCRIPTION
# 1         superseded webservice-0.1  Install complete
# 2         deployed   webservice-0.1  Upgrade complete
```

### Step 6: Roll Back Via Helm

```bash
# Roll back frontend to revision 1
helm rollback frontend 1 -n production

# Verify
helm history frontend -n production
kubectl get deployment frontend -n production
```

### Step 7: Change Global Policy — The Power of Helm

**Scenario:** Company policy changed: all services must have max CPU 300m.

```bash
# Edit webservice/values.yaml:
# limits.cpu: "300m"   ← change this once

# Upgrade ALL services with one command each
helm upgrade frontend ./webservice -f values-frontend.yaml -n production
helm upgrade api ./webservice -f values-api.yaml -n production
helm upgrade worker ./webservice -f values-worker.yaml -n production

# All 3 services updated. Without Helm: edit 3 separate YAML files manually.
```

---

## 🔍 Helm Command Reference

```bash
helm install <name> <chart> -f values.yaml -n namespace     # Deploy
helm upgrade <name> <chart> -f values.yaml -n namespace     # Update
helm rollback <name> <revision> -n namespace                # Roll back
helm uninstall <name> -n namespace                          # Delete
helm list -n namespace                                      # List releases
helm history <name> -n namespace                            # See history
helm template <name> <chart> -f values.yaml                 # Preview YAML
helm lint <chart>                                           # Validate chart
```

---

## 🔥 Bonus Challenges

**1. Preview what Helm will apply before deploying:**
```bash
helm template frontend ./webservice -f values-frontend.yaml
# Outputs the rendered YAML — review before applying
```

**2. Use Helm to install ArgoCD (instead of raw kubectl apply):**
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace
```

**3. Add Helm chart to ArgoCD for full GitOps:**
- Push your chart to GitHub
- Point ArgoCD at the chart path
- Now deployments are: `git commit values.yaml` → ArgoCD → cluster

```bash
# Cleanup
helm uninstall frontend api worker -n production
kubectl delete namespace production
```
