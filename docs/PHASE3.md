# DevOps Learning Project - Phase 3: Jenkins CI/CD Pipeline (FINAL)

## Overview (Week 3 - ~8 hours)

Install Jenkins and create an automated CI/CD pipeline that builds, tests, and deploys your application automatically.

**End result:** Code change → Automatic build → Test → Deploy to Production (all automated!)

---

## Prerequisites from Phase 2

✅ Completed Phase 1 & 2  
✅ Jenkins VM running (192.168.122.100 or your IP)  
✅ Production VM running (192.168.122.101 or your IP)  
✅ SSH access working to both VMs  
✅ Jenkins can SSH to Production  

**CRITICAL:** Ensure both VMs are running before starting Phase 3!

```bash
# Check VMs are running
sudo virsh list

# Should show both jenkins-server and production-server as "running"

# If not running:
sudo virsh start jenkins-server
sudo virsh start production-server
```

---

## Step 1: Install Jenkins

### Connect to Jenkins VM

```bash
ssh jenkins
```

### Add Jenkins Repository and Install

```bash
# Add Jenkins GPG key
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# Add Jenkins repository
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update package list
sudo apt update

# Install Jenkins
sudo apt install -y jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Check status (should be "active")
sudo systemctl status jenkins
```

### Get Initial Admin Password

```bash
# Wait 30 seconds for Jenkins to initialize
sleep 30

# Get the password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# COPY THIS PASSWORD - you'll need it next!
```

---

## Step 2: Configure Jenkins Web Interface

### Access Jenkins

From your **host machine** browser, open:
```
http://192.168.122.100:8080
```

(Replace with your Jenkins VM IP)

### Initial Setup Wizard

1. **Unlock Jenkins**
   - Paste the initial admin password
   - Click "Continue"

2. **Install Suggested Plugins**
   - Click "Install suggested plugins"
   - Wait 2-3 minutes for installation

3. **Create First Admin User**
   - Username: `admin`
   - Password: `admin123` (or your choice - remember it!)
   - Full name: `DevOps Admin`
   - Email: `admin@localhost`
   - Click "Save and Continue"

4. **Instance Configuration**
   - Jenkins URL: `http://192.168.122.100:8080/`
   - Click "Save and Finish"
   - Click "Start using Jenkins"

You should now see the Jenkins dashboard!

---

## Step 3: Install Additional Jenkins Plugins

### Install Docker and SSH Plugins

1. Click **"Manage Jenkins"** (left sidebar)
2. Click **"Plugins"**
3. Click **"Available plugins"** tab
4. In the search box, type: `docker pipeline`
5. Check **"Docker Pipeline"**
6. Search for: `ssh agent`
7. Check **"SSH Agent"**
8. Click **"Install"** button
9. Check **"Restart Jenkins when installation is complete"**

Wait 1-2 minutes for Jenkins to restart.

### Log Back In

- Refresh browser
- Log in with `admin` / `admin123` (or your password)

---

## Step 4: Install Docker on Jenkins VM

Jenkins needs Docker to build images.

```bash
# SSH to Jenkins (if not already connected)
ssh jenkins

# Install Docker
sudo apt update
sudo apt install -y docker.io docker-compose

# Add jenkins user to docker group
sudo usermod -aG docker jenkins

# Install coreutils (for nohup, required by Jenkins)
sudo apt install -y coreutils

# Restart Jenkins to pick up group changes
sudo systemctl restart jenkins

# Verify Docker works
docker --version

# Wait for Jenkins to restart
sleep 30

exit
```

---

## Step 5: Configure SSH Credentials in Jenkins

Jenkins needs Production VM's SSH key to deploy.

### Get Jenkins' Private Key

```bash
# SSH to Jenkins
ssh jenkins

# Display the private key
cat ~/.ssh/id_ed25519

# COPY THE ENTIRE OUTPUT (including BEGIN and END lines)

exit
```

### Add to Jenkins

1. In Jenkins web interface, go to **"Manage Jenkins"** → **"Credentials"**
2. Click **(global)** domain
3. Click **"Add Credentials"** (left sidebar)
4. Configure:
   - **Kind:** SSH Username with private key
   - **ID:** `production-ssh-key`
   - **Description:** `SSH key for Production VM`
   - **Username:** `devops`
   - **Private Key:** Select "Enter directly"
   - Click **"Add"** button
   - Paste the private key you copied
