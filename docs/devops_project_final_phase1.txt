# DevOps Learning Project - Phase 1: Application Development (FINAL)

## Overview (Week 1 - ~8 hours)

Build a Flask web application and containerize it with Docker. This is your foundation for the CI/CD pipeline.

**What you'll build:** A task manager web app with a clean UI that can add, complete, and delete tasks.

---

## Prerequisites

- OpenMandriva Lx or similar Linux distribution
- 32GB RAM, 2TB storage (your hardware is excellent!)
- Basic terminal/command line knowledge
- Internet connection

---

## Step 1: Environment Setup

### Install Required Tools

```bash
# Update system
sudo dnf update -y

# Install Docker
sudo dnf install docker -y
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
# Log out and back in for group changes to take effect

# Install Git
sudo dnf install git -y

# Install Python development tools
sudo dnf install python3-pip python3-venv -y

# Install text editor (if needed)
sudo dnf install nano -y

# Verify installations
docker --version
git --version
python3 --version
```

**After installing Docker, log out and log back in** for group permissions to take effect.

---

## Step 2: Create Project Structure

```bash
# Create main project directory
mkdir -p ~/devops-projects/task-manager-pipeline
cd ~/devops-projects/task-manager-pipeline

# Create subdirectories
mkdir -p app/{templates,static} docker jenkins docs vm-configs

# Verify structure
ls -la
```

You should see:
```
app/
docker/
jenkins/
docs/
vm-configs/
```

---

## Step 3: Build the Flask Application

### Create Backend (app/app.py)

```bash
nano app/app.py
```

Paste this complete code:

```python
from flask import Flask, render_template, request, jsonify
import json
import os
from datetime import datetime

app = Flask(__name__)
TASKS_FILE = 'tasks.json'

def load_tasks():
    if os.path.exists(TASKS_FILE):
        with open(TASKS_FILE, 'r') as f:
            return json.load(f)
    return []

def save_tasks(tasks):
    with open(TASKS_FILE, 'w') as f:
        json.dump(tasks, f, indent=2)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/api/tasks', methods=['GET'])
def get_tasks():
    return jsonify(load_tasks())

@app.route('/api/tasks', methods=['POST'])
def add_task():
    task = request.json
    tasks = load_tasks()
    task['id'] = len(tasks) + 1
    task['created'] = datetime.now().isoformat()
    task['completed'] = False
    tasks.append(task)
    save_tasks(tasks)
    return jsonify(task), 201

@app.route('/api/tasks/<int:task_id>', methods=['DELETE'])
def delete_task(task_id):
    tasks = load_tasks()
    tasks = [t for t in tasks if t['id'] != task_id]
    save_tasks(tasks)
    return '', 204

@app.route('/api/tasks/<int:task_id>/toggle', methods=['PUT'])
def toggle_task(task_id):
    tasks = load_tasks()
    for task in tasks:
        if task['id'] == task_id:
            task['completed'] = not task['completed']
            break
    save_tasks(tasks)
    return jsonify({'success': True})

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'timestamp': datetime.now().isoformat()})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

Save with Ctrl+O, Enter, Ctrl+X.

### Create Dependencies (app/requirements.txt)

```bash
nano app/requirements.txt
```

Paste:
```
Flask==3.0.0
gunicorn==21.2.0
```

Save and exit.

### Create Frontend HTML (app/templates/index.html)

```bash
nano app/templates/index.html
```

Paste:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>DevOps Task Manager</title>
    <link rel="stylesheet" href="/static/style.css">
</head>
<body>
    <div class="container">
        <h1>🚀 DevOps Task Manager</h1>
        <p class="subtitle">Built with CI/CD Pipeline</p>
        
        <div class="add-task">
            <input type="text" id="taskInput" placeholder="Enter a new task...">
            <button onclick="addTask()">Add Task</button>
        </div>
        
        <div id="taskList" class="task-list"></div>
    </div>

    <script src="/static/app.js"></script>
</body>
</html>
```

Save and exit.

### Create CSS (app/static/style.css)

```bash
nano app/static/style.css
```

