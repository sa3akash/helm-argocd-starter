To set up a CI/CD pipeline with **GitHub Actions**, **ArgoCD**, and **Kubernetes**, follow these steps. I'll guide you through the configuration for both **local** and **production** environments.

---

### 1. **Prerequisites**
- **GitHub Repository**: Codebase should be in a GitHub repository.
- **Kubernetes Cluster**: Local (e.g., Minikube, Kind) and production Kubernetes clusters.
- **ArgoCD**: Installed in your Kubernetes clusters (both local and production).
- **kubectl** and **ArgoCD CLI**: Installed on your local machine.
- **Docker**: Installed to build images locally.

---

### 2. **Install ArgoCD**
#### Local:
1. Install ArgoCD in your local Kubernetes cluster:
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```
2. Access the ArgoCD UI:
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```
   Login using the default credentials:
   ```bash
   # Username: admin
   # Password: Run the command below to get the default password
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
   ```

#### Production:
1. Install ArgoCD in your production Kubernetes cluster using the same steps as above.
2. Configure ingress or load balancer to access the ArgoCD server securely.
3. Change the admin password for production:
   ```bash
   argocd account update-password
   ```

---

### 3. **Set Up Kubernetes Deployment**
Prepare your Kubernetes manifests (e.g., `deployment.yaml`, `service.yaml`, etc.) in a GitHub repository. These manifests should reside in a dedicated directory (e.g., `k8s/`).

#### Example: `k8s/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: ghcr.io/<your-username>/<your-app>:latest
        ports:
        - containerPort: 3000
```

---

### 4. **GitHub Actions CI/CD Workflow**
Create a GitHub Actions workflow (`.github/workflows/deploy.yml`) to automate building, pushing Docker images, and updating ArgoCD.

#### Example Workflow:
```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Log in to GitHub Container Registry
      uses: docker/login-action@v2
      with:
        registry: ghcr.io
        username: ${{ secrets.GITHUB_ACTOR }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Build and Push Docker Image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ghcr.io/<your-username>/<your-app>:latest

    - name: Sync with ArgoCD
      env:
        ARGOCD_SERVER: ${{ secrets.ARGOCD_SERVER }}
        ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_AUTH_TOKEN }}
      run: |
        argocd app sync my-app
```

#### Notes:
- Replace `<your-username>` with your GitHub username.
- Set `ARGOCD_SERVER` and `ARGOCD_AUTH_TOKEN` as **GitHub Secrets**.

---

### 5. **Set Up ArgoCD Application**
Create an ArgoCD application that points to your GitHub repository.

#### Example: `argocd-application.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/<your-repo>.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply the application manifest:
```bash
kubectl apply -f argocd-application.yaml
```

---

### 6. **Test in Local and Production**
#### Local:
1. Commit and push your changes to the `main` branch.
2. Watch the pipeline in GitHub Actions to ensure the workflow runs successfully.
3. Verify that ArgoCD syncs the application in the local Kubernetes cluster.

#### Production:
1. Set up a GitOps pipeline for your production cluster, pointing to the production-specific manifests or configurations.
2. Use the same GitHub Actions workflow, but differentiate environments with a branch or directory structure (e.g., `k8s/prod`).

---

### 7. **Monitor and Manage**
- **ArgoCD Dashboard**: Use the UI or CLI to monitor application status.
- **GitHub Actions Logs**: Check logs for any CI/CD issues.

---

By following this setup, you'll have an end-to-end pipeline using GitHub Actions, ArgoCD, and Kubernetes for both local and production environments. Let me know if you need further details or troubleshooting help!







---

Setting up a **complete CI/CD pipeline for a MERN stack application** with **Prometheus** and **Grafana** for monitoring requires several components. Here's a step-by-step guide to configure this setup for **local (Minikube)** and **production (AWS)** environments.

---

### **Architecture Overview**
1. **Frontend**: React (client-side).
2. **Backend**: Express.js (API server).
3. **Database**: MongoDB.
4. **CI/CD Pipeline**: GitHub Actions.
5. **Containerization**: Docker and Kubernetes (Minikube for local and EKS for AWS).
6. **Monitoring**: Prometheus and Grafana.

---

### 1. **Prerequisites**
- Local Kubernetes cluster (**Minikube**) installed.
- AWS CLI configured with IAM roles and permissions for **EKS**.
- ArgoCD installed in both **local** and **production** Kubernetes clusters.
- Docker installed for building images locally.
- GitHub repository containing your MERN stack app.

---

### 2. **Setup Kubernetes Cluster**
#### Local (Minikube):
1. Start Minikube:
   ```bash
   minikube start --driver=docker
   ```
2. Enable Ingress:
   ```bash
   minikube addons enable ingress
   ```

#### Production (AWS EKS):
1. Create an EKS cluster using eksctl:
   ```bash
   eksctl create cluster --name mern-prod --region us-east-1 --nodegroup-name standard-workers --node-type t3.medium --nodes 3
   ```
2. Configure kubectl to use the EKS cluster:
   ```bash
   aws eks --region us-east-1 update-kubeconfig --name mern-prod
   ```

---

### 3. **Dockerize Your MERN Stack Application**
Create Dockerfiles for each component (frontend, backend, and MongoDB if required).

#### Backend (`Dockerfile`):
```dockerfile
FROM node:18-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
```

#### Frontend (`Dockerfile`):
```dockerfile
FROM node:18-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npx", "serve", "-s", "build"]
```

---

### 4. **Prepare Kubernetes Manifests**
Create Kubernetes manifests for deployment, service, and ingress.

#### Backend (`k8s/backend-deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: <your-backend-image>
        ports:
        - containerPort: 5000
        env:
        - name: MONGO_URI
          value: mongodb://mongo-service:27017/mydatabase
```

#### Frontend (`k8s/frontend-deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: <your-frontend-image>
        ports:
        - containerPort: 3000
```

#### MongoDB (`k8s/mongo-deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongo
        image: mongo:latest
        ports:
        - containerPort: 27017
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
spec:
  ports:
  - port: 27017
    targetPort: 27017
  selector:
    app: mongo
```

---

### 5. **CI/CD with GitHub Actions**
Create a `.github/workflows/deploy.yml` file for CI/CD.

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Log in to DockerHub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and Push Backend
      run: |
        docker build -t <your-dockerhub-username>/mern-backend:latest -f backend/Dockerfile ./backend
        docker push <your-dockerhub-username>/mern-backend:latest

    - name: Build and Push Frontend
      run: |
        docker build -t <your-dockerhub-username>/mern-frontend:latest -f frontend/Dockerfile ./frontend
        docker push <your-dockerhub-username>/mern-frontend:latest

    - name: Apply Kubernetes Manifests
      env:
        KUBECONFIG: ${{ secrets.KUBECONFIG }}
      run: |
        kubectl apply -f k8s/
```

---

### 6. **Install Prometheus and Grafana**
#### Local:
1. Install Prometheus and Grafana with Helm:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update

   helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
   ```

2. Access Grafana Dashboard:
   ```bash
   kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
   ```

#### Production:
1. Use the same Helm commands to install Prometheus and Grafana in your AWS EKS cluster.
2. Configure Ingress or LoadBalancer for external access.

---

### 7. **Integrate Prometheus with Your App**
Add instrumentation to your backend API using libraries like `prom-client` for Node.js.

#### Backend Metrics Example:
```javascript
const client = require('prom-client');
const express = require('express');
const app = express();

const collectDefaultMetrics = client.collectDefaultMetrics;
collectDefaultMetrics();

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.send(await client.register.metrics());
});

app.listen(5000, () => console.log('Server running on port 5000'));
```

Expose `/metrics` endpoint in your backend, and configure Prometheus to scrape it.

#### Prometheus Scrape Config:
Update `prometheus.yaml`:
```yaml
scrape_configs:
  - job_name: 'backend'
    static_configs:
      - targets: ['backend-service:5000']
```

---

### 8. **Deploy and Verify**
#### Local:
1. Use `kubectl apply -f k8s/` to deploy your app in Minikube.
2. Access services via `minikube service <service-name>` or ingress.

#### Production:
1. Apply manifests to your EKS cluster:
   ```bash
   kubectl apply -f k8s/
   ```
2. Verify services and pods:
   ```bash
   kubectl get pods -n default
   kubectl get svc -n default
   ```

---

### 9. **Monitoring in Grafana**
1. Add Prometheus as a data source in Grafana.
2. Create dashboards to monitor application metrics (e.g., request rate, error rate, resource usage).

---

### 10. **Summary of Key Components**
- **GitHub Actions**: Automates Docker builds and Kubernetes deployments.
- **ArgoCD**: Syncs Kubernetes manifests with GitHub repository.
- **Prometheus**: Monitors backend metrics and resource usage.
- **Grafana**: Visualizes Prometheus metrics.
- **Minikube**: Local testing environment.
- **AWS EKS**: Production-grade Kubernetes environment.

Let me know if you need help with specific configurations or debugging!


---


### **Part 1: Local Setup Using Minikube**
This part details how to configure your MERN stack application with **Prometheus**, **Grafana**, and **ArgoCD** in a **local Minikube environment**.

---

### **1. Setting Up Minikube**
1. Start Minikube:
   ```bash
   minikube start --driver=docker
   ```

2. Enable required Minikube add-ons:
   ```bash
   minikube addons enable ingress
   minikube addons enable metrics-server
   ```

3. Set up a namespace for your application and monitoring:
   ```bash
   kubectl create namespace mern-local
   kubectl create namespace monitoring
   ```

---

### **2. Install ArgoCD in Minikube**
1. Install ArgoCD:
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. Expose the ArgoCD UI:
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```

3. Get the initial admin password:
   ```bash
   kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode
   ```

4. Access ArgoCD UI at `https://localhost:8080` and log in using `admin` and the retrieved password.

---

