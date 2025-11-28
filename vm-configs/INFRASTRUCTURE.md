# VM Infrastructure Documentation

## Virtual Machines

### Jenkins Server
- **Hostname:** jenkins-server
- **IP Address:** 192.168.122.83
- **Memory:** 4GB
- **CPUs:** 2
- **Disk:** 30GB
- **Purpose:** CI/CD automation server
- **Packages:** Java 17, Git, curl, wget
- **SSH Access:** `ssh jenkins`

### Production Server
- **Hostname:** production-server
- **IP Address:** 192.168.122.127
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
ping 192.168.122.127
ssh devops@192.168.122.127

# Check if Jenkins' public key is on Production:
ssh production "cat ~/.ssh/authorized_keys"
