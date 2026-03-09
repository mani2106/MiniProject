# Jenkins Docker Container Troubleshooting

## Common Issues and Solutions

### Issue 1: "docker: command not found" in Jenkins

**Symptom**: Pipeline fails with `docker: command not found`

**Cause**: Jenkins container doesn't have Docker CLI installed

**Solution**: Use a Jenkins image that includes Docker CLI

```bash
# Stop and remove existing container
docker stop jenkins
docker rm jenkins

# Use image with Docker CLI pre-installed
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.kube:/root/.kube:ro" \
  nikirobot/jenkins-docker:lts
```

### Issue 2: "Cannot connect to Docker daemon"

**Symptom**: `Cannot connect to the Docker daemon at unix:///var/run/docker.sock`

**Cause**: Docker socket not mounted correctly

**Solution**: Verify socket mount and permissions

```bash
# Check if socket is mounted
docker exec jenkins ls -la /var/run/docker.sock

# Should show: srw-rw---- 1 root docker ... docker.sock

# If missing, restart with correct mount
docker stop jenkins
docker rm jenkins
docker run -d --name jenkins \
  -p 8080:8080 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.kube:/root/.kube:ro" \
  nikirobot/jenkins-docker:lts
```

### Issue 3: Windows Path Issues

**Symptom**: Mounts don't work on Windows

**Solution**: Use Git Bash, not PowerShell or CMD

```bash
# In Git Bash, use forward slashes
-v "$HOME/.kube:/root/.kube:ro"

# NOT this Windows style:
-v "%USERPROFILE%\.kube:/root/.kube:ro"
```

### Issue 4: Permission Denied on Docker Socket

**Symptom**: `permission denied while trying to connect to the Docker daemon socket`

**Solution**: Fix permissions inside container

```bash
# Get shell in container
docker exec -it jenkins bash

# Add jenkins user to docker group
usermod -aG docker jenkins

# Exit and restart container
exit
docker restart jenkins
```

### Issue 5: kubectl Not Found

**Symptom**: `kubectl: command not found`

**Cause**: kubectl not installed in Jenkins container

**Solution**: Install kubectl in container

```bash
docker exec -it jenkins bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
mv kubectl /usr/local/bin/
exit
```

Or use Jenkins with kubectl pre-installed:

```bash
docker run -d --name jenkins \
  -p 8080:8080 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.kube:/root/.kube:ro" \
  nikirobot/jenkins-docker:lts
```

### Issue 6: kubeconfig Not Mounted

**Symptom**: kubectl can't find cluster config

**Cause**: kubeconfig not mounted or wrong path

**Solution**: Verify mount and set KUBECONFIG env

```bash
# Check if kubeconfig exists
docker exec jenkins cat /root/.kube/config

# If empty, fix the mount
docker stop jenkins
docker rm jenkins
docker run -d --name jenkins \
  -p 8080:8080 \
  -v "$HOME/.kube:/root/.kube:ro" \
  -e KUBECONFIG=/root/.kube/config \
  nikirobot/jenkins-docker:lts
```

### Issue 7: minikube Not Accessible from Jenkins

**Symptom**: `kubectl get nodes` fails with "connection refused"

**Cause**: minikube not running or kubeconfig pointing to wrong cluster

**Solution**: Ensure minikube is running and use correct context

```bash
# On host machine
minikube status
minikube start  # if not running

# Check current context
kubectl config current-context

# If not minikube, switch to it
kubectl config use-context minikube

# Restart Jenkins to pick up updated kubeconfig
docker restart jenkins
```

## Quick Verification Commands

Run these to verify everything is working:

```bash
# 1. Check container is running
docker ps | grep jenkins

# 2. Check Docker CLI is available
docker exec jenkins docker --version

# 3. Check Docker socket access
docker exec jenkins docker ps

# 4. Check kubectl is available
docker exec jenkins kubectl version --client

# 5. Check kubeconfig is mounted
docker exec jenkins ls -la /root/.kube/config

# 6. Check cluster access
docker exec jenkins kubectl get nodes

# 7. Test full pipeline setup
docker exec jenkins sh -c "docker --version && kubectl get nodes"
```

## Platform-Specific Notes

### Mac (macOS)

```bash
# Everything should work with standard commands
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.kube:/root/.kube:ro" \
  -e KUBECONFIG=/root/.kube/config \
  nikirobot/jenkins-docker:lts
```

### Windows (Git Bash)

```bash
# Same as Mac, use Git Bash not PowerShell
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.kube:/root/.kube:ro" \
  -e KUBECONFIG=/root/.kube/config \
  nikirobot/jenkins-docker:lts
```

### Windows (PowerShell - Not Recommended)

```powershell
# PowerShell has different path syntax, use Git Bash instead
# If you must use PowerShell:
docker run -d --name jenkins `
  -p 8080:8080 -p 50000:50000 `
  -v jenkins_home:/var/jenkins_home `
  -v "/var/run/docker.sock:/var/run/docker.sock" `
  -v "$env:USERPROFILE\.kube:/root/.kube:ro" `
  -e KUBECONFIG=/root/.kube/config `
  nikirobot/jenkins-docker:lts
```

## Complete Working Setup

After fixing issues, your complete setup should be:

```bash
# 1. Start minikube
minikube start --driver=docker

# 2. Start Jenkins with all mounts
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.kube:/root/.kube:ro" \
  -e KUBECONFIG=/root/.kube/config \
  nikirobot/jenkins-docker:lts

# 3. Get initial password
docker logs jenkins 2>&1 | grep -A 5 "initialAdminPassword"

# 4. Open Jenkins
open http://localhost:8080  # Mac
# or open browser to http://localhost:8080  # Windows

# 5. Verify everything works
docker exec jenkins docker --version
docker exec jenkins kubectl get nodes
```

## Still Having Issues?

Enable debug logging in Jenkins:

```bash
# Enable Docker debug
docker exec jenkins sh -c 'export DOCKER_BUILDKIT=0; docker --debug version'

# Enable kubectl debug
docker exec jenkins kubectl get nodes --v=9

# Check Jenkins logs
docker logs jenkins -f
```