Paste:
```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
    padding: 20px;
}

.container {
    max-width: 600px;
    margin: 0 auto;
    background: white;
    border-radius: 10px;
    padding: 30px;
    box-shadow: 0 10px 40px rgba(0,0,0,0.2);
}

h1 {
    color: #333;
    margin-bottom: 10px;
}

.subtitle {
    color: #666;
    margin-bottom: 30px;
    font-size: 14px;
}

.add-task {
    display: flex;
    gap: 10px;
    margin-bottom: 20px;
}

input[type="text"] {
    flex: 1;
    padding: 12px;
    border: 2px solid #ddd;
    border-radius: 5px;
    font-size: 16px;
}

button {
    padding: 12px 24px;
    background: #667eea;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    font-size: 16px;
    transition: background 0.3s;
}

button:hover {
    background: #5568d3;
}

.task-list {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.task-item {
    display: flex;
    align-items: center;
    padding: 15px;
    background: #f8f9fa;
    border-radius: 5px;
    transition: all 0.3s;
}

.task-item:hover {
    background: #e9ecef;
}

.task-item.completed {
    opacity: 0.6;
}

.task-item.completed .task-text {
    text-decoration: line-through;
}

.task-checkbox {
    margin-right: 15px;
    width: 20px;
    height: 20px;
    cursor: pointer;
}

.task-text {
    flex: 1;
    font-size: 16px;
    color: #333;
}

.delete-btn {
    padding: 8px 16px;
    background: #dc3545;
    font-size: 14px;
}

.delete-btn:hover {
    background: #c82333;
}
```

Save and exit.

### Create JavaScript (app/static/app.js)

```bash
nano app/static/app.js
```

Paste:
```javascript
async function loadTasks() {
    const response = await fetch('/api/tasks');
    const tasks = await response.json();
    
    const taskList = document.getElementById('taskList');
    taskList.innerHTML = '';
    
    tasks.forEach(task => {
        const taskDiv = document.createElement('div');
        taskDiv.className = `task-item ${task.completed ? 'completed' : ''}`;
        taskDiv.innerHTML = `
            <input type="checkbox" class="task-checkbox" 
                   ${task.completed ? 'checked' : ''} 
                   onchange="toggleTask(${task.id})">
            <span class="task-text">${task.title}</span>
            <button class="delete-btn" onclick="deleteTask(${task.id})">Delete</button>
        `;
        taskList.appendChild(taskDiv);
    });
}

async function addTask() {
    const input = document.getElementById('taskInput');
    const title = input.value.trim();
    
    if (!title) return;
    
    await fetch('/api/tasks', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({title})
    });
    
    input.value = '';
    loadTasks();
}

async function deleteTask(id) {
    await fetch(`/api/tasks/${id}`, {method: 'DELETE'});
    loadTasks();
}

async function toggleTask(id) {
    await fetch(`/api/tasks/${id}/toggle`, {method: 'PUT'});
    loadTasks();
}

document.addEventListener('DOMContentLoaded', () => {
    loadTasks();
    document.getElementById('taskInput').addEventListener('keypress', (e) => {
        if (e.key === 'Enter') addTask();
    });
});
```

Save and exit.

---

## Step 4: Test the Application Locally

```bash
cd ~/devops-projects/task-manager-pipeline/app

# Create virtual environment
python3 -m venv venv

# Activate it
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run the application
python app.py
```

**Expected output:**
```
 * Running on http://0.0.0.0:5000
```

### Test in Browser

Open: `http://localhost:5000`

**You should see:**
- Purple gradient background
- "🚀 DevOps Task Manager" title
- Input field and "Add Task" button

**Test functionality:**
- Add a task
- Mark it complete (checkbox)
- Delete it

**Test health endpoint:** `http://localhost:5000/health`

Should return JSON with status and timestamp.

Press Ctrl+C to stop the server when done.

**Deactivate venv:**
```bash
deactivate
```

---

## Step 5: Create Docker Configuration

### Dockerfile (docker/Dockerfile)

```bash
cd ~/devops-projects/task-manager-pipeline
nano docker/Dockerfile
```

Paste:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Copy requirements first for better caching
COPY app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app/ .

# Create directory for data persistence
RUN mkdir -p /app/data

EXPOSE 5000

# Use gunicorn for production
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

Save and exit.

### Docker Compose (docker/docker-compose.yml)

```bash
nano docker/docker-compose.yml
```

Paste:
```yaml
version: '3.8'

services:
  task-manager:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    ports:
      - "5000:5000"
    volumes:
      - task-data:/app/data
    environment:
      - FLASK_ENV=production
    restart: unless-stopped

volumes:
  task-data:
```

Save and exit.

---

## Step 6: Test Docker Build

```bash
cd ~/devops-projects/task-manager-pipeline

# Build the Docker image
docker build -t task-manager:latest -f docker/Dockerfile .

# This takes 1-2 minutes on first build
# Watch for "Successfully built" message

# Run the container
docker run -d -p 5000:5000 --name task-manager-test task-manager:latest

# Check if it's running
docker ps

# Should show task-manager-test container

# Test the health endpoint
curl http://localhost:5000/health

# Should return: {"status":"healthy","timestamp":"..."}

# Test in browser
# Open: http://localhost:5000
# Should work exactly like before!

# View logs
docker logs task-manager-test

# Stop and remove test container
docker stop task-manager-test
docker rm task-manager-test
```

