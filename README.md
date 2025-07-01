# k8s-nginx

A simple Kubernetes nginx deployment with automated CI/CD using GitHub Actions.

## Prerequisites

- Linux server (Ubuntu 20.04+ recommended)
- GitHub repository access
- Docker Hub account

## Quick Setup

### 1. Install Dependencies
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y curl wget git socat

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### 2. Start Minikube
```bash
minikube start --driver=docker
minikube addons enable ingress # Or install it with helm
```

### 3. Setup GitHub Runner
1. Go to your GitHub repo → Settings → Actions → Runners
2. Click "New self-hosted runner" → Linux x64
3. Follow the setup commands provided by GitHub

### 4. Configure GitHub Secrets
Add these secrets to your repository (Settings → Secrets and variables → Actions):
- `DOCKER_USERNAME`: Your Docker Hub username
- `DOCKER_PASSWORD`: Your Docker Hub password/token

### 5. Deploy
Just push to the main branch! The CI will automatically:
- Build and push your Docker image
- Deploy to Kubernetes

### 6. Access Your App
After CI completes, expose the app with one simple command:

```bash
socat TCP-LISTEN:8081,fork,reuseaddr,bind=0.0.0.0 TCP:$(minikube ip):30080 &
```

Now access your app at: `http://YOUR-SERVER-IP:8081`

## Alternative Access Methods

If running locally, you can just access it via the Minikube IP:

```bash
curl http://$(minikube ip):30080
```

**Check what's running:**
```bash
kubectl get pods
kubectl get services
minikube ip
```

## Cleanup
```bash
# Stop port forwarding
sudo pkill -f "8081"

# Delete everything
kubectl delete -f k8s/
minikube delete
```

That's it! 🎉
