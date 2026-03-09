# CI/CD Pipeline: Jenkins → Docker → Kubernetes

Deploy a Python Flask app from GitHub through Jenkins to Kubernetes (minikube).

**Your Setup:**
- **DockerHub**: [madhumitha3199/flask-app](https://hub.docker.com/r/madhumitha3199/flask-app)
- **GitHub**: [RPMadhumitha/MiniProject](https://github.com/RPMadhumitha/MiniProject)

## Architecture (All Local!)

```
┌─────────────────────────────────────────┐
│  Your Machine (Mac/Windows)              │
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

**Everything runs locally!** Only external traffic: pushing/pulling from DockerHub.

## Prerequisites

- Docker Desktop installed and running
- minikube for local Kubernetes
- DockerHub account (you have: `madhumitha3199`)
- GitHub account (you have: `RPMadhumitha`)

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

### 2. Start Jenkins (with Docker CLI)

```bash
# Jenkins image with Docker CLI pre-installed
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.kube:/root/.kube:ro" \
  -e KUBECONFIG=/root/.kube/config \
  nikirobot/jenkins-docker:lts
```

Get initial password:
```bash
docker logs jenkins 2>&1 | grep -A 5 "initialAdminPassword"
```

Access Jenkins: http://localhost:8080

### 3. Configure Jenkins

1. **Install Plugins** (via Manage Jenkins → Plugins):
   - Docker Pipeline
   - Kubernetes CLI

2. **Create Credentials** (via Manage Jenkins → Credentials):

   **a. DockerHub Credentials:**
   - **ID**: `dockerhub-creds`
   - **Username**: `madhumitha3199`
   - **Password**: Your DockerHub access token (create at hub.docker.com → Account Settings → Security)

   **b. GitHub Credentials** (for private repo or if using PAT):
   - **ID**: `github-creds`
   - **Username**: `RPMadhumitha`
   - **Password**: Your GitHub personal access token (create at github.com → Settings → Developer settings → Personal access tokens)

3. **Create Pipeline Job**:
   - New Item → Pipeline
   - Pipeline → Definition: Pipeline script from SCM
   - SCM: Git
   - Repository URL: `https://github.com/RPMadhumitha/MiniProject.git`
   - Credentials: `github-creds` (if using)

### 4. Configuration Already Done! ✓

Your files are already configured with your details:
- ✓ **deployment.yaml**: Uses `madhumitha3199/flask-app:latest`
- ✓ **Jenkinsfile**: Uses your DockerHub username and GitHub repo
- ✓ **Credentials**: Uses `dockerhub-creds` and `github-creds`

### 5. Push to GitHub (if not done)

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/RPMadhumitha/MiniProject.git
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

**Pipeline Stages:**
1. **Verify Setup** - Checks Docker and kubectl are accessible
2. **Clone** - Pulls code from GitHub
3. **Build** - Builds Docker image
4. **Test Locally** - Tests container before pushing
5. **Push** - Pushes to DockerHub (tagged with build number and `latest`)
6. **Deploy** - Deploys to Kubernetes
7. **Verify Deployment** - Checks pods and tests health endpoint

**Automatic Rollback:** If any stage fails, the pipeline automatically rolls back to the previous deployment.

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

## Testing the Deployed Kubernetes App

### Method 1: Using minikube service (Easiest)

```bash
# Opens the app in your default browser
minikube service flask-service

# Or get the URL without opening browser
minikube service flask-service --url
```

**Test the endpoints:**
```bash
# Get the service URL
SERVICE_URL=$(minikube service flask-service --url)

# Test root endpoint
curl $SERVICE_URL/
# Expected: "Hello from CI/CD!"

# Test health endpoint
curl $SERVICE_URL/health
# Expected: {"status": "healthy"}
```

### Method 2: Using kubectl port-forward

```bash
# Forward service to localhost
kubectl port-forward svc/flask-service 8080:80

# Test in another terminal
curl http://localhost:8080/
curl http://localhost:8080/health
```

### Method 3: Accessing via NodePort

```bash
# Get the NodePort
kubectl get svc flask-service

# Access via minikube IP
minikube ip
# Then use: http://<minikube-ip>:<nodeport>/
```

### Method 4: From inside the cluster

```bash
# Run a temporary pod to test from inside
kubectl run temp-curl --rm -i --restart=Never --image=curlimages/curl \
  --command -- curl -s http://flask-service/

# Test health endpoint
kubectl run temp-curl --rm -i --restart=Never --image=curlimages/curl \
  --command -- curl -s http://flask-service/health
```

### Comprehensive Test Script

```bash
#!/bin/bash
echo "=== Testing Flask App Deployment ==="
echo ""

# 1. Check pods
echo "1. Checking pods..."
kubectl get pods -l app=flask
echo ""

# 2. Check service
echo "2. Checking service..."
kubectl get svc flask-service
echo ""

# 3. Get service URL
echo "3. Getting service URL..."
SERVICE_URL=$(minikube service flask-service --url)
echo "Service URL: $SERVICE_URL"
echo ""

# 4. Test root endpoint
echo "4. Testing root endpoint..."
curl -s $SERVICE_URL/
echo ""
echo ""

# 5. Test health endpoint
echo "5. Testing health endpoint..."
curl -s $SERVICE_URL/health | jq .
echo ""

# 6. Test multiple times (check load balancing)
echo "6. Testing multiple requests (should hit different pods)..."
for i in {1..5}; do
  echo "Request $i:"
  curl -s $SERVICE_URL/health | jq .
  sleep 1
done

echo ""
echo "=== All tests completed! ==="
```

### Browser Testing

1. **Get the URL:**
   ```bash
   minikube service flask-service --url
   ```

2. **Open in browser:**
   - Navigate to: `http://<URL-from-above>/`
   - Should see: "Hello from CI/CD!"
   - Navigate to: `http://<URL-from-above>/health`
   - Should see: `{"status": "healthy"}`

### Checking Pod Logs

```bash
# Get pod names
kubectl get pods -l app=flask

# View logs from a specific pod
kubectl logs <pod-name>

# Follow logs in real-time
kubectl logs -f <pod-name>

# View logs from all pods
kubectl logs -l app=flask --all-containers=true
```

### Checking Deployment Status

```bash
# Check deployment rollout status
kubectl rollout status deployment/flask-app

# Check deployment history
kubectl rollout history deployment/flask-app

# View detailed deployment info
kubectl describe deployment flask-app

# View detailed pod info
kubectl describe pod <pod-name>
```

### Troubleshooting Deployment Issues

**Pods not running?**
```bash
# Check pod status
kubectl get pods -l app=flask

# Describe pod to see events
kubectl describe pod <pod-name>

# Check pod logs
kubectl logs <pod-name>

# Common issues:
# - ImagePullBackOff: Check image name in deployment.yaml
# - CrashLoopBackOff: Check app logs for errors
# - Pending: Check resource availability
```

**Can't access service?**
```bash
# Check service exists
kubectl get svc flask-service

# Check service endpoints
kubectl get endpoints flask-service

# Should show pod IPs, if empty:
# - Check pod labels match service selector
# - Check pods are running

# Test service from inside cluster
kubectl run test-pod --rm -i --restart=Never --image=curlimages/curl \
  --command -- curl -v http://flask-service/
```

**Port-forward not working?**
```bash
# Make sure service name is correct
kubectl get svc

# Check if port-forward is already running
# (only one port-forward per service:port)

# Use correct syntax
kubectl port-forward svc/flask-service 8080:80
```

### Load Balancing Test

Verify both replicas are receiving traffic:

```bash
# Terminal 1: Watch pod logs
kubectl logs -f -l app=flask --all-containers=true

# Terminal 2: Send multiple requests
for i in {1..10}; do
  curl http://localhost:8080/health
  sleep 1
done
```

You should see requests distributed across both pods.

### Performance Testing

```bash
# Get service URL
SERVICE_URL=$(minikube service flask-service --url)

# Simple load test (100 requests)
echo "Testing with 100 requests..."
time for i in {1..100}; do
  curl -s $SERVICE_URL/health > /dev/null
done

# With concurrency
echo "Testing with 10 concurrent connections..."
for i in {1..10}; do
  curl -s $SERVICE_URL/health &
done
wait
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

## Quick Reference (Your Setup)

**DockerHub Image:** `madhumitha3199/flask-app`
**GitHub Repo:** `https://github.com/RPMadhumitha/MiniProject`
**Jenkins URL:** `http://localhost:8080`
**App URL:** `minikube service flask-service`

**Useful Commands:**
```bash
# Check Jenkins container
docker ps | grep jenkins

# Check Jenkins logs
docker logs jenkins -f

# Restart Jenkins
docker restart jenkins

# Check deployment
kubectl get pods -l app=flask
kubectl get svc flask-service

# Access the app (opens in browser)
minikube service flask-service

# Get service URL for testing
SERVICE_URL=$(minikube service flask-service --url)
curl $SERVICE_URL/
curl $SERVICE_URL/health

# Check pod logs
kubectl logs -l app=flask --all-containers=true

# Check deployment history
kubectl rollout history deployment/flask-app

# Manual rollback
kubectl rollout undo deployment/flask-app

# Port-forward to localhost
kubectl port-forward svc/flask-service 8080:80
# Then test: curl http://localhost:8080/
```