---

## Step 7: Initialize Git Repository <<<

### Create .gitignore

```bash
cd ~/devops-projects/task-manager-pipeline
nano .gitignore
```

Paste this comprehensive .gitignore:

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
venv/
env/
ENV/
*.egg
*.egg-info/
dist/
build/
.eggs/
pip-log.txt
pip-delete-this-directory.txt

# Application runtime
tasks.json
*.log
*.sqlite
*.db

# IDE and Editors
.vscode/
.idea/
*.swp
*.swo
*~
.project
.pydevproject
.settings/
*.sublime-project
*.sublime-workspace

# OS
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db
desktop.ini

# Docker
*.tar
*.tar.gz

# VM Images and Virtualization - CRITICAL!
*.qcow2
*.qcow
*.iso
*.img
*.vmdk
*.vdi
*.vhd
*.ova
*.ovf

# Java/Jenkins artifacts
*.jar
*.war
*.ear
*.class
target/

# Node.js
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*
package-lock.json
yarn.lock

# Archives
*.zip
*.rar
*.7z
*.gz
*.bz2

# Temporary files
tmp/
temp/
*.tmp
*.bak
*.swp
*~.nib

# Security - never commit these!
*.key
*.pem
*.crt
*.p12
*.pfx
id_rsa
id_dsa
id_ed25519
*.secret
.env
.env.local
secrets.yml

# Jenkins specific
.jenkins/
jenkins-home/

# Cloud-init generated files
*-cidata.iso

# VM configs - keep YAML, not images
vm-configs/*.qcow2
vm-configs/*.iso
vm-configs/*.img

# Backup files
*.backup
*-backup
.git.backup/

# Large binary files
*.exe
*.dll
*.dylib

# Test coverage
htmlcov/
.tox/
.coverage
.coverage.*
.cache
nosetests.xml
coverage.xml
*.cover

# Documentation builds
docs/_build/
site/

# Virtual environment in app directory
app/venv/
```

Save and exit.

### Initialize Git

```bash
cd ~/devops-projects/task-manager-pipeline

# Configure Git (use your actual info)
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Initialize repository
git init

# Add files (respects .gitignore)
git add .

# Check what will be committed
git status

# Should show app/, docker/, .gitignore
# Should NOT show venv/, *.pyc, or VM images

# Make first commit
git commit -m "Initial commit: Flask task manager with Docker"

# Verify
git log --oneline
```

---

## Phase 1 Checklist

Verify you completed everything:

- [ ] Docker installed and working (can run containers)
- [ ] Project directory structure created
- [ ] Flask application files created (app.py, HTML, CSS, JS)
- [ ] Application runs locally and works (can add/delete/complete tasks)
- [ ] Docker image builds successfully
- [ ] Application works in Docker container
- [ ] .gitignore created with comprehensive exclusions
- [ ] Git repository initialized
- [ ] First commit made
- [ ] No venv/ or large files in Git (check with `git ls-files`)

### Verification Commands

```bash
cd ~/devops-projects/task-manager-pipeline

# Check Git repo size (should be under 1MB)
du -sh .git

# Check what Git is tracking
git ls-files

# Should only show source code, no venv, no VM images

# Verify Docker image exists
docker images | grep task-manager

# Check app directory size without venv
du -sh app/ --exclude=venv

# Should be under 500KB
```

---

## Common Issues and Solutions

### Issue: Docker permission denied
```bash
sudo usermod -aG docker $USER
# Log out and back in
```

### Issue: App shows only health endpoint text
- You're visiting `/health` instead of `/`
- Visit: `http://localhost:5000` (no `/health`)

### Issue: Can't check or delete tasks
- app.js file might be incomplete or has errors
- Recreate it exactly as shown in Step 3
- Check browser console (F12) for JavaScript errors

### Issue: Git repository is huge (>100MB)
- You committed venv/ or VM images by mistake
- Remove .git and start fresh:
```bash
rm -rf .git
git init
git add .
git commit -m "Initial commit: Flask task manager with Docker"
```

---

## What You've Learned

✅ Flask web application development  
✅ Frontend (HTML/CSS/JavaScript)  
✅ Backend API design (REST)  
✅ Docker containerization  
✅ Dockerfile best practices  
✅ Git version control  
✅ .gitignore configuration  

**Time spent:** ~8 hours ✓

---

## Ready for Phase 2?

Phase 2 will build the virtual infrastructure where your CI/CD pipeline will run:
- 2 KVM virtual machines (Jenkins + Production)
- Automated VM provisioning with cloud-init
- Secure SSH networking
- Infrastructure as code

Let me know when you're ready to continue!