### **3. Deploy MERN Stack to Minikube**
#### Prepare Kubernetes manifests for the MERN stack (frontend, backend, and MongoDB):
Store the following YAML files in a `k8s` folder.

- **Backend Deployment**: `k8s/backend-deployment.yaml`
- **Frontend Deployment**: `k8s/frontend-deployment.yaml`
- **MongoDB Deployment**: `k8s/mongo-deployment.yaml`

#### Apply Kubernetes Manifests:
```bash
kubectl apply -f k8s/ -n mern-local
```

---

### **4. Install Prometheus and Grafana in Minikube**
1. Add the Prometheus Helm repo:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

2. Install Prometheus and Grafana:
   ```bash
   helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
   ```

3. Expose Grafana:
   ```bash
   kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
   ```

   Access Grafana at `http://localhost:3000`.

4. Configure Prometheus to scrape your backend metrics:
   Add this to `prometheus.yaml` under `scrape_configs`:
   ```yaml
   scrape_configs:
     - job_name: 'backend'
       static_configs:
         - targets: ['backend-service.mern-local:5000']
   ```

---

### **5. Test Application Locally**
1. Use `minikube service` to access your services:
   ```bash
   minikube service <service-name> -n mern-local
   ```

2. Check monitoring dashboards in Grafana at `http://localhost:3000`.

---

### **Part 2: Production Setup Using AWS and Terraform**

For the production setup, we will use **AWS EKS**, **Terraform**, and **ArgoCD**. Prometheus and Grafana will also be deployed for monitoring.

---

### **1. Prerequisites**
- AWS CLI configured with a valid IAM role.
- Terraform installed.

---

### **2. Create AWS EKS Cluster with Terraform**
1. Create a Terraform configuration file `eks-cluster.tf`:

```hcl
provider "aws" {
  region = "us-east-1"
}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "mern-prod-cluster"
  cluster_version = "1.25"
  subnets         = ["<subnet-ids>"]
  vpc_id          = "<vpc-id>"

  node_groups = {
    eks_nodes = {
      desired_capacity = 3
      max_capacity     = 5
      min_capacity     = 2

      instance_type = "t3.medium"
    }
  }
}

output "kubeconfig" {
  value = module.eks.kubeconfig
  sensitive = true
}
```

2. Initialize and apply Terraform:
   ```bash
   terraform init
   terraform apply
   ```

   This creates an EKS cluster.

3. Update `kubectl` to use the new cluster:
   ```bash
   aws eks --region us-east-1 update-kubeconfig --name mern-prod-cluster
   ```

---

### **3. Install ArgoCD in Production**
1. Install ArgoCD:
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. Expose ArgoCD with a LoadBalancer service:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: argocd-server
     namespace: argocd
   spec:
     type: LoadBalancer
     ports:
       - port: 80
         targetPort: 443
     selector:
       app.kubernetes.io/name: argocd-server
   ```

3. Access ArgoCD from the LoadBalancer's public IP.

---

### **4. Deploy MERN Stack to EKS**
1. Use the same Kubernetes manifests (`k8s/` folder) from the local setup.
2. Apply them to the production namespace:
   ```bash
   kubectl apply -f k8s/ -n mern-prod
   ```

---

### **5. Install Prometheus and Grafana in EKS**
1. Add the Prometheus Helm repo:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

2. Install Prometheus and Grafana:
   ```bash
   helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
   ```

3. Expose Grafana with a LoadBalancer:
   ```bash
   kubectl edit svc prometheus-grafana -n monitoring
   ```

   Change the `type` to `LoadBalancer`.

4. Access Grafana via the LoadBalancer's public IP.

---

### **6. Monitor Your Application**
1. Add Prometheus as a data source in Grafana.
2. Create dashboards to monitor backend metrics and Kubernetes cluster health.

---

### **Summary**
- **Local**: Minikube with Kubernetes, ArgoCD, Prometheus, and Grafana.
- **Production**: AWS EKS provisioned via Terraform, ArgoCD, Prometheus, and Grafana for production-grade deployment and monitoring.

Let me know which section you'd like to dive deeper into!




---


### **Advanced Local Configuration: Minikube with NodePort, Local DNS, and HTTPS using Let's Encrypt**

In this advanced configuration, we'll set up **NodePort services** for ArgoCD and Grafana, configure a **local DNS** using `sa3akash.com`, and secure your setup with **Let's Encrypt** SSL certificates for HTTPS.

---

### **1. Configure Local DNS**

1. Add an entry to your `hosts` file:
   On Linux/MacOS, edit `/etc/hosts`. On Windows, edit `C:\Windows\System32\drivers\etc\hosts`.

   Add:
   ```plaintext
   127.0.0.1   sa3akash.com argocd.sa3akash.com grafana.sa3akash.com
   ```

   This resolves `sa3akash.com` and subdomains to your local Minikube cluster.

2. Ensure Minikube is configured to bind traffic to your host:
   ```bash
   minikube start --driver=docker --addons=ingress
   ```

---

### **2. Set Up ArgoCD with NodePort**

1. **Edit the ArgoCD Server Service** to use `NodePort`:
   ```bash
   kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort", "ports": [{"port": 443, "nodePort": 32080, "targetPort": 8080}]}}'
   ```

   This exposes ArgoCD on port `32080`.

2. Access ArgoCD locally:
   - Visit `https://argocd.sa3akash.com:32080`.

---

### **3. Set Up Grafana with NodePort**

1. **Edit the Grafana Service**:
   ```bash
   kubectl patch svc prometheus-grafana -n monitoring -p '{"spec": {"type": "NodePort", "ports": [{"port": 80, "nodePort": 32081}]}}'
   ```

   This exposes Grafana on port `32081`.

2. Access Grafana locally:
   - Visit `http://grafana.sa3akash.com:32081`.

---

### **4. Enable HTTPS with Let's Encrypt**

To use **Let's Encrypt** for HTTPS locally, you'll use a combination of **Certbot** and the **Minikube ingress** controller.

---

#### **4.1. Install Certbot**

1. Install Certbot:
   ```bash
   sudo apt update
   sudo apt install certbot
   ```

2. Use the DNS challenge (e.g., `acme-dns`) to issue a wildcard certificate for your domain:
   ```bash
   sudo certbot certonly --manual --preferred-challenges dns \
   -d "*.sa3akash.com" -d "sa3akash.com"
   ```

3. Certbot will prompt you to add a DNS TXT record for validation. Add the record in your DNS configuration or use a DNS server if managing locally.

4. After verification, Certbot will save your certificates at `/etc/letsencrypt/live/sa3akash.com/`.

---

#### **4.2. Configure Minikube Ingress for HTTPS**

1. Create a Kubernetes Secret for the TLS certificate:
   ```bash
   kubectl create secret tls sa3akash-tls \
   --cert=/etc/letsencrypt/live/sa3akash.com/fullchain.pem \
   --key=/etc/letsencrypt/live/sa3akash.com/privkey.pem \
   -n ingress-nginx
   ```

2. Set up an Ingress resource for your services:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: sa3akash-ingress
     namespace: argocd
     annotations:
       kubernetes.io/ingress.class: "nginx"
       cert-manager.io/cluster-issuer: "letsencrypt-prod"
   spec:
     tls:
       - hosts:
           - argocd.sa3akash.com
           - grafana.sa3akash.com
         secretName: sa3akash-tls
     rules:
       - host: argocd.sa3akash.com
         http:
           paths:
             - path: /
               pathType: Prefix
               backend:
                 service:
                   name: argocd-server
                   port:
                     number: 443
       - host: grafana.sa3akash.com
         http:
           paths:
             - path: /
               pathType: Prefix
               backend:
                 service:
                   name: prometheus-grafana
                   port:
                     number: 80
   ```

3. Apply the Ingress configuration:
   ```bash
   kubectl apply -f ingress.yaml
   ```

4. Access services with HTTPS:
   - ArgoCD: `https://argocd.sa3akash.com`
   - Grafana: `https://grafana.sa3akash.com`

---

### **5. Test the Setup**

1. Validate DNS resolution:
   ```bash
   ping sa3akash.com
   ```

   Ensure it resolves to `127.0.0.1`.

2. Validate HTTPS using `curl`:
   ```bash
   curl -k https://argocd.sa3akash.com
   curl -k https://grafana.sa3akash.com
   ```

3. Access the dashboards via your browser:
   - `https://argocd.sa3akash.com`
   - `https://grafana.sa3akash.com`

---

### **Key Notes**
1. **Certbot Renewal**:
   - Automate Let's Encrypt renewal using a cron job:
     ```bash
     sudo crontab -e
     ```
     Add:
     ```bash
     0 0 * * * certbot renew --quiet && kubectl delete secret sa3akash-tls -n ingress-nginx && kubectl create secret tls sa3akash-tls --cert=/etc/letsencrypt/live/sa3akash.com/fullchain.pem --key=/etc/letsencrypt/live/sa3akash.com/privkey.pem -n ingress-nginx
     ```

2. **Ingress Controller**:
   Ensure Minikube’s ingress controller is running:
   ```bash
   kubectl get pods -n ingress-nginx
   ```

3. **Local HTTPS**:
   Using Let's Encrypt locally requires a valid DNS setup (ensure your domain points to `127.0.0.1`).

---

### **Summary**

- **NodePort**: Exposed ArgoCD and Grafana on custom ports (`32080` and `32081`).
- **Local DNS**: Configured `sa3akash.com` and subdomains in `/etc/hosts`.
- **HTTPS**: Enabled using Let's Encrypt and Minikube ingress with a wildcard certificate.

Would you like to add more advanced configurations, like logging for your services or custom dashboards in Grafana?



---


Since you're using Windows and Minikube, the commands `sudo apt update` and `sudo apt install certbot` are meant for a Linux environment. However, Minikube runs inside a VM or container, so these commands can't be directly used in your Windows system. Instead, we can use a Kubernetes **Job** to run Certbot within a temporary container to issue and renew Let's Encrypt certificates.