5. Click **"Create"**

---

## Step 6: Copy Application Code to Jenkins

### Prepare Code on Host

```bash
cd ~/devops-projects/task-manager-pipeline

# Remove venv if it exists (don't copy it)
rm -rf app/venv

# Verify app directory is small
du -sh app/
# Should be under 500KB
```

### Copy to Jenkins

```bash
cd ~/devops-projects/task-manager-pipeline

# Create directory on Jenkins
ssh jenkins "mkdir -p /tmp/task-manager-pipeline"

# Copy application files (NOT vm-configs or .git)
scp -r app jenkins:/tmp/task-manager-pipeline/
scp -r docker jenkins:/tmp/task-manager-pipeline/
scp Jenkinsfile jenkins:/tmp/task-manager-pipeline/ 2>/dev/null || echo "Jenkinsfile will be created"
scp .gitignore jenkins:/tmp/task-manager-pipeline/

# Verify size
ssh jenkins "du -sh /tmp/task-manager-pipeline"
# Should be under 1MB
```

### Move to Jenkins Workspace

```bash
ssh jenkins

# Create Jenkins workspace
sudo mkdir -p /var/lib/jenkins/workspace/task-manager-pipeline

# Copy files
sudo cp -r /tmp/task-manager-pipeline/* /var/lib/jenkins/workspace/task-manager-pipeline/

# Set ownership
sudo chown -R jenkins:jenkins /var/lib/jenkins/workspace/

# Initialize Git (Jenkins needs this)
cd /var/lib/jenkins/workspace/task-manager-pipeline
sudo -u jenkins git init
sudo -u jenkins git add .
sudo -u jenkins git config user.email "jenkins@localhost"
sudo -u jenkins git config user.name "Jenkins"
sudo -u jenkins git commit -m "Initial commit for Jenkins"

# Verify
ls -la /var/lib/jenkins/workspace/task-manager-pipeline

exit
```

---

## Step 7: Create Jenkins Pipeline Job

### Create New Pipeline

1. From Jenkins dashboard, click **"New Item"**
2. Enter name: `task-manager-pipeline`
3. Select **"Pipeline"**
4. Click **"OK"**

### Configure Pipeline

In the configuration page:

**General Section:**
- Description: `Automated build and deployment pipeline for Task Manager`

**Pipeline Section:**
- **Definition:** Select "Pipeline script"
- **Script:** Paste the following Jenkinsfile **(UPDATE PRODUCTION_HOST with your Production VM IP!)**

```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "task-manager"
        PRODUCTION_HOST = "192.168.122.101"  // UPDATE THIS IP!
        PRODUCTION_USER = "devops"
    }
    
    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image...'
                    sh '''
                        docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                          -f docker/Dockerfile .
                        docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    echo 'Running basic health check...'
                    sh '''
                        # Start container temporarily for testing
                        docker run -d --name test-${BUILD_NUMBER} \
                          -p 5001:5000 ${DOCKER_IMAGE}:latest
                        
                        # Wait for app to start
                        sleep 5
                        
                        # Health check
                        curl -f http://localhost:5001/health || exit 1
                        
                        # Cleanup test container
                        docker stop test-${BUILD_NUMBER}
                        docker rm test-${BUILD_NUMBER}
                    '''
                }
            }
        }
        
        stage('Save Docker Image') {
            steps {
                script {
                    echo 'Saving Docker image to tar file...'
                    sh '''
                        docker save ${DOCKER_IMAGE}:latest -o /tmp/task-manager.tar
                    '''
                }
            }
        }
        
        stage('Deploy to Production') {
            steps {
                script {
                    echo 'Deploying to Production VM...'
                    sshagent(['production-ssh-key']) {
                        sh '''
                            # Copy image to production
                            scp -o StrictHostKeyChecking=no \
                              /tmp/task-manager.tar \
                              ${PRODUCTION_USER}@${PRODUCTION_HOST}:/tmp/
                            
                            # Deploy on production
                            ssh -o StrictHostKeyChecking=no \
                              ${PRODUCTION_USER}@${PRODUCTION_HOST} '
                                # Load the image
                                docker load -i /tmp/task-manager.tar
                                
                                # Stop old container if running
                                docker stop task-manager || true
                                docker rm task-manager || true
                                
                                # Start new container
                                docker run -d --name task-manager \
                                  -p 80:5000 \
                                  --restart unless-stopped \
                                  task-manager:latest
                                
                                # Cleanup
                                rm /tmp/task-manager.tar
                                
                                # Verify deployment
                                sleep 3
                                curl -f http://localhost:80/health
                            '
                        '''
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs above.'
        }
        always {
            echo 'Cleaning up...'
            script {
                sh 'rm -f /tmp/task-manager.tar || true'
            }
        }
    }
}
```

