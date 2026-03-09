# Simple CI/CD Pipeline: Jenkins → Docker → Kubernetes

A minimal CI/CD pipeline for learning. Deploy a simple Python Flask app from GitHub through Jenkins to Kubernetes.

## Architecture (All Local!)

```
┌─────────────────────────────────────────┐
│  Your Windows Machine                    │
│                                          │
│  ┌────────────────┐  ┌────────────────┐ │
│  │  Jenkins       │  │  minikube       │ │
│  │  (Container)   │  │  (Container)    │ │
│  │  Port: 8080    │──┼─> K8s Cluster   │ │
│  └────────────────┘  └────────────────┘ │
│         │                    │           │
│         └────────────────────┘           │
│           (Docker Daemon)                │
└─────────────────────────────────────────┘
```

## Prerequisites

- **Docker Desktop** installed and running
- **minikube** for local Kubernetes
- **DockerHub account** (free at hub.docker.com)
- **GitHub account**

## Setup Instructions

### 1. Install and Start minikube

```bash
# Download from https://minikube.sigs.k8s.io/
minikube start --driver=docker
minikube status
```

Verify:
```bash
kubectl get nodes
```

### 2. Start Jenkins

```bash
docker run -d -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v //var/run/docker.sock:/var/run/docker.sock \
  -v ~/.kube:/root/.kube \
  -e KUBECONFIG=/root/.kube/config \
  jenkins/jenkins:lts
```

Get initial password:
```bash
docker exec -it <jenkins-container-id> cat /var/jenkins_home/secrets/initialAdminPassword
```

Access Jenkins: http://localhost:8080

### 3. Configure Jenkins

1. **Install Plugins** (via Manage Jenkins → Plugins):
   - Docker Pipeline
   - Kubernetes CLI

2. **Create Credentials** (via Manage Jenkins → Credentials):
   - **ID**: `dockerhub-creds`
   - **Username**: Your DockerHub username
   - **Password**: Your DockerHub access token (create at hub.docker.com → Account Settings → Security)

3. **Create Pipeline Job**:
   - New Item → Pipeline
   - Pipeline → Definition: Pipeline script from SCM
   - SCM: Git
   - Repository URL: Your GitHub repo URL

### 4. Update Configuration Files

**IMPORTANT**: Replace placeholders in these files:

1. **deployment.yaml**: Replace `YOUR_DOCKERHUB_USER` with your DockerHub username
2. **Jenkinsfile**: Replace `your-dockerhub-user` with your DockerHub username
3. **Jenkinsfile**: Replace `https://github.com/youruser/testjenkins.git` with your repo URL

### 5. Create GitHub Repository

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/yourusername/testjenkins.git
git push -u origin main
```

## Testing

### Test 1: Flask App Locally
```bash
pip install -r requirements.txt
python app.py
# Visit http://localhost:5000
```

### Test 2: Docker Image
```bash
docker build -t flask-app:test .
docker run -p 5000:5000 flask-app:test
# Visit http://localhost:5000
```

### Test 3: Kubernetes (Manual)
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get pods
minikube service flask-service
```

### Test 4: Jenkins Pipeline
1. Go to http://localhost:8080
2. Click "Build Now" on your job
3. Watch the console output
4. Verify each stage completes

## Verification

After pipeline completes:
```bash
# Check pods
kubectl get pods

# Check service
kubectl get svc

# Access the app
minikube service flask-service
```

## Rollback Test

To test automatic rollback:
1. Break the app (e.g., change port in app.py)
2. Push to GitHub
3. Trigger Jenkins build
4. Watch it fail and automatically rollback

## What Happens

1. **You** push code to GitHub
2. **Jenkins** pulls from GitHub
3. **Jenkins** builds Docker image
4. **Jenkins** pushes to DockerHub
5. **Jenkins** deploys to minikube via kubectl
6. **minikube** pulls image from DockerHub
7. **You** access app via `minikube service`

## Troubleshooting

**minikube not starting?**
```bash
minikube delete
minikube start --driver=docker
```

**Jenkins can't access kubectl?**
```bash
# Verify kubeconfig is mounted
docker exec -it <jenkins-container-id> kubectl get nodes
```

**Pods not starting?**
```bash
kubectl describe pods
kubectl logs <pod-name>
```

**Image pull errors?**
- Make sure deployment.yaml has correct DockerHub username
- Make sure image is public on DockerHub

## Files

- `app.py` - Flask application
- `requirements.txt` - Python dependencies
- `Dockerfile` - Container definition
- `deployment.yaml` - Kubernetes deployment
- `service.yaml` - Kubernetes service
- `Jenkinsfile` - CI/CD pipeline