---

### **Steps to Run Certbot in Kubernetes**

We'll create a Kubernetes **Job** for issuing certificates using Certbot. This approach doesn't rely on a Linux environment on your Windows machine and directly integrates with your Minikube cluster.

---

### **Certbot Kubernetes YAML Configuration**

1. **Create a Certbot Job** YAML file (`certbot-job.yaml`):
   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: certbot
     namespace: default
   spec:
     template:
       spec:
         containers:
           - name: certbot
             image: certbot/certbot:latest
             command:
               - /bin/sh
               - -c
               - |
                 certbot certonly --manual --preferred-challenges dns \
                 -d "*.sa3akash.com" -d "sa3akash.com" \
                 --agree-tos --email your-email@example.com \
                 --non-interactive \
                 --manual-public-ip-logging-ok
             env:
               - name: CERTBOT_DOMAIN
                 value: "sa3akash.com"
               - name: EMAIL
                 value: "your-email@example.com"
         restartPolicy: Never
   ```

   Replace:
   - `your-email@example.com` with your email address.
   - `sa3akash.com` with your actual domain.

2. **Apply the Job**:
   ```bash
   kubectl apply -f certbot-job.yaml
   ```

3. **Check Job Logs**:
   ```bash
   kubectl logs job/certbot
   ```

   Certbot will prompt you to add a DNS TXT record. You'll need to update your domain's DNS configuration (on your DNS provider) to verify ownership.

4. **Save the Certificates**:
   After verification, Certbot will generate certificates. These can be copied to a Kubernetes **Secret** for use with your Ingress.

---

### **Store the Certificates in Kubernetes**

1. Create a Kubernetes Secret for the TLS certificate:
   ```bash
   kubectl create secret tls sa3akash-tls \
   --cert=/path/to/fullchain.pem \
   --key=/path/to/privkey.pem \
   -n ingress-nginx
   ```

   Replace `/path/to/fullchain.pem` and `/path/to/privkey.pem` with the paths where Certbot saved the certificates.

2. Update your Ingress to use the TLS Secret:
   ```yaml
   tls:
     - hosts:
         - sa3akash.com
       secretName: sa3akash-tls
   ```

---

### **Automate Certificate Renewal**

1. Create a CronJob for Certbot Renewal:
   ```yaml
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: certbot-renewal
     namespace: default
   spec:
     schedule: "0 0 * * *"  # Run daily at midnight
     jobTemplate:
       spec:
         template:
           spec:
             containers:
               - name: certbot
                 image: certbot/certbot:latest
                 command:
                   - /bin/sh
                   - -c
                   - |
                     certbot renew && \
                     kubectl delete secret sa3akash-tls -n ingress-nginx && \
                     kubectl create secret tls sa3akash-tls \
                     --cert=/etc/letsencrypt/live/sa3akash.com/fullchain.pem \
                     --key=/etc/letsencrypt/live/sa3akash.com/privkey.pem \
                     -n ingress-nginx
             volumeMounts:
               - name: certs
                 mountPath: /etc/letsencrypt
             restartPolicy: OnFailure
             volumes:
               - name: certs
                 emptyDir: {}
   ```

2. Apply the CronJob:
   ```bash
   kubectl apply -f certbot-renewal.yaml
   ```

---

### **Summary**

- The **Certbot Job** issues a wildcard certificate for your domain.
- The certificate is stored as a **Kubernetes Secret**.
- The **Ingress** uses this Secret for HTTPS.
- A **CronJob** automates certificate renewal.

This setup allows you to use Certbot within Minikube and ensures HTTPS for your local DNS (`sa3akash.com`). Let me know if you need help troubleshooting or additional advanced features!




---

In production, the process will be similar but with a few adjustments to ensure scalability, security, and compatibility with AWS infrastructure. Below is the step-by-step process for using Certbot in production with AWS, Terraform, and Kubernetes.

---

### **Production Setup for Certbot on AWS**

In production, Certbot can run as a **Kubernetes Job** or through a dedicated **Terraform setup** for issuing and renewing certificates using Route 53 (AWS DNS). Certificates are then stored in Kubernetes Secrets and used by your Ingress controller.

---

### **Using Certbot with AWS Route 53 in Kubernetes**

This setup assumes:
- You’re using AWS Route 53 for DNS.
- You’ve deployed your production Kubernetes cluster with Terraform.

---

#### **1. Certbot Kubernetes Job for AWS**

Create a **Certbot Job** that uses AWS Route 53 to manage DNS challenges for issuing certificates.

1. **Certbot Job YAML for Production** (`certbot-job.yaml`):
   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: certbot
     namespace: default
   spec:
     template:
       spec:
         containers:
           - name: certbot
             image: certbot/dns-route53:latest
             command:
               - /bin/sh
               - -c
               - |
                 certbot certonly \
                 --dns-route53 \
                 --agree-tos \
                 --non-interactive \
                 --email your-email@example.com \
                 -d "*.yourdomain.com" -d "yourdomain.com"
             env:
               - name: AWS_ACCESS_KEY_ID
                 valueFrom:
                   secretKeyRef:
                     name: aws-credentials
                     key: aws-access-key
               - name: AWS_SECRET_ACCESS_KEY
                 valueFrom:
                   secretKeyRef:
                     name: aws-credentials
                     key: aws-secret-key
         restartPolicy: Never
   ```

2. **Create AWS Credentials Secret**:
   Use your AWS access key and secret key to allow Certbot to create DNS records in Route 53.

   ```bash
   kubectl create secret generic aws-credentials \
   --from-literal=aws-access-key=<AWS_ACCESS_KEY_ID> \
   --from-literal=aws-secret-key=<AWS_SECRET_ACCESS_KEY>
   ```

3. **Run the Certbot Job**:
   ```bash
   kubectl apply -f certbot-job.yaml
   ```

4. **Store the Certificates in a Kubernetes Secret**:
   ```bash
   kubectl create secret tls yourdomain-tls \
   --cert=/etc/letsencrypt/live/yourdomain.com/fullchain.pem \
   --key=/etc/letsencrypt/live/yourdomain.com/privkey.pem \
   -n ingress-nginx
   ```

5. **Update Ingress with the TLS Secret**:
   ```yaml
   tls:
     - hosts:
         - yourdomain.com
       secretName: yourdomain-tls
   ```

---

#### **2. Automate Certificate Renewal with CronJob**

Set up a Kubernetes **CronJob** to automatically renew certificates and update the TLS secret.

1. **CronJob YAML for Renewal** (`certbot-renewal.yaml`):
   ```yaml
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: certbot-renewal
     namespace: default
   spec:
     schedule: "0 0 * * *"  # Run daily at midnight
     jobTemplate:
       spec:
         template:
           spec:
             containers:
               - name: certbot
                 image: certbot/dns-route53:latest
                 command:
                   - /bin/sh
                   - -c
                   - |
                     certbot renew && \
                     kubectl delete secret yourdomain-tls -n ingress-nginx && \
                     kubectl create secret tls yourdomain-tls \
                     --cert=/etc/letsencrypt/live/yourdomain.com/fullchain.pem \
                     --key=/etc/letsencrypt/live/yourdomain.com/privkey.pem \
                     -n ingress-nginx
             env:
               - name: AWS_ACCESS_KEY_ID
                 valueFrom:
                   secretKeyRef:
                     name: aws-credentials
                     key: aws-access-key
               - name: AWS_SECRET_ACCESS_KEY
                 valueFrom:
                   secretKeyRef:
                     name: aws-credentials
                     key: aws-secret-key
             volumeMounts:
               - name: certs
                 mountPath: /etc/letsencrypt
             restartPolicy: OnFailure
             volumes:
               - name: certs
                 emptyDir: {}
   ```

2. **Apply the CronJob**:
   ```bash
   kubectl apply -f certbot-renewal.yaml
   ```

---

### **Terraform Setup for Production**

To streamline production configuration, Terraform can automate the setup of DNS records and Kubernetes resources.

#### **Terraform Example**

1. **DNS Records for Route 53 (Terraform)**:
   Use Terraform to configure Route 53 records for certificate issuance.

   ```hcl
   resource "aws_route53_record" "certbot_dns" {
     zone_id = "<your_zone_id>"
     name    = "_acme-challenge.yourdomain.com"
     type    = "TXT"
     ttl     = 60
     records = [aws_acm_certificate.cert.validation_record.0.value]
   }
   ```

2. **Apply Terraform**:
   ```bash
   terraform init
   terraform apply
   ```

3. **Deploy Kubernetes Resources**:
   Integrate Terraform with your Kubernetes cluster using the `kubernetes_manifest` provider to apply the Certbot Job and CronJob YAMLs directly from Terraform.

---

### **Summary of Production Setup**

1. **Certbot** runs as a Kubernetes Job to issue and renew certificates using AWS Route 53 DNS challenges.
2. Certificates are stored as Kubernetes **Secrets**.
3. A **CronJob** automates certificate renewal and updates the Secret.
4. Terraform manages DNS records in AWS and optionally applies Kubernetes resources for a fully automated workflow.

Let me know if you'd like further details or adjustments!


---


To deploy a MERN stack application to a **VPS** server using **GitHub Actions**, **ArgoCD**, and **Kubernetes**, we need to:

1. **Set up the VPS for Kubernetes** (install Kubernetes and ArgoCD).
2. **Deploy the MERN stack on Kubernetes** using ArgoCD.
3. **Automate CI/CD with GitHub Actions** for building and deploying the application.
4. **Use Let's Encrypt for HTTPS** on the VPS for your domain.

Here’s the complete setup:

---

## **1. VPS Setup for Kubernetes and ArgoCD**

### **Step 1. Install Kubernetes on VPS**
Run these commands on your VPS:

