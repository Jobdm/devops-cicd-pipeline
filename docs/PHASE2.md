# DevOps Learning Project - Phase 2: KVM Infrastructure

## Overview

Create two virtual machines using KVM that will host your CI/CD pipeline:
- **Jenkins VM:** Automation server (4GB RAM, 2 CPUs)
- **Production VM:** Application deployment target (2GB RAM, 1 CPU)

---

## Prerequisites from Phase 1

✅ Completed Phase 1  
✅ Application code in ~/devops-projects/task-manager-pipeline  
✅ Git repository initialized  

---

## Step 1: Verify KVM Environment

### Check KVM Status

```bash
# Verify libvirt is running
sudo systemctl status libvirtd

# Should show "active (running)"

# Check your user is in libvirt group
groups | grep libvirt

# If NOT in libvirt group:
sudo usermod -a -G libvirt $USER
# Log out and back in

# Verify KVM support
egrep -c '(vmx|svm)' /proc/cpuinfo
# Should return a number > 0

# Check default network
sudo virsh net-list --all

# If default network is inactive:
sudo virsh net-start default
sudo virsh net-autostart default
```

---

## Step 2: Download Ubuntu Server Image

```bash
cd ~/devops-projects/task-manager-pipeline/vm-configs

# Download Ubuntu 22.04 LTS cloud image
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Verify download (should be ~600MB)
ls -lh jammy-server-cloudimg-amd64.img
```

---

## Step 3: Create SSH Keys

```bash
# Generate SSH key pair
ssh-keygen -t ed25519 -f ~/.ssh/devops_project_key -C "devops-project-vms"

# Press Enter for all prompts (no passphrase for automation)

# View your public key
cat ~/.ssh/devops_project_key.pub

# Should start with: ssh-ed25519 AAAA...
# COPY THIS - you'll need it in the next step!

# Set proper permissions
chmod 600 ~/.ssh/devops_project_key
chmod 644 ~/.ssh/devops_project_key.pub
```

---

## Step 4: Create Cloud-Init Configurations

### Jenkins VM Configuration

```bash
cd ~/devops-projects/task-manager-pipeline/vm-configs
mkdir -p jenkins production

nano jenkins/user-data
```

Paste this content, **replacing YOUR_PUBLIC_KEY_HERE** with your actual public key from Step 3:

```yaml
#cloud-config
hostname: jenkins-server
fqdn: jenkins-server.local

users:
  - name: devops
    ssh_authorized_keys:
      - YOUR_PUBLIC_KEY_HERE
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups: sudo
    shell: /bin/bash

package_update: true
package_upgrade: true

packages:
  - curl
  - wget
  - git
  - openjdk-17-jdk
  - apt-transport-https
  - ca-certificates
  - gnupg
  - software-properties-common

runcmd:
  - echo "Jenkins VM initialized" > /home/devops/setup-complete.txt
  - chown devops:devops /home/devops/setup-complete.txt

ssh_pwauth: false
timezone: America/New_York

power_state:
  mode: reboot
  timeout: 300
  condition: true
```

**CRITICAL:** Replace `YOUR_PUBLIC_KEY_HERE` with the output from `cat ~/.ssh/devops_project_key.pub`

Save (Ctrl+O, Enter, Ctrl+X).

### Jenkins Meta-Data

```bash
nano jenkins/meta-data
```

Paste:
```yaml
instance-id: jenkins-vm-001
local-hostname: jenkins-server
```

Save and exit.

### Production VM Configuration

```bash
nano production/user-data
```

Paste this content, **replacing YOUR_PUBLIC_KEY_HERE** with your public key:

```yaml
#cloud-config
hostname: production-server
fqdn: production-server.local

users:
  - name: devops
    ssh_authorized_keys:
      - YOUR_PUBLIC_KEY_HERE
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups: sudo, docker
    shell: /bin/bash

package_update: true
package_upgrade: true

packages:
  - curl
  - wget
  - git
  - docker.io
  - docker-compose

runcmd:
  - systemctl enable docker
  - systemctl start docker
  - usermod -aG docker devops
  - echo "Production VM initialized" > /home/devops/setup-complete.txt
  - chown devops:devops /home/devops/setup-complete.txt

ssh_pwauth: false
timezone: America/New_York

power_state:
  mode: reboot
  timeout: 300
  condition: true
```

