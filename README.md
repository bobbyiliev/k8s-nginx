# k8s-nginx

A simple Kubernetes nginx deployment with automated CI/CD using GitHub Actions, featuring horizontal pod autoscaling and ingress configuration.

## Prerequisites

- Linux server (Ubuntu 20.04+ recommended)
- GitHub repository access
- Docker Hub account

## Setup Instructions

### 1. Linux Server Setup

#### Initial Server Configuration
```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y curl wget git vim htop net-tools

# Create a non-root user (if not already exists)
sudo useradd -m -s /bin/bash k8s-user
sudo usermod -aG sudo k8s-user
```

### 2. Install Docker

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add current user to docker group
sudo usermod -aG docker $USER

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Verify installation
docker --version
```

### 3. Install kubectl

```bash
# Download kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version --client
```

### 4. Install Minikube

```bash
# Download minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Install minikube
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify installation
minikube version
```

### 5. Install socat (for port forwarding)

```bash
# Install socat
sudo apt install -y socat

# Verify installation
socat -V
```

### 6. Start Minikube

```bash
# Start minikube with docker driver
minikube start --driver=docker

# Enable ingress addon
minikube addons enable ingress

# Verify cluster status
kubectl get nodes
minikube status
```

### 7. GitHub Runner Setup

#### Overview
You'll need to set up a self-hosted GitHub runner on your server to enable the CI/CD pipeline.

#### Steps:
1. **Navigate to your GitHub repository**
2. **Go to Settings → Actions → Runners**
3. **Click "New self-hosted runner"**
4. **Select Linux x64**
5. **Follow the provided commands** to download and configure the runner

#### Basic Runner Setup Commands:
```bash
# Create a folder for the runner
mkdir actions-runner && cd actions-runner

# Download the latest runner package (replace URL with the one from GitHub)
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz

# Extract the installer
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Configure the runner (use token from GitHub)
./config.sh --url https://github.com/YOUR-USERNAME/YOUR-REPO --token YOUR-TOKEN

# Install and start the service
sudo ./svc.sh install
sudo ./svc.sh start
```

### 8. Configure GitHub Secrets

Add the following secrets to your GitHub repository (Settings → Secrets and variables → Actions):

- `DOCKER_USERNAME`: Your Docker Hub username
- `DOCKER_PASSWORD`: Your Docker Hub password/token

### 9. Network Access Configuration

The GitHub Actions workflow automatically sets up external access using socat on port 8080.

#### Manual Setup (if needed)
```bash
# Create a proxy to access the NodePort service externally
socat TCP-LISTEN:8080,fork TCP:$(minikube ip):30080 &

# To make it persistent
nohup socat TCP-LISTEN:8080,fork TCP:$(minikube ip):30080 > /tmp/socat.log 2>&1 &
disown
```

**Note:** We use port 8080 to avoid needing sudo permissions. The GitHub Actions workflow handles this automatically.

### 10. Deployment

#### Automated Deployment (Recommended)
The GitHub Actions workflow automatically:
- Builds and pushes the Docker image to your Docker Hub
- Updates the deployment.yaml with your image reference  
- Deploys to Kubernetes
- Sets up external access on port 8080

Just push to the main branch and everything happens automatically!

#### Manual Deployment
```bash
# Clone the repository
git clone https://github.com/YOUR-USERNAME/k8s-nginx.git
cd k8s-nginx

# Build and push Docker image (replace with your Docker Hub username)
docker build -t YOUR-DOCKERHUB-USERNAME/nginx-hello:latest .
docker push YOUR-DOCKERHUB-USERNAME/nginx-hello:latest

# Update deployment.yaml to use your image
sed -i "s|bobbyiliev/nginx-hello:latest|YOUR-DOCKERHUB-USERNAME/nginx-hello:latest|" k8s/deployment.yaml

# Deploy to Kubernetes
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
kubectl apply -f k8s/hpa.yaml

# Verify deployment
kubectl get pods
kubectl get services
kubectl get ingress
kubectl get hpa
```

### 11. Access Your Application

#### Via Automated Proxy (GitHub Actions)
The workflow automatically sets up a proxy on port 8080:
```bash
# Access via server IP
curl http://YOUR-SERVER-IP:8080
```

#### Via NodePort (manual access)
```bash
# Access via minikube IP directly
curl http://$(minikube ip):30080
```

#### Via Ingress (local testing)
```bash
# Add to /etc/hosts if testing locally
echo "$(minikube ip) nginx.local" | sudo tee -a /etc/hosts

# Access via ingress
curl http://nginx.local
```

## Monitoring and Troubleshooting

### Check Pod Status
```bash
kubectl get pods -w
kubectl describe pod POD-NAME
kubectl logs POD-NAME
```

### Check Service
```bash
kubectl get svc
kubectl describe svc nginx-service
```

### Check HPA
```bash
kubectl get hpa
kubectl describe hpa nginx-hpa
```

### Check Ingress
```bash
kubectl get ingress
kubectl describe ingress nginx-ingress
```

### Socat Proxy Troubleshooting
```bash
# Check if socat is running
ps aux | grep socat

# Check if port 8080 is listening
netstat -tlnp | grep 8080

# Kill socat processes
pkill -f "TCP-LISTEN:8080"

# Restart socat manually
nohup socat TCP-LISTEN:8080,fork TCP:$(minikube ip):30080 > /tmp/socat.log 2>&1 &
disown

# Check socat logs
tail -f /tmp/socat.log
```

### Minikube Troubleshooting
```bash
# Restart minikube
minikube stop
minikube start --driver=docker

# Check minikube status
minikube status

# Access minikube dashboard
minikube dashboard
```

## Cleanup

```bash
# Delete Kubernetes resources
kubectl delete -f k8s/

# Stop minikube
minikube stop

# Delete minikube cluster
minikube delete
```

## Notes

- The application will be available on port 8080 (automatically configured by GitHub Actions)
- Direct access to minikube NodePort is available on port 30080
- HPA will scale pods between 2-10 replicas based on CPU usage (50% threshold)
- The GitHub Actions workflow automatically updates the deployment image reference
- Socat proxy persists after deployment and survives server reboots
- Make sure to replace placeholder values (YOUR-USERNAME, YOUR-REPO, etc.) with actual values
- For production deployments, consider using LoadBalancer or proper Ingress with TLS