1. Install Docker and Kubernetes tools:
   ```bash
   sudo apt update
   sudo apt install -y docker.io curl apt-transport-https
   curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
   echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   sudo apt update
   sudo apt install -y kubelet kubeadm kubectl
   sudo systemctl enable docker
   ```

2. Initialize Kubernetes cluster:
   ```bash
   sudo kubeadm init --pod-network-cidr=192.168.0.0/16
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

3. Install a CNI (e.g., Calico):
   ```bash
   kubectl apply -f https://docs.projectcalico.org/v3.25/manifests/calico.yaml
   ```

4. Join additional worker nodes (if any):
   Use the `kubeadm join` command printed during `kubeadm init`.

---

### **Step 2. Install ArgoCD**
1. Install ArgoCD:
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. Expose ArgoCD server using `NodePort`:
   ```bash
   kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort", "ports": [{"port": 80, "targetPort": 8080, "nodePort": 30080}]}}'
   ```

3. Get ArgoCD admin password:
   ```bash
   kubectl get pods -n argocd
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
   ```

4. Access ArgoCD:
   Visit `http://<VPS_IP>:30080` and log in with `admin` and the password retrieved above.

---

### **Step 3. Install Ingress Controller**
1. Install Nginx Ingress:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
   ```

2. Verify the installation:
   ```bash
   kubectl get pods -n ingress-nginx
   ```

---

## **2. Deploy MERN Stack Using ArgoCD**

### **Step 1. Create Kubernetes Manifests for MERN**

1. **Backend Deployment (`backend-deployment.yaml`)**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: backend
     namespace: default
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: backend
     template:
       metadata:
         labels:
           app: backend
       spec:
         containers:
         - name: backend
           image: your-dockerhub-user/backend:latest
           ports:
           - containerPort: 5000
           env:
           - name: MONGO_URI
             value: "mongodb://mongo:27017/mydatabase"
   ```

2. **Frontend Deployment (`frontend-deployment.yaml`)**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: frontend
     namespace: default
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: frontend
     template:
       metadata:
         labels:
           app: frontend
       spec:
         containers:
         - name: frontend
           image: your-dockerhub-user/frontend:latest
           ports:
           - containerPort: 3000
   ```

3. **MongoDB Deployment (`mongo-deployment.yaml`)**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: mongo
     namespace: default
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: mongo
     template:
       metadata:
         labels:
           app: mongo
       spec:
         containers:
         - name: mongo
           image: mongo:latest
           ports:
           - containerPort: 27017
           volumeMounts:
           - name: mongo-data
             mountPath: /data/db
         volumes:
         - name: mongo-data
           persistentVolumeClaim:
             claimName: mongo-pvc
   ```

4. **MongoDB PVC (`mongo-pvc.yaml`)**:
   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: mongo-pvc
     namespace: default
   spec:
     accessModes:
     - ReadWriteOnce
     resources:
       requests:
         storage: 5Gi
   ```

5. **Ingress for MERN (`mern-ingress.yaml`)**:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: mern-ingress
     namespace: default
     annotations:
       kubernetes.io/ingress.class: "nginx"
   spec:
     rules:
     - host: sa3akash.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: frontend
               port:
                 number: 3000
         - path: /api
           pathType: Prefix
           backend:
             service:
               name: backend
               port:
                 number: 5000
   ```

---

### **Step 2. Use ArgoCD for Deployment**

1. Create an ArgoCD Application for MERN:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: mern-stack
     namespace: argocd
   spec:
     destination:
       namespace: default
       server: https://kubernetes.default.svc
     source:
       repoURL: https://github.com/your-repo/mern-stack-k8s
       path: manifests
       targetRevision: main
     project: default
   ```

2. Apply the ArgoCD Application:
   ```bash
   kubectl apply -f argo-application.yaml
   ```

---

## **3. Automate Deployment with GitHub Actions**

1. **GitHub Actions Workflow (`mern-ci-cd.yml`)**:
   ```yaml
   name: MERN CI/CD

   on:
     push:
       branches:
         - main

   jobs:
     build-and-deploy:
       runs-on: ubuntu-latest
       steps:
       - name: Checkout code
         uses: actions/checkout@v3

       - name: Set up Docker
         uses: docker/setup-buildx-action@v2

       - name: Login to DockerHub
         run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

       - name: Build and push backend
         run: |
           docker build -t your-dockerhub-user/backend:latest ./backend
           docker push your-dockerhub-user/backend:latest

       - name: Build and push frontend
         run: |
           docker build -t your-dockerhub-user/frontend:latest ./frontend
           docker push your-dockerhub-user/frontend:latest

       - name: Sync ArgoCD
         run: kubectl apply -f argo-application.yaml
         env:
           KUBECONFIG: ${{ secrets.KUBECONFIG }}
   ```

---

## **4. Enable HTTPS with Let's Encrypt**

1. Add TLS to the Ingress:
   Update `mern-ingress.yaml` with TLS configuration:
   ```yaml
   tls:
   - hosts:
     - sa3akash.com
     secretName: sa3akash-tls
   ```

2. Use Certbot to Get Certificates:
   ```bash
   certbot certonly --standalone -d sa3akash.com
   ```

3. Add the Certificate as a Secret:
   ```bash
   kubectl create secret tls sa3akash-tls --cert=/etc/letsencrypt/live/sa3akash.com/fullchain.pem --key=/etc/letsencrypt/live/sa3akash.com/privkey.pem
   ```

---

This setup ensures a production-grade deployment for your MERN stack on a VPS with CI/CD, ArgoCD, and HTTPS using Kubernetes. Let me know if you'd like additional optimizations!


---




Here’s the complete setup for a **local Kubernetes cluster using KIND** with **1 master node** and **4 worker nodes**, and configuration for **auto-scaling** and **auto-healing** using Horizontal Pod Autoscaler (HPA) and Kubernetes native features. This setup will also include a **load balancer** using MetalLB for local environments.

---

### Step 1: Create KIND Cluster Configuration File
Define a custom KIND configuration to specify the cluster topology with 1 master node and 4 worker nodes.

```yaml
# kind-cluster-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: false
  podSubnet: "10.244.0.0/16" # Flannel default
  serviceSubnet: "10.96.0.0/12"
```

---

### Step 2: Create the KIND Cluster

Run the following command to create the cluster:

```bash
kind create cluster --config kind-cluster-config.yaml --name mern-cluster
```

---

### Step 3: Install Network Plugin (Flannel)

Install Flannel as the CNI (Container Network Interface) to enable pod-to-pod communication:

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

---

### Step 4: Set Up MetalLB (Load Balancer for Local)

Install MetalLB, a load balancer solution for bare-metal Kubernetes clusters.

1. **Install MetalLB:**

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
   ```

2. **Configure MetalLB with a pool of IP addresses:**

   Create a ConfigMap for MetalLB to allocate IP addresses in your local network range.

   ```yaml
   # metallb-config.yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     namespace: metallb-system
     name: config
   data:
     config: |
       address-pools:
       - name: default
         protocol: layer2
         addresses:
         - 192.168.1.240-192.168.1.250
   ```

   Apply the configuration:

   ```bash
   kubectl apply -f metallb-config.yaml
   ```

---

### Step 5: Deploy Metrics Server (For Autoscaling)

Install the Metrics Server to enable HPA (Horizontal Pod Autoscaler):

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Edit the Metrics Server deployment to allow insecure TLS (for local environments):

```bash
kubectl edit deployment metrics-server -n kube-system
```

Add the following arguments under the `spec.containers.args` section:

```yaml
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP
```

Restart the Metrics Server:

```bash
kubectl rollout restart deployment metrics-server -n kube-system
```

---

### Step 6: Deploy the MERN Stack

1. **MongoDB Deployment:**

   Create a `mongo-deployment.yaml` file:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: mongo
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: mongo
     template:
       metadata:
         labels:
           app: mongo
       spec:
         containers:
         - name: mongo
           image: mongo:5.0
           ports:
           - containerPort: 27017
           volumeMounts:
           - name: mongo-data
             mountPath: /data/db
         volumes:
         - name: mongo-data
           emptyDir: {}
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: mongo
   spec:
     ports:
     - port: 27017
       targetPort: 27017
     selector:
       app: mongo
   ```

   Apply the MongoDB deployment:

   ```bash
   kubectl apply -f mongo-deployment.yaml
   ```

2. **Backend Deployment (Node.js):**

   Create a `backend-deployment.yaml` file:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: backend
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: backend
     template:
       metadata:
         labels:
           app: backend
       spec:
         containers:
         - name: backend
           image: your-backend-image:latest
           ports:
           - containerPort: 5000
           env:
           - name: MONGO_URI
             value: mongodb://mongo:27017/mern
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: backend
   spec:
     ports:
     - port: 5000
       targetPort: 5000
     selector:
       app: backend
   ```

   Apply the Backend deployment:

   ```bash
   kubectl apply -f backend-deployment.yaml
   ```