**Again, replace YOUR_PUBLIC_KEY_HERE with your actual public key.**

Save and exit.

### Production Meta-Data

```bash
nano production/meta-data
```

Paste:
```yaml
instance-id: production-vm-001
local-hostname: production-server
```

Save and exit.

---

## Step 5: Create Cloud-Init ISO Images

```bash
cd ~/devops-projects/task-manager-pipeline/vm-configs

# OpenMandriva uses mkisofs (part of cdrtools)
# Verify it's available
which mkisofs

# Create Jenkins cloud-init ISO
mkisofs -output jenkins-cidata.iso \
  -volid cidata -joliet -rock \
  jenkins/user-data jenkins/meta-data

# Create Production cloud-init ISO
mkisofs -output production-cidata.iso \
  -volid cidata -joliet -rock \
  production/user-data production/meta-data

# Verify ISOs were created
ls -lh *-cidata.iso

# Should show two ISO files, each ~10-20KB
```

---

## Step 6: Create Virtual Disk Images

```bash
cd ~/devops-projects/task-manager-pipeline/vm-configs

# Create Jenkins VM disk (30GB for Jenkins + builds)
qemu-img create -f qcow2 -F qcow2 \
  -b jammy-server-cloudimg-amd64.img \
  jenkins-disk.qcow2 30G

# Create Production VM disk (20GB)
qemu-img create -f qcow2 -F qcow2 \
  -b jammy-server-cloudimg-amd64.img \
  production-disk.qcow2 20G

# Verify disks
ls -lh *.qcow2
```

---

## Step 7: Create Jenkins VM

```bash
cd ~/devops-projects/task-manager-pipeline/vm-configs

# Get absolute path (needed for virt-install)
VMCONFIG_PATH=$(pwd)

# Create Jenkins VM
sudo virt-install \
  --name jenkins-server \
  --memory 4096 \
  --vcpus 2 \
  --disk path=${VMCONFIG_PATH}/jenkins-disk.qcow2,format=qcow2 \
  --disk path=${VMCONFIG_PATH}/jenkins-cidata.iso,device=cdrom \
  --os-variant ubuntu22.04 \
  --network network=default \
  --graphics none \
  --console pty,target_type=serial \
  --import \
  --noautoconsole
```

**Wait 3-4 minutes** for cloud-init to complete and VM to reboot.

### Monitor Boot (Optional)

```bash
# Watch the VM boot
sudo virsh console jenkins-server

# Press Enter if blank screen
# You'll see boot messages
# When you see "login:" prompt, cloud-init is done

# Exit console: Press Ctrl+] (Ctrl and right bracket)
```

---

## Step 8: Create Production VM

```bash
cd ~/devops-projects/task-manager-pipeline/vm-configs
VMCONFIG_PATH=$(pwd)

# Create Production VM
sudo virt-install \
  --name production-server \
  --memory 2048 \
  --vcpus 1 \
  --disk path=${VMCONFIG_PATH}/production-disk.qcow2,format=qcow2 \
  --disk path=${VMCONFIG_PATH}/production-cidata.iso,device=cdrom \
  --os-variant ubuntu22.04 \
  --network network=default \
  --graphics none \
  --console pty,target_type=serial \
  --import \
  --noautoconsole
```

**Wait 3-4 minutes** for cloud-init to complete.

---

## Step 9: Find VM IP Addresses

```bash
# Wait a few minutes after creating VMs
sleep 180

# Get Jenkins VM IP
sudo virsh domifaddr jenkins-server

# Get Production VM IP
sudo virsh domifaddr production-server

# Alternative method:
sudo virsh net-dhcp-leases default
```

**Record these IPs!** You'll need them constantly.

Example output:
```
jenkins-server: 192.168.122.100
production-server: 192.168.122.101
```

### If IPs Don't Show

```bash
# Check VMs are running
sudo virsh list

# Wait longer (cloud-init takes time)
sleep 120

# Try again
sudo virsh net-dhcp-leases default
```