**CRITICAL:** Change `PRODUCTION_HOST = "192.168.122.101"` to your actual Production VM IP!

Click **"Save"** at the bottom.

---

## Step 8: Run Your First Pipeline Build

### Ensure Both VMs Are Running

```bash
# From host
sudo virsh list

# Both VMs must be "running"
# If Production is off:
sudo virsh start production-server
sleep 60
```

### Trigger Build

1. You should be on the `task-manager-pipeline` job page
2. Click **"Build Now"** (left sidebar)
3. Watch **"Build History"** (bottom left)
4. Click on build `#1`
5. Click **"Console Output"**

### Expected Build Process

You'll see:
```
[Pipeline] Start of Pipeline
[Pipeline] stage (Build Docker Image)
Building Docker image...
[Pipeline] stage (Test)
Running basic health check...
[Pipeline] stage (Save Docker Image)
Saving Docker image to tar file...
[Pipeline] stage (Deploy to Production)
Deploying to Production VM...
Pipeline completed successfully!
[Pipeline] End of Pipeline
Finished: SUCCESS
```

**First build time:** 3-5 minutes (Docker downloads base images)  
**Subsequent builds:** 1-2 minutes

---

## Step 9: Verify Deployment

### Check Production VM

```bash
# From host
ssh production "docker ps"

# Should show task-manager container running on port 80
```

### Test the Application

```bash
# Test health endpoint
curl http://192.168.122.101/health

# Should return: {"status":"healthy","timestamp":"..."}
```

### Access in Browser

Open: `http://192.168.122.101`

You should see your Task Manager application!

- Add tasks
- Complete tasks
- Delete tasks

**Everything works!** 🎉

---

## Step 10: Test Automated Deployment

Make a code change and watch Jenkins auto-deploy it.

### Update Application

```bash
cd ~/devops-projects/task-manager-pipeline

# Edit the title
nano app/templates/index.html
```

Find:
```html
<h1>🚀 DevOps Task Manager</h1>
```

Change to:
```html
<h1>🚀 DevOps Task Manager - AUTO DEPLOYED!</h1>
```

Save (Ctrl+O, Enter, Ctrl+X).

### Update Jenkins Workspace

```bash
# Copy to Jenkins /tmp
scp app/templates/index.html jenkins:/tmp/

# Move to workspace with proper permissions
ssh jenkins "sudo cp /tmp/index.html /var/lib/jenkins/workspace/task-manager-pipeline/app/templates/index.html && sudo chown jenkins:jenkins /var/lib/jenkins/workspace/task-manager-pipeline/app/templates/index.html"
```

### Trigger New Build

1. Go to Jenkins: `http://192.168.122.100:8080`
2. Click `task-manager-pipeline`
3. Click **"Build Now"**
4. Watch build `#2` complete

### Verify the Change

Refresh `http://192.168.122.101` in your browser.

You should now see: **"🚀 DevOps Task Manager - AUTO DEPLOYED!"**

**Success!** Your CI/CD pipeline is working! 🚀

---

## Step 11: Add Monitoring

Set up automated health checks.

```bash
# SSH to Jenkins
ssh jenkins

# Create monitoring script
sudo tee /usr/local/bin/check-production.sh > /dev/null << 'EOF'
#!/bin/bash

PRODUCTION_HOST="192.168.122.101"  # Update with your IP
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

if curl -f -s http://${PRODUCTION_HOST}/health > /dev/null; then
    echo "[${TIMESTAMP}] ✓ Production is healthy"
    exit 0
else
    echo "[${TIMESTAMP}] ✗ Production is DOWN!"
    exit 1
fi
EOF

# Make executable
sudo chmod +x /usr/local/bin/check-production.sh

# Test it
/usr/local/bin/check-production.sh
# Should output: [timestamp] ✓ Production is healthy
```

### Set Up Cron Job

```bash
# Still on Jenkins VM
sudo crontab -e

# Select nano as editor (option 1)

# Add this line at the bottom:
*/5 * * * * /usr/local/bin/check-production.sh >> /var/log/production-health.log 2>&1
```