3. **Frontend Deployment (React):**

   Create a `frontend-deployment.yaml` file:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: frontend
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: frontend
     template:
       metadata:
         labels:
           app: frontend
       spec:
         containers:
         - name: frontend
           image: your-frontend-image:latest
           ports:
           - containerPort: 3000
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: frontend
   spec:
     ports:
     - port: 80
       targetPort: 3000
     selector:
       app: frontend
   ```

   Apply the Frontend deployment:

   ```bash
   kubectl apply -f frontend-deployment.yaml
   ```

---

### Step 7: Configure HPA for Backend

Create an HPA configuration for the Backend service:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply the HPA:

```bash
kubectl apply -f backend-hpa.yaml
```

---

### Step 8: Enable Ingress for Local DNS and HTTPS with Let's Encrypt

1. **Install Cert-Manager:**

   ```bash
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
   ```

2. **Create an Issuer:**

   ```yaml
   # issuer.yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt
   spec:
     acme:
       server: https://acme-v02.api.letsencrypt.org/directory
       email: your-email@example.com
       privateKeySecretRef:
         name: letsencrypt-key
       solvers:
       - http01:
           ingress:
             class: nginx
   ```

   Apply the Issuer:

   ```bash
   kubectl apply -f issuer.yaml
   ```

3. **Create Ingress for Your App:**

   ```yaml
   # ingress.yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: mern-ingress
     annotations:
       cert-manager.io/cluster-issuer: "letsencrypt"
   spec:
     tls:
     - hosts:
       - sa3akash.com
       secretName: sa3akash-com-tls
     rules:
     - host: sa3akash.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: frontend
               port:
                 number: 80
   ```

   Apply the Ingress:

   ```bash
   kubectl apply -f ingress.yaml
   ```

---

### Step 9: Test Your Cluster

Access the services locally using `sa3akash.com`. Verify HTTPS is working with Let's Encrypt. Test auto-scaling by applying a load to the backend service.

--- 

This setup ensures advanced local configurations with KIND, HTTPS, and auto-scaling features.


---

To deploy a **MERN stack** application in a **production Kubernetes cluster** on a high-performance VPS server, integrated with **ArgoCD for GitOps**, **GitHub Actions for CI/CD**, and including advanced production features like:

1. **Auto-scaling** and **Auto-healing** (Horizontal Pod Autoscaler, Node Auto-healing).
2. **Load Balancer** with HTTPS (using **Let's Encrypt**).
3. **Monitoring** (with **Prometheus** and **Grafana**).

Here’s the complete guide:

---

### Step 1: Set Up Kubernetes Cluster on VPS (With High Availability)

1. **Install Kubernetes using Kubeadm**  
   On your VPS, initialize the Kubernetes cluster with `kubeadm`:
   ```bash
   sudo kubeadm init --pod-network-cidr=192.168.0.0/16
   ```

2. **Install Flannel CNI:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
   ```

3. **Join Worker Nodes (Optional for Scaling):**  
   If you have multiple VPS servers, join additional worker nodes using the `kubeadm join` command.

---

### Step 2: Install and Configure ArgoCD

1. **Install ArgoCD:**
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. **Expose ArgoCD with Load Balancer:**  
   Create a NodePort Service for ArgoCD:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: argocd-server
     namespace: argocd
   spec:
     type: NodePort
     ports:
     - port: 80
       targetPort: 8080
       nodePort: 30080
     selector:
       app.kubernetes.io/name: argocd-server
   ```

   Apply:
   ```bash
   kubectl apply -f argocd-server-service.yaml
   ```

3. **Access ArgoCD:**  
   Open `http://<VPS-IP>:30080` in your browser.

4. **Login to ArgoCD:**  
   Get the default password:
   ```bash
   kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
   ```

---

### Step 3: Configure GitHub Actions for CI/CD

Create a `.github/workflows/deploy.yml` file in your repository:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Set up Docker
      uses: docker/setup-buildx-action@v2

    - name: Build and Push Docker Image
      env:
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      run: |
        docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
        docker build -t $DOCKER_USERNAME/mern-app:latest .
        docker push $DOCKER_USERNAME/mern-app:latest

    - name: Sync to ArgoCD
      env:
        ARGOCD_SERVER: ${{ secrets.ARGOCD_SERVER }}
        ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_AUTH_TOKEN }}
      run: |
        argocd app sync mern-app --server $ARGOCD_SERVER --auth-token $ARGOCD_AUTH_TOKEN
```

---

### Step 4: Deploy MERN Stack to Kubernetes

1. **MongoDB Deployment:**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: mongodb
     labels:
       app: mongodb
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: mongodb
     template:
       metadata:
         labels:
           app: mongodb
       spec:
         containers:
         - name: mongodb
           image: mongo:5.0
           ports:
           - containerPort: 27017
           volumeMounts:
           - name: mongo-data
             mountPath: /data/db
         volumes:
         - name: mongo-data
           persistentVolumeClaim:
             claimName: mongo-pvc
   ---
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: mongo-pvc
   spec:
     accessModes:
     - ReadWriteOnce
     resources:
       requests:
         storage: 10Gi
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: mongodb
   spec:
     ports:
     - port: 27017
       targetPort: 27017
     selector:
       app: mongodb
   ```

2. **Backend Deployment:**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: backend
     labels:
       app: backend
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: backend
     template:
       metadata:
         labels:
           app: backend
       spec:
         containers:
         - name: backend
           image: your-backend-image:latest
           ports:
           - containerPort: 5000
           env:
           - name: MONGO_URI
             value: mongodb://mongodb:27017/mern
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: backend
   spec:
     ports:
     - port: 5000
       targetPort: 5000
     selector:
       app: backend
   ```

3. **Frontend Deployment:**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: frontend
     labels:
       app: frontend
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: frontend
     template:
       metadata:
         labels:
           app: frontend
       spec:
         containers:
         - name: frontend
           image: your-frontend-image:latest
           ports:
           - containerPort: 80
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: frontend
   spec:
     ports:
     - port: 80
       targetPort: 80
     selector:
       app: frontend
   ```

---

### Step 5: Enable Autoscaling and Auto-healing

1. **Horizontal Pod Autoscaler (HPA):**
   ```yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: backend-hpa
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: backend
     minReplicas: 3
     maxReplicas: 10
     metrics:
     - type: Resource
       resource:
         name: cpu
         target:
           type: Utilization
           averageUtilization: 50
   ```

2. **Enable Node Auto-healing:**  
   Configure Kubernetes to restart failed pods automatically using default pod health checks.

---

### Step 6: Add Prometheus and Grafana Monitoring

1. **Install Prometheus:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml
   ```

2. **Install Grafana:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/grafana/helm-charts/main/charts/grafana/templates/all.yaml
   ```

3. **Expose Prometheus and Grafana:**
   Use a NodePort or LoadBalancer service for Prometheus and Grafana access.

---

### Step 7: HTTPS with Let's Encrypt

1. **Install Cert-Manager:**
   ```bash
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
   ```

2. **Create a ClusterIssuer for Let's Encrypt:**
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt
   spec:
     acme:
       server: https://acme-v02.api.letsencrypt.org/directory
       email: your-email@example.com
       privateKeySecretRef:
         name: letsencrypt-key
       solvers:
       - http01:
           ingress:
             class: nginx
   ```

3. **Configure HTTPS for Ingress:**
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: mern-ingress
     annotations:
       cert-manager.io/cluster-issuer: "letsencrypt"
   spec:
     tls:
     - hosts:
       - yourdomain.com
       secretName: yourdomain-com-tls
     rules:
     - host: yourdomain.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: frontend
               port:
                 number: 80
   ```

---

### Step 8: Monitor and Maintain the Cluster

- Use **Prometheus** to monitor cluster metrics.
- Use **Grafana dashboards** for detailed visualization

.
- Ensure ArgoCD and GitHub Actions sync the latest code without manual intervention.

Would you like further assistance with configuration files or a specific part of the setup?


---

Here’s a **step-by-step guide** to deploy your **MERN stack application** on a **production Kubernetes cluster** with **GitOps (ArgoCD)**, **GitHub Actions for CI/CD**, **auto-scaling, auto-healing**, **Prometheus & Grafana for monitoring**, and **Let's Encrypt for HTTPS**.

The guide ensures nothing is missed and is designed for a **high-performance VPS server** running Kubernetes with **kind (Kubernetes in Docker)** and one master node with 4 worker nodes.

---

## **Step 1: Set Up the Kubernetes Cluster Using Kind**

### 1.1 Install Required Tools
On your VPS, install:
- Docker
- Kubernetes `kubectl`
- `kind` (Kubernetes in Docker)

#### Install Docker:
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```

#### Install kubectl:
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

#### Install kind:
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x kind
sudo mv kind /usr/local/bin/
```

---

### 1.2 Create the Kubernetes Cluster
Create a `kind-config.yaml` file to define a cluster with **1 master node and 4 worker nodes**:
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
  - role: worker
```

Create the cluster:
```bash
kind create cluster --config kind-config.yaml
```

Verify the cluster is running:
```bash
kubectl get nodes
```

---

## **Step 2: Install ArgoCD**

### 2.1 Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2.2 Expose ArgoCD with a LoadBalancer Service
Create a file `argocd-service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: argocd-server
  namespace: argocd
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app.kubernetes.io/name: argocd-server
```

Apply the service:
```bash
kubectl apply -f argocd-service.yaml
```

---

### 2.3 Access ArgoCD
Get the external IP:
```bash
kubectl get svc -n argocd
```

Visit `http://<EXTERNAL-IP>` to access the ArgoCD dashboard.  
Retrieve the default admin password:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

---

## **Step 3: Set Up GitHub Actions for CI/CD**

Create a `.github/workflows/deploy.yml` in your MERN stack repository:
```yaml
name: Deploy to Kubernetes

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Repository
      uses: actions/checkout@v3

    - name: Build Docker Images
      run: |
        docker build -t <your-docker-username>/backend:latest ./backend
        docker build -t <your-docker-username>/frontend:latest ./frontend
        docker push <your-docker-username>/backend:latest
        docker push <your-docker-username>/frontend:latest

    - name: Deploy to ArgoCD
      run: |
        argocd app sync mern-stack --auth-token ${{ secrets.ARGOCD_AUTH_TOKEN }}
```

---

## **Step 4: Deploy the MERN Stack**

### 4.1 MongoDB Deployment
Create `mongodb.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:5.0
        ports:
        - containerPort: 27017
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  ports:
  - port: 27017
    targetPort: 27017
  selector:
    app: mongodb
```

---

### 4.2 Backend Deployment
Create `backend.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: <your-docker-username>/backend:latest
        env:
        - name: MONGO_URI
          value: mongodb://mongodb:27017/mern
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  ports:
  - port: 5000
    targetPort: 5000
  selector:
    app: backend
```

---

### 4.3 Frontend Deployment
Create `frontend.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: <your-docker-username>/frontend:latest
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: frontend
```