---

## Step 10: Test SSH Access

```bash
# Test Jenkins VM (replace with your actual IP)
ssh -i ~/.ssh/devops_project_key devops@192.168.122.100

# You should log in without password!

# Verify setup completed
cat /home/devops/setup-complete.txt
# Should say: "Jenkins VM initialized"

# Check Java installed
java -version

# Exit
exit

# Test Production VM
ssh -i ~/.ssh/devops_project_key devops@192.168.122.101

# Check Docker installed
docker --version

# Exit
exit
```

---

## Step 11: Create SSH Config for Easy Access

```bash
nano ~/.ssh/config
```

Add this content **(update IPs to match your VMs)**:

```
Host jenkins
    HostName 192.168.122.100
    User devops
    IdentityFile ~/.ssh/devops_project_key
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

Host production
    HostName 192.168.122.101
    User devops
    IdentityFile ~/.ssh/devops_project_key
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```

Save and set permissions:
```bash
chmod 600 ~/.ssh/config
```

Now you can connect easily:
```bash
# Connect to Jenkins
ssh jenkins

# Connect to Production
ssh production
```

---

## Step 12: Set Up Jenkins → Production SSH Access

Production VM needs Jenkins' public key for automated deployments.

```bash
# SSH to Jenkins
ssh jenkins

# Generate SSH key on Jenkins VM
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# View Jenkins' public key
cat ~/.ssh/id_ed25519.pub

# COPY THIS OUTPUT

exit
```

Add Jenkins' public key to Production:

```bash
# SSH to Production
ssh production

# Edit authorized_keys
nano ~/.ssh/authorized_keys

# Paste Jenkins' public key on a NEW LINE (don't delete existing keys)
# Save and exit

exit
```

### Test Jenkins → Production Connection

```bash
# From Jenkins
ssh jenkins

# Try to SSH to Production (use your Production IP)
ssh devops@192.168.122.101

# Type "yes" when asked about fingerprint

# Should connect! You're now on Production from Jenkins

# Exit back to Jenkins
exit

# Exit back to host
exit
```

---

## Step 13: Test VM Networking

```bash
# From host, connect to Jenkins
ssh jenkins

# Ping Production (use your Production IP)
ping -c 3 192.168.122.101

# Should get replies

# Test Docker on Production from Jenkins
ssh devops@192.168.122.101 "docker ps"

# Should show empty list (no containers yet)

exit
```

---

## Step 14: Set VMs to Auto-Start

```bash
# VMs will start automatically when host boots
sudo virsh autostart jenkins-server
sudo virsh autostart production-server

# Verify
sudo virsh list --all --autostart
```

---

## Step 15: Document Infrastructure

```bash
cd ~/devops-projects/task-manager-pipeline
nano vm-configs/INFRASTRUCTURE.md
```

Paste this content **(update with your actual IPs)**:

```markdown
# VM Infrastructure Documentation

## Virtual Machines

### Jenkins Server
- **Hostname:** jenkins-server
- **IP Address:** 192.168.122.100
- **Memory:** 4GB
- **CPUs:** 2
- **Disk:** 30GB
- **Purpose:** CI/CD automation server
- **Packages:** Java 17, Git, curl, wget
- **SSH Access:** `ssh jenkins`

### Production Server
- **Hostname:** production-server
- **IP Address:** 192.168.122.101
- **Memory:** 2GB
- **CPUs:** 1
- **Disk:** 20GB
- **Purpose:** Application deployment target
- **Packages:** Docker, Docker Compose, Git
- **SSH Access:** `ssh production`

## Network Configuration
- **Network:** libvirt default NAT network (192.168.122.0/24)
- **Host → VMs:** SSH via ~/.ssh/devops_project_key
- **Jenkins → Production:** SSH via Jenkins' ~/.ssh/id_ed25519
- **Internet Access:** Via NAT through host

## SSH Key Setup
- **Host Key:** ~/.ssh/devops_project_key
- **Jenkins Key:** /home/devops/.ssh/id_ed25519 (on Jenkins VM)
- **Both keys added to Production authorized_keys**

## VM Management Commands

### Start VMs
```bash
sudo virsh start jenkins-server
sudo virsh start production-server
```

### Stop VMs
```bash
sudo virsh shutdown jenkins-server
sudo virsh shutdown production-server
```

### Force stop (if shutdown hangs)
```bash
sudo virsh destroy jenkins-server
sudo virsh destroy production-server
```

### Check VM Status
```bash
sudo virsh list --all
```

### Connect to Console
```bash
sudo virsh console jenkins-server
# Exit with Ctrl+]
```

### Get VM IP Addresses
```bash
sudo virsh net-dhcp-leases default
```

### Delete VMs (DESTRUCTIVE!)
```bash
sudo virsh destroy jenkins-server
sudo virsh undefine jenkins-server --remove-all-storage

