My Flask App - Full Local Kubernetes/Helm/Docker Workflow
Overview

This project demonstrates:

    A simple Python Flask web app
    Dockerizing the app
    Storing code in GitHub
    Helm for Kubernetes manifests and deployment
    Running on a local Kubernetes cluster with Kind
    (Optional) GitHub Actions for CI

This documentation is meant for total beginners. Every folder, filename, and command is explained.
Folder Structure

my-flask-app/
├── src/
│   └── app.py
├── Dockerfile
├── requirements.txt
├── .gitignore
├── helm-chart/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── _helpers.tpl
│       ├── deployment.yaml
│       └── service.yaml
├── .github/
│   └── workflows/
│       └── ci-cd.yaml      # optional
├── README.md

1. Local Development
Set Up

Requirements:

    Python 3.x
    Docker
    Kind
    kubectl
    Helm

Develop and Test the App

Create the main app file:

src/app.py

from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello, Kubernetes World!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)

List the dependency:

requirements.txt

flask

Run locally:

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python src/app.py

Visit http://localhost:5000.
2. Dockerizing

Dockerfile

FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ .

EXPOSE 5000

CMD ["python", "app.py"]

Build and test image:

docker build -t my-flask-app:latest .
docker run -p 5000:5000 my-flask-app:latest

Open http://localhost:5000.
3. Source Control with GitHub

Typical .gitignore:

.gitignore

__pycache__/
*.pyc
*.pyo
*.pyd
.env
venv/
env/
*.sqlite3
docker_image.tar

Initialize repo and commit:

git init
git add .
git commit -m "Initial commit"

Push to GitHub:

    Create a GitHub repo named my-flask-app

git remote add origin https://github.com/your-username/my-flask-app.git
git branch -M main
git push -u origin main

4. Helm Chart for Kubernetes
Chart Metadata

helm-chart/Chart.yaml

apiVersion: v2
name: my-flask-app
description: A simple Flask app
version: 0.1.0
appVersion: "1.0"

Values

helm-chart/values.yaml

replicaCount: 1

image:
  repository: my-flask-app
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: NodePort
  port: 5000
  nodePort: 30007

resources:
  limits:
    cpu: 200m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 64Mi

Templates

_helpers.tpl
helm-chart/templates/_helpers.tpl

{{- define "my-flask-app.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "my-flask-app.name" -}}
{{- .Chart.Name -}}
{{- end -}}

Deployment
helm-chart/templates/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-flask-app.fullname" . }}
  labels:
    app: {{ include "my-flask-app.name" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "my-flask-app.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "my-flask-app.name" . }}
    spec:
      containers:
        - name: my-flask-app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 5000
          resources:
            limits:
              cpu: {{ .Values.resources.limits.cpu }}
              memory: {{ .Values.resources.limits.memory }}
            requests:
              cpu: {{ .Values.resources.requests.cpu }}
              memory: {{ .Values.resources.requests.memory }}

Service
helm-chart/templates/service.yaml

apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-flask-app.fullname" . }}
  labels:
    app: {{ include "my-flask-app.name" . }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 5000
      nodePort: {{ .Values.service.nodePort }}
  selector:
    app: {{ include "my-flask-app.name" . }}

5. Kubernetes with Kind
Step A: Create cluster with port mapping

Create kind-config.yaml:

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30007
        hostPort: 30007
        protocol: TCP

Create the cluster:

kind create cluster --config kind-config.yaml

Step B: Load and Deploy

docker build -t my-flask-app:latest .
kind load docker-image my-flask-app:latest
helm upgrade --install my-flask-app helm-chart/

Step C: Verify

kubectl get pods
kubectl get svc

You should see your service on PORT(S) 5000:30007/TCP.

Visit: http://localhost:30007
(Alternative) Test with port-forward if needed

kubectl port-forward svc/my-flask-app-my-flask-app 5000:5000

Then visit http://localhost:5000.
6. (Optional) GitHub Actions: CI Example

.github/workflows/ci-cd.yaml

name: CI/CD Pipeline

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.x
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Test code
        run: |
          python src/app.py & sleep 5
          curl -f http://localhost:5000

This job checks that the Flask app runs and serves HTTP 200 OK for /.
7. Best Practices

    Never commit secrets: Only configure via Kubernetes secrets, not in your repo!
    Separate code/configs/charts (see structure above)
    Use a .gitignore, and README.md for documentation
    Set CPU and memory resource limits in your templates
    Change app code, then rebuild the Docker image and reload into Kind every time!

8. Common Problems & Solutions

    Pod ImagePullBackOff: Did you reload image into Kind? (kind load docker-image ...)
    NodePort not accessible: Did you create Kind cluster with extraPortMappings? See above.
    Nothing at localhost: Pod may be CrashLoopBackOff, run kubectl logs <pod> to debug.

9. Clean Up

Stop Kind cluster and free resources:

kind delete cluster

10. Further Reading

    Kind Documentation
    Helm Documentation
    Python Flask
    Docker Documentation

Enjoy learning cloud-native development, one easy step at a time! 🚀
