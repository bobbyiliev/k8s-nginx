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

### 5. Start Minikube

```bash
# Start minikube with docker driver
minikube start --driver=docker

# Enable ingress addon
minikube addons enable ingress

# Verify cluster status
kubectl get nodes
minikube status
```

### 6. GitHub Runner Setup

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

### 7. Configure GitHub Secrets

Add the following secrets to your GitHub repository (Settings → Secrets and variables → Actions):

- `DOCKER_USERNAME`: Your Docker Hub username
- `DOCKER_PASSWORD`: Your Docker Hub password/token

### 8. Network Access Configuration

The GitHub Actions workflow automatically sets up port forwarding using iptables rules.

#### Manual Setup (if needed)
```bash
# Get minikube IP
MINIKUBE_IP=$(minikube ip)

# Create iptables forwarding rule (port 8080 -> minikube:30080)
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination ${MINIKUBE_IP}:30080

# Enable IP forwarding
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

### 9. Deployment

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

### 10. Access Your Application

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

### iptables Troubleshooting
```bash
# Check current iptables NAT rules
sudo iptables -t nat -L PREROUTING -n

# Remove the forwarding rule
MINIKUBE_IP=$(minikube ip)
sudo iptables -t nat -D PREROUTING -p tcp --dport 8080 -j DNAT --to-destination ${MINIKUBE_IP}:30080

# Re-add the forwarding rule
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination ${MINIKUBE_IP}:30080

# Check if IP forwarding is enabled
cat /proc/sys/net/ipv4/ip_forward

# Enable IP forwarding
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
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

- The application will be available on port 8080 (automatically configured by GitHub Actions using iptables)
- Direct access to minikube NodePort is available on port 30080
- HPA will scale pods between 2-10 replicas based on CPU usage (50% threshold)
- The GitHub Actions workflow automatically updates the deployment image reference
- iptables forwarding rules persist without requiring any running processes (more reliable than socat)
- Make sure to replace placeholder values (YOUR-USERNAME, YOUR-REPO, etc.) with actual values
- For production deployments, consider using LoadBalancer or proper Ingress with TLS