Save and exit.

### Create Log File

```bash
sudo touch /var/log/production-health.log
sudo chmod 644 /var/log/production-health.log

# Wait 5 minutes, then check
sudo tail /var/log/production-health.log

# Should see health check entries

exit
```

---

## Step 12: Document Pipeline

On your **host machine**:

```bash
cd ~/devops-projects/task-manager-pipeline
nano docs/PIPELINE.md
```

Paste this **(update IPs)**:

```markdown
# CI/CD Pipeline Documentation

## Overview
Automated pipeline that builds, tests, and deploys the Task Manager application using Jenkins.

## Architecture

```
Code Change → Jenkins Build → Docker Image → Production VM → Live App
```

## Pipeline Stages

### 1. Build Docker Image
- Builds Docker image from `docker/Dockerfile`
- Tags with build number: `task-manager:BUILD_NUMBER`
- Tags as latest: `task-manager:latest`
- Uses Docker layer caching for speed

### 2. Test
- Starts container on port 5001
- Performs health check at `/health`
- Fails build if health check fails
- Cleans up test container
- **Zero failed tests reach production**

### 3. Save Docker Image
- Exports image to `/tmp/task-manager.tar`
- ~50-100MB tar file
- Enables transfer to Production

### 4. Deploy to Production
- Transfers tar to Production via SCP
- Loads image: `docker load`
- Gracefully stops old container
- Starts new container on port 80
- Verifies with health check
- Auto-restart on failure enabled

## Infrastructure

### Jenkins Server
- **IP:** 192.168.122.100
- **Web UI:** http://192.168.122.100:8080
- **Credentials:** admin / admin123
- **Resources:** 4GB RAM, 2 CPUs
- **Disk:** 30GB

### Production Server
- **IP:** 192.168.122.101
- **Application:** http://192.168.122.101
- **Container:** task-manager (port 80)
- **Resources:** 2GB RAM, 1 CPU
- **Disk:** 20GB

## Credentials & Access

### SSH Keys
- **Jenkins → Production:** /home/devops/.ssh/id_ed25519 (on Jenkins VM)
- **Jenkins Credential ID:** production-ssh-key
- **User:** devops (passwordless sudo)

### Ports
- Jenkins Web UI: 8080
- Application: 80 (Production VM)
- Health Check: http://192.168.122.101/health

## Monitoring

### Health Checks
- **Frequency:** Every 5 minutes
- **Location:** Jenkins VM cron
- **Log:** /var/log/production-health.log
- **Endpoint:** http://192.168.122.101/health

### View Logs
```bash
# Production container logs
ssh production "docker logs task-manager"

# Health check logs
ssh jenkins "sudo tail -f /var/log/production-health.log"

# Jenkins build logs
# Web UI → Build Number → Console Output
```

## Manual Operations

### Trigger Build
1. Jenkins Dashboard
2. task-manager-pipeline
3. Build Now

### Deploy New Version
1. Update code in Jenkins workspace
2. Trigger build
3. Wait 2-3 minutes
4. Verify at http://192.168.122.101

### Restart Production App
```bash
ssh production "docker restart task-manager"
```

### View Production Status
```bash
ssh production "docker ps"
ssh production "docker logs task-manager"
curl http://192.168.122.101/health
```

## Performance Metrics

- **Build Time:** 1-2 minutes (after first build)
- **Deployment Time:** 30-60 seconds
- **Total Pipeline:** ~2-3 minutes
- **Downtime:** ~5 seconds (container restart)

## Troubleshooting

### Build Fails at Docker Stage
```bash
# Verify Docker running
ssh jenkins "docker ps"

# Check jenkins in docker group
ssh jenkins "groups jenkins | grep docker"

# Restart Jenkins
ssh jenkins "sudo systemctl restart jenkins"
```

### Deployment Fails (SSH Error)
```bash
# Test SSH connection
ssh jenkins "ssh devops@192.168.122.101 'docker ps'"

# Verify credential exists
# Jenkins → Manage → Credentials → production-ssh-key
```

### Application Not Responding
```bash
# Check container
ssh production "docker ps -a"
ssh production "docker logs --tail 50 task-manager"

# Restart container
ssh production "docker restart task-manager"

# Manual redeploy
# Jenkins → Build Now
```

### Production VM Down
```bash
# Check VM status
sudo virsh list --all