---

### 4.4 Ingress with Let's Encrypt
Install Cert-Manager:
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

Create an `Ingress` for your domain:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mern-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt"
spec:
  tls:
  - hosts:
    - your-domain.com
    secretName: mern-tls
  rules:
  - host: your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

---

## **Step 5: Auto-Scaling and Monitoring**

### Horizontal Pod Autoscaler
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### Install Prometheus and Grafana
```bash
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml
kubectl apply -f https://raw.githubusercontent.com/grafana/helm-charts/main/charts/grafana/templates/all.yaml
```

---

This setup ensures a production-ready, scalable, and monitored MERN stack deployment. Let me know if you need help with specific configuration files!


---


For a production environment, using **Kind (Kubernetes in Docker)** is **not recommended**. Kind is best suited for **local development and testing** because it runs Kubernetes clusters within Docker containers, which isn’t ideal for high-performance, high-availability, or scalable production systems.

For **production** deployments, especially for a **sensitive application**, I recommend setting up a **Kubernetes cluster on a cloud provider** like **AWS, Google Cloud, or Azure** or using **VMs**/bare-metal servers. You should also focus on using **managed services** and **cloud-native solutions** for higher availability, reliability, and scaling.

Here’s a **best-practice approach** with **advanced configurations**:

### **1. Kubernetes Cluster Setup (Production-Grade)**

#### **Option A: Using Cloud Provider's Managed Kubernetes (e.g., AWS EKS)**

**AWS EKS (Elastic Kubernetes Service)** provides a fully managed Kubernetes service with built-in scaling, security, and integration with other AWS services. This option is ideal for production as it abstracts a lot of the underlying infrastructure management and offers **high availability** and **scalability**.

1. **Create an EKS Cluster**:
   Use **AWS Management Console**, **AWS CLI**, or **Terraform** to set up your Kubernetes cluster. Here's how to set it up with **AWS CLI**:

   ```bash
   aws eks create-cluster --name mern-cluster \
   --role-arn arn:aws:iam::your-account-id:role/eks-cluster-role \
   --resources-vpc-config subnetIds=subnet-abc123,subnet-def456
   ```

   Follow AWS documentation for detailed steps: [EKS Setup](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html)

2. **Set Up Worker Nodes**:
   EKS will automatically provision worker nodes (EC2 instances) or you can set them up manually with the **EKS Managed Node Group**.

#### **Option B: Self-Managed Kubernetes Cluster**

For **VPS** servers or **bare-metal hardware**, you can manually deploy Kubernetes on your own VMs (or use tools like **Kubespray** or **Rancher**).

1. **Provision VMs or Bare-Metal Servers** for Kubernetes master and worker nodes.
2. Install **Kubernetes** on the VMs manually or using a tool like **kubeadm**.

   Example of **kubeadm** installation on each node:

   ```bash
   sudo apt-get update && sudo apt-get install -y apt-transport-https curl
   curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
   echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

---

### **2. Advanced Configuration: Auto-Scaling, Auto-Healing, Load Balancer**

For a **sensitive app**, we need advanced configurations to ensure that your application can **scale automatically** and **recover** from failures.

#### **2.1 Horizontal Pod Autoscaler (HPA)**

Set up an HPA for your **backend** and **frontend** services based on resource usage (CPU/Memory).

Example for **backend** autoscaling based on **CPU usage**:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

#### **2.2 Auto-Healing with Pod Disruption Budgets**

Define **Pod Disruption Budgets (PDB)** to ensure **high availability** and **resilience** by specifying minimum availability during voluntary disruptions (e.g., upgrades).

Example for **backend**:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: backend
```

#### **2.3 Load Balancer Setup**

In AWS, you can set up **Application Load Balancer (ALB)** or **Network Load Balancer (NLB)**. With **AWS EKS**, Kubernetes will automatically provision an **ALB** for services exposed via **LoadBalancer** type.

**Example for backend service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
```

---

### **3. Monitoring and Observability: Prometheus and Grafana**

#### **3.1 Prometheus Setup**
1. **Install Prometheus** using **Helm** in your Kubernetes cluster.
   First, add the Prometheus Helm chart repository:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

   Install Prometheus:
   ```bash
   helm install prometheus prometheus-community/kube-prometheus-stack
   ```

   This will deploy Prometheus, Alertmanager, and Grafana.

#### **3.2 Grafana Setup**
You can expose **Grafana** via a **LoadBalancer** or an **Ingress** for access.

To install Grafana (if it's not installed with Prometheus):
```bash
helm install grafana grafana/grafana
```

You can access Grafana using the following service definition:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
spec:
  selector:
    app: grafana
  ports:
    - port: 3000
      targetPort: 3000
  type: LoadBalancer
```

#### **3.3 Dashboard and Alerts**
- Set up **Prometheus dashboards** in Grafana to monitor CPU, Memory, and Custom metrics from your MERN stack.
- Configure **Alertmanager** in Prometheus to send notifications if any metric crosses a threshold (e.g., high CPU utilization).

---

### **4. Use Let's Encrypt for HTTPS**

To configure **Let's Encrypt** for **automatic SSL/TLS** certificates, use **cert-manager** in Kubernetes.

#### **4.1 Install cert-manager**
```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml
```

#### **4.2 Create ClusterIssuer for Let's Encrypt**
Create a `ClusterIssuer` resource that will allow you to request certificates from **Let's Encrypt**.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your-email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

Apply this configuration:
```bash
kubectl apply -f cluster-issuer.yaml
```

#### **4.3 Configure Ingress for HTTPS**
Create an Ingress resource with **TLS** enabled:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mern-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - your-domain.com
    secretName: mern-tls
  rules:
  - host: your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

---

### **5. Deploy with ArgoCD and GitHub Actions**

#### **5.1 Setup ArgoCD for GitOps**
- Sync your code with **ArgoCD** to automatically deploy any changes from your GitHub repository.
- You should configure **GitHub Actions** to trigger a **deployment pipeline** and sync with **ArgoCD**.

Example GitHub Action:
```yaml
name: Deploy MERN Stack to Kubernetes

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Build Docker images
      run: |
        docker build -t <your-docker-username>/backend .
        docker build -t <your-docker-username>/frontend .

    - name: Push Docker images
      run: |
        docker push <your-docker-username>/backend
        docker push <your-docker-username>/frontend

    - name: Sync ArgoCD
      run: |
        argocd app sync mern-app --auth-token ${{ secrets.ARGOCD_AUTH_TOKEN }}
```

---

### **Conclusion**

This setup ensures that your **MERN stack** is highly **available**, **scalable**, and **monitored** with **Prometheus** and **Grafana**. You also benefit from **automatic scaling**, **auto-healing**, and---



---



To set up a **production-ready environment** for your MERN stack on a **VPS** with **32 GB RAM** and **12-core CPU**, we’ll focus on creating a Kubernetes cluster with **high availability, auto-scaling, auto-healing**, **monitoring**, and **security** configurations. I’ll break this setup down into stages for better clarity.

### **Prerequisites for VPS Setup**
Ensure you have the following installed on your VPS:

1. **Ubuntu 20.04+** (or another supported Linux distro)
2. **Docker** for containerization
3. **Kubernetes** and **kubectl**
4. **Helm** for easy deployment of Kubernetes resources
5. **Nginx** for Ingress Controller (with SSL)
6. **ArgoCD** for GitOps deployment
7. **Cert-Manager** for automatic Let's Encrypt SSL certificates
8. **Prometheus** and **Grafana** for monitoring

---

### **Step 1: Set Up Kubernetes Cluster**

#### **1.1 Install Kubernetes (kubeadm, kubelet, kubectl)**

1. **Update your VPS:**

    ```bash
    sudo apt-get update
    sudo apt-get upgrade -y
    ```

2. **Install Docker:**

    ```bash
    sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh
    sudo usermod -aG docker $USER
    ```

3. **Install Kubernetes tools (`kubeadm`, `kubelet`, `kubectl`):**

    ```bash
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    sudo apt-add-repository "deb https://apt.kubernetes.io/ kubernetes-xenial main"
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

4. **Initialize Kubernetes Cluster (Master Node)**:

   On the master node (the first VPS), run:

    ```bash
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16
    ```

   After successful initialization, run the following commands as suggested by `kubeadm` to set up your local kubeconfig:

    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

5. **Install Pod Network Plugin (Flannel or Calico)**:

   Install **Flannel** as the network plugin for your cluster:

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    ```

6. **Join Worker Nodes to the Cluster**:
   On each worker node (your other VPS instances), run the `kubeadm join` command generated during the `kubeadm init` process on the master node.

---

### **Step 2: Set Up Nginx Ingress Controller for Load Balancing and SSL**

#### **2.1 Install Nginx Ingress Controller**:

1. **Install the Nginx Ingress Controller** via Helm:

    ```bash
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm install ingress-nginx ingress-nginx/ingress-nginx --set controller.replicaCount=2 --set controller.nodeSelector."kubernetes\.io/role"=ingress
    ```

#### **2.2 Set Up Ingress Resource for SSL**:

Create an **Ingress** resource to manage your **backend** and **frontend** services, with SSL:

1. **Create Ingress YAML (ingress.yml):**

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: mern-ingress
      annotations:
        cert-manager.io/cluster-issuer: "letsencrypt-prod"
    spec:
      tls:
      - hosts:
        - your-domain.com
        secretName: mern-tls
      rules:
      - host: your-domain.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
    ```

2. **Create Secret and Apply Ingress**:

    ```bash
    kubectl apply -f ingress.yml
    ```

---

### **Step 3: Install Cert-Manager for SSL Certificates**

1. **Install Cert-Manager**:

    ```bash
    kubectl apply -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml
    ```

2. **Create ClusterIssuer for Let's Encrypt** (Create `cluster-issuer.yml`):

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod
    spec:
      acme:
        email: your-email@example.com
        server: https://acme-v02.api.letsencrypt.org/directory
        privateKeySecretRef:
          name: letsencrypt-prod
        solvers:
        - http01:
            ingress:
              class: nginx
    ```

   Apply it:

    ```bash
    kubectl apply -f cluster-issuer.yml
    ```

