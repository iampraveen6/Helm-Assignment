# Learner Report System Helm Deployment

This repository contains Helm charts for deploying a MERN stack application consisting of a React frontend, Node.js backend, and MongoDB database.

## Prerequisites

1. Install Kubernetes CLI (kubectl):
```bash
# For Windows (using Chocolatey)
choco install kubernetes-cli

# For MacOS
brew install kubernetes-cli

# For Ubuntu
sudo snap install kubectl --classic
```

2. Install Helm:
```bash
# For Windows (using Chocolatey)
choco install kubernetes-helm

# For MacOS
brew install helm

# For Ubuntu
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /usr/share/keyrings/helm-stable-debian.gpg
sudo apt-get update
sudo apt-get install helm
```

## Project Setup

1. Clone the repositories:
```bash
# Clone frontend repository
git clone https://github.com/UnpredictablePrashant/learnerReportCS_frontend.git

# Clone backend repository
git clone https://github.com/UnpredictablePrashant/learnerReportCS_backend.git
```

2. Build Docker images:
```bash
# Build frontend image
cd learnerReportCS_frontend
docker build -t iampraveen6/learner-report-frontend:latest .

# Build backend image
cd ../learnerReportCS_backend
docker build -t iampraveen6/learner-report-backend:latest .
```

## Helm Chart Structure

```
learner-report-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── backend-deployment.yaml
    ├── backend-service.yaml
    ├── frontend-deployment.yaml
    ├── frontend-service.yaml
    ├── mongo-deployment.yaml
    ├── mongo-service.yaml
    ├── mongo-pvc.yaml
    └── mongo-secret.yaml
```

## Configuration

The deployment can be customized through the `values.yaml` file:

```yaml
mongodb:
  uri: mongodb+srv://your-connection-string

backend:
  image: iampraveen6/learner-report-backend
  tag: latest
  replicas: 2
  port: 5000

frontend:
  image: iampraveen6/learner-report-frontend
  tag: latest
  replicas: 2
  port: 80
  serviceType: LoadBalancer
```

## Deployment

1. Create MongoDB secret:
```bash
kubectl create secret generic mongodb-secret \
  --from-literal=username=your-username \
  --from-literal=password=your-password \
  --from-literal=uri=your-mongodb-uri
```

2. Install the Helm chart:
```bash
# Add the chart repository
helm repo add learner-report https://your-chart-repo-url

# Install the chart
helm install learner-report ./learner-report-chart
```

3. Verify the deployment:
```bash
# Check pod status
kubectl get pods

# Check services
kubectl get services
```

## Access the Application

Once deployed, you can access the application through:

- Frontend: http://<load-balancer-ip>
- Backend API: http://<load-balancer-ip>:5000

To get the LoadBalancer IP:
```bash
kubectl get svc frontend -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

## Cleanup

To remove the deployment:
```bash
helm uninstall learner-report
kubectl delete pvc mongodb-pv-claim
kubectl delete secret mongodb-secret
```

## Architecture

The application consists of three main components:

1. Frontend (React.js):
   - Serves the user interface
   - Exposed via LoadBalancer service
   - Configured with environment variables for backend connectivity

2. Backend (Node.js):
   - Handles API requests
   - Connects to MongoDB
   - Internal service accessible only within the cluster

3. MongoDB:
   - Persistent data storage
   - Uses PersistentVolumeClaim for data persistence
   - Protected by Kubernetes Secrets

## Troubleshooting

1. Check pod logs:
```bash
kubectl logs <pod-name>
```

2. Check pod status:
```bash
kubectl describe pod <pod-name>
```

3. Common issues:
   - MongoDB connection failures: Check secret configuration
   - Frontend cannot reach backend: Verify service names and ports
   - Persistent volume issues: Check storage class availability

## Contributing

Please submit issues and pull requests to improve the Helm charts.