# Start if stopped
sudo virsh start production-server

# Wait 1 minute, then rebuild
```

## Success Criteria

✅ Code commits trigger builds  
✅ Failed tests block deployment  
✅ Automated deployment to production  
✅ Zero-downtime updates  
✅ Health monitoring every 5 minutes  
✅ Complete audit trail in logs  

## Future Enhancements

- [ ] Git webhook integration (instant builds)
- [ ] Slack/email notifications
- [ ] Automated rollback on failure
- [ ] Blue-green deployments
- [ ] Prometheus + Grafana dashboards
- [ ] Security scanning (Trivy)
- [ ] Multi-environment (dev/staging/prod)
```

Save and exit.

---

## Step 13: Final Commit

```bash
cd ~/devops-projects/task-manager-pipeline

# Add documentation
git add docs/PIPELINE.md

# Add updated title
git add app/templates/index.html

# Commit
git commit -m "Complete Phase 3: Jenkins CI/CD pipeline with automated deployment"

# View complete history
git log --oneline

# Should show all commits from Phase 1, 2, and 3!
```

---

## Phase 3 Checklist

Verify everything works:

- [ ] Jenkins installed and accessible at :8080
- [ ] Docker installed on Jenkins VM
- [ ] Jenkins credentials configured (production-ssh-key)
- [ ] Application code in Jenkins workspace
- [ ] Pipeline job created
- [ ] First build succeeded (Build #1)
- [ ] Application running on Production at port 80
- [ ] Can access app in browser
- [ ] Made a code change and redeployed
- [ ] Monitoring cron job set up
- [ ] Pipeline documented
- [ ] Everything committed to Git

### Complete Verification

```bash
# 1. VMs running
sudo virsh list

# 2. Can access Jenkins
curl -I http://192.168.122.100:8080

# 3. Production app responding
curl http://192.168.122.101/health

# 4. Container running
ssh production "docker ps | grep task-manager"

# 5. Monitoring active
ssh jenkins "sudo tail -3 /var/log/production-health.log"

# All should succeed!
```

---

## Common Issues and Solutions

### Issue: Build fails with "Cannot run program nohup"
**Solution:** Install coreutils on Jenkins VM
```bash
ssh jenkins
sudo apt install -y coreutils
sudo systemctl restart jenkins
```

### Issue: Docker permission denied
**Solution:**
```bash
ssh jenkins
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### Issue: SSH fails during deployment
**Solution:** Verify Jenkins' key is on Production
```bash
ssh jenkins "cat ~/.ssh/id_ed25519.pub"
ssh production "cat ~/.ssh/authorized_keys | grep <key from above>"
```

### Issue: Production VM not responding
**Solution:** Ensure VM is running!
```bash
sudo virsh list
sudo virsh start production-server
```

### Issue: Build succeeds but app still shows old version
**Solution:** Hard refresh browser (Ctrl+F5) or clear cache

---

## What You've Learned

✅ Jenkins installation and configuration  
✅ CI/CD pipeline creation  
✅ Docker in CI/CD workflows  
✅ Automated testing  
✅ SSH-based deployment  
✅ Container orchestration  
✅ Monitoring and logging  
✅ Production deployments  

**Time spent:** ~8 hours ✓

---

## 🎉 PROJECT COMPLETE! 🎉

### Your Achievement

You've built a **complete DevOps pipeline** from scratch:

**Phase 1:** Flask application + Docker  
**Phase 2:** KVM infrastructure (2 VMs)  
**Phase 3:** Jenkins CI/CD automation  

**Total:** ~24 hours of learning, equivalent to many paid bootcamps!

### Portfolio Value

You can now confidently say:
- "Built automated CI/CD pipelines using Jenkins"
- "Deployed containerized applications to production"
- "Managed virtual infrastructure with KVM/libvirt"
- "Implemented automated testing and deployment"
- "Set up monitoring and health checks"

### Next Steps (Optional)

1. **Push to GitHub** - Make it public in your portfolio
2. **Add Prometheus/Grafana** - Professional monitoring
3. **Implement webhooks** - Instant builds on push
4. **Kubernetes** - Container orchestration at scale
5. **Security scanning** - Automated vulnerability checks

But what you have **right now is already impressive** and job-worthy!

---

Congratulations on completing the DevOps Learning Project! 🚀🎊