---

### **Step 4: Set Up Prometheus and Grafana for Monitoring**

#### **4.1 Install Prometheus using Helm**:

1. **Add Prometheus Helm Chart**:

    ```bash
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    ```

2. **Install Prometheus**:

    ```bash
    helm install prometheus prometheus-community/kube-prometheus-stack
    ```

   This will install Prometheus, Alertmanager, and Grafana on your cluster.

#### **4.2 Access Grafana Dashboard**:

1. Expose **Grafana** via LoadBalancer (create `grafana-service.yml`):

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: grafana
    spec:
      selector:
        app.kubernetes.io/name: grafana
      ports:
        - port: 3000
          targetPort: 3000
      type: LoadBalancer
    ```

   Apply the service:

    ```bash
    kubectl apply -f grafana-service.yml
    ```

2. Get the external IP of Grafana:

    ```bash
    kubectl get svc grafana
    ```

---

### **Step 5: Auto-Scaling and Auto-Healing**

#### **5.1 Set Up Horizontal Pod Autoscaler (HPA)**

1. Create a `HorizontalPodAutoscaler` for your **backend** service (e.g., scale based on CPU usage):

    ```yaml
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: backend-hpa
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: backend
      minReplicas: 3
      maxReplicas: 10
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 50
    ```

   Apply the HPA:

    ```bash
    kubectl apply -f hpa.yml
    ```

#### **5.2 Set Up Pod Disruption Budget (PDB)**

1. Create `pod-disruption-budget.yml` to ensure high availability:

    ```yaml
    apiVersion: policy/v1
    kind: PodDisruptionBudget
    metadata:
      name: backend-pdb
    spec:
      minAvailable: 2
      selector:
        matchLabels:
          app: backend
    ```

   Apply the PDB:

    ```bash
    kubectl apply -f pod-disruption-budget.yml
    ```

---

### **Step 6: GitOps with ArgoCD and GitHub Actions**

#### **6.1 Install ArgoCD**:

1. Install ArgoCD in your Kubernetes cluster:

    ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```

2. Expose ArgoCD API server:

    ```bash
    kubectl expose svc argocd-server -n argocd --type=LoadBalancer --name=argocd-server
    ```

#### **6.2 Configure GitHub Actions**:

1. Create a GitHub Actions workflow to build and deploy your MERN stack to the Kubernetes cluster, triggering on `push` events to `main` or `production` branches.

   Example:

    ```yaml
    name: Build and Deploy MERN Stack

    on:
      push:
        branches:
          - main

    jobs:
      build:
        runs-on: ubuntu-latest
        steps:
        - name: Checkout code
          uses: actions/checkout@v2

        - name: Build Docker images
          run: |
            docker build -t your-docker-username/backend .
            docker build -t your-docker-username/frontend .

        - name: Push Docker images
          run:

|
docker push your-docker-username/backend
docker push your-docker-username/frontend

        - name: Set up kubectl
          uses: azure/setup-kubectl@v2
          with:
            kubeconfig: ${{ secrets.KUBECONFIG }}

        - name: Deploy to Kubernetes
          run: |
            kubectl apply -f k8s/backend-deployment.yml
            kubectl apply -f k8s/frontend-deployment.yml
    ```

This setup ensures **high availability**, **scalability**, **monitoring**, and **security** for your MERN stack. Let me know if you'd like more specific configuration for other services!



---


Here's a **step-by-step guide** to set up your **MERN stack** in **Kubernetes** with **ArgoCD**, **GitHub Actions**, **Prometheus & Grafana**, **Let's Encrypt SSL certificates**, and all the necessary configurations for **production**. We will cover each of the components in the order they need to be set up, ensuring that nothing is missed.

---

### **Step 1: Set Up the Kubernetes Cluster**

Since you're deploying to **a VPS** (with **32GB RAM, 12-core CPU**), we’ll use **Kubernetes** to manage your application. We'll use **Kubeadm** to set up the cluster, but you could also use **K3s** or **Docker Desktop** if you'd prefer.

1. **Install Docker** on all nodes (Master and Worker):
   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io
   sudo systemctl enable docker
   sudo systemctl start docker
   ```

2. **Install Kubernetes** on all nodes:
   On each node (Master and Workers):
   ```bash
   curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
   sudo apt-add-repository "deb https://apt.kubernetes.io/ kubernetes-xenial main"
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

3. **Set up Kubernetes Cluster** on Master Node:
   On your master node:
   ```bash
   sudo kubeadm init --pod-network-cidr=10.244.0.0/16
   ```

4. **Set up kubectl** for the Master Node:
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

5. **Install Flannel (Network Plugin)** on Master:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   ```

6. **Join Worker Nodes** to the Cluster:
   Run the `kubeadm join` command on each worker node (as instructed by the `kubeadm init` output on the master node).

---

### **Step 2: Set Up ArgoCD**

1. **Install ArgoCD** in your Kubernetes Cluster:
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. **Expose ArgoCD Server** (using LoadBalancer or Port-Forward):
   ```bash
   kubectl expose svc argocd-server -n argocd --type=LoadBalancer --name=argocd-server
   ```

3. **Access ArgoCD UI**:
   Get the external IP for ArgoCD:
   ```bash
   kubectl get svc -n argocd
   ```

4. **Login to ArgoCD**:
   By default, the password is the name of the `argocd-server` pod:
   ```bash
   kubectl get pods -n argocd
   kubectl describe secret argocd-initial-admin-secret -n argocd
   ```

   Use the **ArgoCD UI** to log in.

---

### **Step 3: Set Up GitHub Actions for CI/CD**

1. **Create a `.github/workflows/ci-cd.yml`** file in your repository:

   ```yaml
   name: Build and Deploy MERN Stack

   on:
     push:
       branches:
         - main

   jobs:
     build:
       runs-on: ubuntu-latest
       steps:
         - name: Checkout code
           uses: actions/checkout@v2

         - name: Set up Docker Buildx
           uses: docker/setup-buildx-action@v2

         - name: Login to Docker Hub
           uses: docker/login-action@v2
           with:
             username: ${{ secrets.DOCKER_USERNAME }}
             password: ${{ secrets.DOCKER_PASSWORD }}

         - name: Build Docker images
           run: |
             docker build -t your-docker-username/frontend ./frontend
             docker build -t your-docker-username/backend ./backend

         - name: Push Docker images
           run: |
             docker push your-docker-username/frontend
             docker push your-docker-username/backend

         - name: Set up kubectl
           uses: azure/setup-kubectl@v2
           with:
             kubeconfig: ${{ secrets.KUBECONFIG }}

         - name: Deploy to Kubernetes
           run: |
             kubectl apply -f k8s/backend-deployment.yaml
             kubectl apply -f k8s/frontend-deployment.yaml
   ```

2. **GitHub Secrets**:
   - `DOCKER_USERNAME`: Docker Hub username.
   - `DOCKER_PASSWORD`: Docker Hub password.
   - `KUBECONFIG`: Base64 encoded `kubeconfig` file from your Kubernetes setup.

---

### **Step 4: Install Prometheus and Grafana**

1. **Install Prometheus and Grafana** using Helm:

   Add the Prometheus Helm repository:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

2. **Install Prometheus Stack**:
   ```bash
   helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring
   ```

3. **Access Grafana UI**:
   Expose Grafana through LoadBalancer or port-forward:
   ```bash
   kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
   ```

   Access Grafana at `http://localhost:3000` using the default username (`admin`) and password (`prom-operator`).

---

### **Step 5: Install Cert-Manager and Configure SSL with Let's Encrypt**

1. **Install Cert-Manager**:

   Create the namespace and install Cert-Manager:
   ```bash
   kubectl create namespace cert-manager
   kubectl apply -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml
   ```

