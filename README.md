# 🚀 Deploy Elastic ELK Stack on Kubernetes using ECK

This guide provides a complete, step-by-step walkthrough to deploy the **ELK Stack (Elasticsearch, Logstash, Kibana)** using **Elastic Cloud on Kubernetes (ECK)**.

---

## 📋 Prerequisites

- ✅ Kubernetes cluster (v1.21+ recommended)
- ✅ `kubectl` configured and connected to your cluster
- ✅ Cluster resources: Minimum 2–4 CPUs and 8–16 GB RAM
- ✅ [Helm](https://helm.sh/) (for Logstash, optional)

---

## 📦 Step 1: Install the ECK Operator

Install the Custom Resource Definitions (CRDs) and Operator:

```bash
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/operator.yaml
```

## 📦 Step 2: Create a Namespace for ELK
```bash
kubectl create namespace elastic
```

## 📦 Step 3: Deploy Elasticsearch
Create a file named elasticsearch.yaml with the following content:
```bash
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
  namespace: elastic
spec:
  version: 8.13.4
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 2Gi
              cpu: 1

```
and then apply
```bash
kubectl apply -f elasticsearch.yaml
```

## 📦 Step 4: Deploy Kibana
Create a file named kibana.yaml:
```bash
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: quickstart
  namespace: elastic
spec:
  version: 8.13.4
  count: 1
  elasticsearchRef:
    name: quickstart
```
Apply it:
```bash
kubectl apply -f kibana.yaml
```


## 📦 Step 5: Deploy Logstash
Add Elastic Helm repo:
```bash
helm repo add elastic https://helm.elastic.co
helm repo update
```
Install Logstash:
```bash
helm install logstash elastic/logstash -n elastic
```
Configure via values, so create a logstash-values.yaml:
```bash
logstashConfig:
  logstash.yml: |
    http.host: "0.0.0.0"
    xpack.monitoring.enabled: false

pipeline:
  logstash.conf: |
    input {
      tcp {
        port => 5044
        codec => json_lines
      }
    }
    output {
      elasticsearch {
        hosts => ["http://quickstart-es-http.elastic.svc:9200"]
        user => "elastic"
        password => "${ELASTIC_PASSWORD}"
      }
    }
```
Next Install it
```bash
helm install logstash elastic/logstash -n elastic -f logstash-values.yaml
```
## 📦 Step 6: Access Kibana
Forward the Kibana service to your local port: if you want to use a ingress controller, please go to step 8
```bash
kubectl port-forward service/quickstart-kb-http 5601 -n elastic
```
Open http://localhost:5601 in your browser.

## 📦 Step 7: Get Elasticsearch Credentials
To retrieve the default elastic password from linux:
```bash
kubectl get secret quickstart-es-elastic-user -n elastic -o go-template='{{.data.elastic | base64decode}}'
```
To retrieve the default elastic password from windows:
```bash
[System.Text.Encoding]::UTF8.GetString([Convert]::FromBase64String(
    (kubectl get secret quickstart-es-elastic-user -n elastic -o json | ConvertFrom-Json).data.elastic
))
```
Use this password to log in to Kibana.

## 📦 Step 8: Using ingress controller
```bash
If your cluster doesn’t already have one, install the NGINX Ingress Controller:
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml

Wait until it's ready:
kubectl get pods -n ingress-nginx

Ensure Kibana is deployed in a way that exposes HTTP. By default, the ECK operator creates a Kubernetes Service named like: quickstart-kb-http
Check it with: kubectl get svc -n elastic

Create a file named kibana-ingress.yaml:

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana-ingress
  namespace: elastic
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: nginx
  rules:
  - host: kibana.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: quickstart-kb-http
            port:
              number: 5601

Apply the Ingress:
kubectl apply -f kibana-ingress.yaml

for windows: 
Edit your local /etc/hosts file (or C:\Windows\System32\drivers\etc\hosts on Windows) and add:
127.0.0.1 kibana.local
For cloud-based clusters, use the external IP of the ingress controller instead of 127.0.0.1.

for linux:
sudo nano /etc/hosts
Scroll to the bottom and add a line like:
127.0.0.1   kibana.local

now access the url
http://kibana.local

```


## 📚 Useful Resources
```bash
📖 [Deploy an orchestrator](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/deploy-an-orchestrator)
📦 [Elastic Helm Charts](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s/install-using-helm-chart)
```

## 👤 About the Author

**Ashan Malinda Perera**  
📍 United Arab Emirates  
🎓 MSc in IT | 💼 DevOps & Cloud Enthusiast | 🧩 Microservices Architect

- 📧 Email: [ashanp@gmail.com](mailto:ashanp@gmail.com)
- 🔗 LinkedIn: [[linkedin.com/in/ashanperera](https://www.linkedin.com/in/ashanperera)](https://www.linkedin.com/in/ashan-malinda-perera/)
- 💻 GitHub: [github.com/ashan-perera](https://github.com/ashan-perera)
---
> ⚡ Passionate about Kubernetes, ECK, cloud-native systems, and automation at scale.