sudo virsh destroy production-server
sudo virsh undefine production-server --remove-all-storage
```

## Troubleshooting

### VM won't start
```bash
sudo virsh dominfo jenkins-server
sudo journalctl -u libvirtd | tail -20
```

### Can't get VM IP
```bash
# Wait 3-5 minutes after VM creation
sudo virsh net-dhcp-leases default

# Or check ARP
arp -an | grep 192.168.122
```

### SSH connection refused
- Wait for cloud-init (3-5 minutes)
- Check VM is running: `sudo virsh list`
- Try console: `sudo virsh console jenkins-server`
- Verify public key is correct in cloud-init configs

### Jenkins can't reach Production
```bash
# From Jenkins:
ssh jenkins
ping 192.168.122.101
ssh devops@192.168.122.101

# Check if Jenkins' public key is on Production:
ssh production "cat ~/.ssh/authorized_keys"
```
```

Save and exit.

---

## Step 16: Commit Your Work

```bash
cd ~/devops-projects/task-manager-pipeline

# Add cloud-init configs and documentation
# Note: Large files (.iso, .qcow2, .img) are in .gitignore
git add vm-configs/jenkins/user-data vm-configs/jenkins/meta-data
git add vm-configs/production/user-data vm-configs/production/meta-data
git add vm-configs/INFRASTRUCTURE.md

# Check what's being added
git status

# Should NOT include .iso or .qcow2 files

# Commit
git commit -m "Add KVM VM infrastructure configuration and documentation"

# Verify
git log --oneline
```

---

## Phase 2 Checklist

Verify everything is working:

- [ ] Jenkins VM created and running
- [ ] Production VM created and running
- [ ] Can SSH to Jenkins: `ssh jenkins`
- [ ] Can SSH to Production: `ssh production`
- [ ] Jenkins can SSH to Production
- [ ] VMs set to auto-start
- [ ] Infrastructure documented
- [ ] Changes committed to Git (configs only, not images)
- [ ] Recorded IP addresses for both VMs

### Verification Commands

```bash
# Check both VMs are running
sudo virsh list

# Quick connectivity test
ssh jenkins "hostname && java -version"
ssh production "hostname && docker --version"
ssh jenkins "ssh -o StrictHostKeyChecking=no devops@192.168.122.101 'docker ps'"

# All should work without errors
```

---

## Common Issues and Solutions

### Issue: VM IP address not showing
**Solution:** Wait longer (3-5 minutes), cloud-init takes time. Try `sudo virsh net-dhcp-leases default`

### Issue: SSH connection refused
**Solution:** Cloud-init still running. Wait 5 minutes, then try again.

### Issue: Wrong public key in cloud-init
**Solution:** Must recreate VM:
```bash
sudo virsh destroy jenkins-server
sudo virsh undefine jenkins-server
# Fix user-data file
# Recreate cloud-init ISO
# Recreate VM
```

### Issue: Jenkins can't SSH to Production
**Solution:** 
```bash
ssh jenkins
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
# Add this to Production's ~/.ssh/authorized_keys
```

### Issue: VMs slow or laggy
**Solution:** Hardware (32GB RAM) is perfect. Check host CPU usage: `top`

---

## Ready for Phase 3?

Phase 3 will install Jenkins and create your automated CI/CD pipeline:
- Jenkins installation and configuration
- Docker integration
- Automated build pipeline
- Deployment automation
- Monitoring setup