2. **Set up a ClusterIssuer** (for Let's Encrypt):

   Create a `cluster-issuer.yml`:
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt-prod
   spec:
     acme:
       email: your-email@example.com
       server: https://acme-v02.api.letsencrypt.org/directory
       privateKeySecretRef:
         name: letsencrypt-prod
       solvers:
       - http01:
           ingress:
             class: nginx
   ```

   Apply the `ClusterIssuer`:
   ```bash
   kubectl apply -f cluster-issuer.yml
   ```

3. **Set up Ingress Resource with SSL**:

   Create an `ingress.yml` for your app:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: mern-ingress
     annotations:
       cert-manager.io/cluster-issuer: "letsencrypt-prod"
   spec:
     tls:
     - hosts:
       - your-domain.com
       secretName: mern-tls
     rules:
     - host: your-domain.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: frontend
               port:
                 number: 80
   ```

   Apply the ingress resource:
   ```bash
   kubectl apply -f ingress.yml
   ```

---

### **Step 6: GitOps with ArgoCD for Deployment**

1. **Create a Git Repository for Kubernetes Manifests**:

   Create a repository that will store all your Kubernetes YAML files (for example, `frontend-deployment.yml`, `backend-deployment.yml`, `ingress.yml`, etc.).

2. **Connect GitHub Repository with ArgoCD**:

   From the ArgoCD UI, go to **Settings** → **Repositories** → **Connect Repo** and enter your GitHub repository details. Ensure the repository has access to the Kubernetes YAML files.

3. **Create an Application in ArgoCD**:

   In ArgoCD, go to **Applications** → **Create Application**:
   - **Repository URL**: URL of your Kubernetes manifests repository.
   - **Target Revision**: `main` (or any branch you want to deploy from).
   - **Path**: Path to the folder containing your Kubernetes manifests.
   - **Cluster**: Choose the cluster you want to deploy to.

4. **Sync Application**:

   After the application is created, ArgoCD will automatically sync and deploy your app. You can also manually sync the application when there are changes to your Git repository.

---

### **Step 7: Autoscaling and Auto-Healing**

1. **Horizontal Pod Autoscaler (HPA)**:

   Create a `hpa.yml` to scale the backend app based on CPU utilization:
   ```yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: backend-hpa
   spec:
     scaleTargetRef:
       apiVersion: apps/v1


       kind: Deployment
       name: backend
     minReplicas: 2
     maxReplicas: 10
     metrics:
     - type: Resource
       resource:
         name: cpu
         target:
           type: AverageValue
           averageValue: 200m
   ```

   Apply it:
   ```bash
   kubectl apply -f hpa.yml
   ```

2. **Pod Disruption Budgets**:

   Ensure your app remains highly available by setting a Pod Disruption Budget:
   ```yaml
   apiVersion: policy/v1
   kind: PodDisruptionBudget
   metadata:
     name: backend-pdb
   spec:
     minAvailable: 2
     selector:
       matchLabels:
         app: backend
   ```

   Apply it:
   ```bash
   kubectl apply -f pdb.yml
   ```

---

### **Step 8: Monitoring with Prometheus and Grafana**

You already installed **Prometheus** and **Grafana** earlier. Make sure to configure Prometheus to scrape metrics from your app:

1. **Configure Metrics** in your app:
   - Add Prometheus metrics in your **Node.js** backend and frontend (using `prom-client` for Node.js).

2. **Monitor Metrics** in Grafana:
   - In Grafana, import a Prometheus dashboard and configure it to visualize application metrics like CPU usage, memory usage, and request counts.

---

This setup ensures that your **MERN stack** application is highly available, scalable, secure, and monitored in both **development** and **production** environments. You can easily manage deployments via **ArgoCD** using **GitOps**, and monitor performance with **Prometheus** and **Grafana**.

Let me know if you need more details on any specific part of the setup!




---



To deploy your **MERN stack** application using **Helm** for Kubernetes, along with **ArgoCD**, **GitHub Actions**, **Prometheus**, **Grafana**, and **Let's Encrypt SSL**, we’ll organize the configuration into separate Helm charts and implement Helm-based automation for deployment. I'll walk you through the full setup using **Helm**.

### **Step 1: Prepare the Environment**

Before proceeding with the Helm-based deployment, ensure your **Kubernetes** environment is set up correctly as mentioned previously:

1. **Kubernetes Cluster** with `kubeadm` or any Kubernetes distribution (like `K3s`).
2. **Helm** installed on your system. If not installed, run:
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

### **Step 2: Set Up Helm Repositories**

First, add the necessary Helm repositories for **Prometheus**, **Grafana**, and **Cert-Manager**.

```bash
# Add the official Prometheus Helm chart
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Add the official Grafana Helm chart
helm repo add grafana https://grafana.github.io/helm-charts

# Add the official Cert-Manager Helm chart
helm repo add jetstack https://charts.jetstack.io

# Update the repositories
helm repo update
```

### **Step 3: Create a Helm Chart for the MERN Stack**

1. **Initialize Helm Chart** for both the frontend and backend services (MERN stack).

```bash
# Create a Helm chart for the backend (Node.js API)
helm create backend

# Create a Helm chart for the frontend (React)
helm create frontend
```

2. **Configure the Backend** chart:

   - Modify the `values.yaml` file for the backend chart to configure environment variables (e.g., database connections, port, etc.).
   - Update `deployment.yaml` to include relevant container specifications.

3. **Configure the Frontend** chart:

   - Modify the `values.yaml` file for the frontend to define the container image and environment variables.
   - Update `deployment.yaml` for frontend deployment specifics.

### **Step 4: Set Up ArgoCD with Helm Deployment**

1. **Install ArgoCD** in your Kubernetes cluster as described earlier (ensure it’s accessible and running).

2. **Set Up ArgoCD to Sync Helm Charts**:

   You’ll create a Git repository for your **Helm charts** and then add this repository to ArgoCD.

   - **Create a Git Repository** with Helm charts (frontend and backend).

   In the **ArgoCD UI**, add this Git repository under **Settings > Repositories**.

3. **Create ArgoCD Applications** for both Frontend and Backend.

   - **Application for Backend**:
      - **Repository URL**: Git URL of your Helm chart repository.
      - **Path**: The directory containing your backend chart (e.g., `backend/`).
      - **Cluster**: Choose your cluster.
      - **Namespace**: `default` or any namespace you created.

   - **Application for Frontend**:
      - **Repository URL**: Git URL of your Helm chart repository.
      - **Path**: The directory containing your frontend chart (e.g., `frontend/`).
      - **Cluster**: Choose your cluster.
      - **Namespace**: `default` or any namespace you created.

4. **Sync and Monitor the Application**:

   Once the application is created in ArgoCD, it will automatically sync the Helm charts and deploy your MERN stack. You can trigger manual syncs from the ArgoCD UI if necessary.

### **Step 5: Set Up Prometheus and Grafana with Helm**

1. **Install Prometheus using Helm**:

   Install the Prometheus stack, which includes both Prometheus and Alertmanager:
   ```bash
   helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring
   ```

2. **Install Grafana using Helm**:
   Grafana is automatically installed along with Prometheus if you’re using the kube-prometheus-stack chart. But if you want to install it separately, you can:
   ```bash
   helm install grafana grafana/grafana --namespace monitoring
   ```

3. **Access Prometheus and Grafana**:

   Expose **Prometheus** and **Grafana** through port forwarding or using a LoadBalancer.

   Example for Grafana:
   ```bash
   kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
   ```

   Access **Grafana** UI on `http://localhost:3000`, with the default username `admin` and password `prom-operator`.

4. **Configure Prometheus** to scrape metrics from your **backend** (Node.js) and **frontend** (React) applications. For that, you need to expose application metrics endpoints using a library like `prom-client` for Node.js.

### **Step 6: Set Up Let's Encrypt with Helm (Cert-Manager)**

1. **Install Cert-Manager using Helm**:

   Install Cert-Manager for automatic SSL certificate management:
   ```bash
   helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace
   ```

2. **Create ClusterIssuer for Let's Encrypt**:

   Define a `ClusterIssuer` for issuing SSL certificates from Let's Encrypt:

   Create a `cluster-issuer.yaml` file:
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt-prod
   spec:
     acme:
       email: your-email@example.com
       server: https://acme-v02.api.letsencrypt.org/directory
       privateKeySecretRef:
         name: letsencrypt-prod
       solvers:
       - http01:
           ingress:
             class: nginx
   ```

   Apply the `ClusterIssuer`:
   ```bash
   kubectl apply -f cluster-issuer.yaml
   ```

3. **Create Ingress Resources with SSL**:

   Define Ingress for both your frontend and backend services with SSL termination:

   Example `ingress.yaml`:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: mern-ingress
     annotations:
       cert-manager.io/cluster-issuer: "letsencrypt-prod"
   spec:
     tls:
     - hosts:
       - your-domain.com
       secretName: mern-tls
     rules:
     - host: your-domain.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: frontend
               port:
                 number: 80
   ```

   Apply the Ingress:
   ```bash
   kubectl apply -f ingress.yaml
   ```

4. **Test SSL Certificates**:
   After a few minutes, the SSL certificates from Let's Encrypt should be generated and applied to your domain.

### **Step 7: Set Up Auto-Scaling and Auto-Healing**

1. **Horizontal Pod Autoscaler (HPA)**:

   Add an HPA configuration in your `backend` and `frontend` charts to scale your pods automatically based on CPU utilization or memory usage.

   Example `hpa.yaml`:
   ```yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: backend-hpa
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: backend
     minReplicas: 2
     maxReplicas: 10
     metrics:
     - type: Resource
       resource:
         name: cpu
         target:
           type: AverageValue
           averageValue: 200m
   ```

   Apply the HPA:
   ```bash
   kubectl apply -f hpa.yaml
   ```

2. **Pod Disruption Budget (PDB)**:

   Define a **PodDisruptionBudget** to ensure that your services maintain availability during node maintenance or pod evictions:
   ```yaml
   apiVersion: policy/v1
   kind: PodDisruptionBudget
   metadata:
     name: backend-pdb
   spec:
     minAvailable: 2
     selector:
       matchLabels:
         app: backend
   ```

   Apply the PDB:
   ```bash
   kubectl apply -f pdb.yaml
   ```

### **Step 8: GitHub Actions for CI/CD with Helm**

1. **Create GitHub Actions Workflow** to build and push Docker images, then deploy with Helm:

   Example `.github/workflows/ci-cd.yml`:

   ```yaml
   name: Build and Deploy MERN Stack with Helm

   on:
     push:
       branches:
         - main

   jobs:
     build-and-deploy:
       runs-on: ubuntu-latest

       steps:
         - name: Checkout code
           uses: actions/checkout@v2

         - name: Set up Helm
           uses: Azure/setup-helm@v1

         - name: Set up kubectl
           uses: azure/setup-kubectl@v2
           with:
             kubeconfig: ${{ secrets.KUBECONFIG }}

         - name: Build Docker images
           run: |
             docker build -t your-docker-username/frontend ./frontend
             docker build -t your-docker-username/backend ./backend

         - name: Push Docker images
           run: |
             docker push your-docker-username/frontend
             docker push your-d

ocker-username/backend

         - name: Deploy with Helm
           run: |
             helm upgrade --install frontend ./frontend-chart
             helm upgrade --install backend ./backend-chart
   ```

   Push the workflow to GitHub, and it will automatically trigger on `main` branch pushes, building and deploying to your Kubernetes cluster.

---

With this setup, you have a complete **MERN stack** deployment using **Helm**, **ArgoCD**, **GitHub Actions**, **Prometheus**, **Grafana**, **Let's Encrypt**, and **Auto-Scaling**.

Let me know if you need more specific adjustments or clarifications!






