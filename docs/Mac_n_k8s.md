# Ansible: To Configure Kubernetes on Docker on Mac

We’re going to automate:
- Installing dependencies
- Enabling Kubernetes in Docker Desktop
- Verifying the cluster
- Deploying a sample workload
- Making it repeatable

All from a single Ansible playbook.

## 🧭 Architecture Overview

### On macOS:
- Docker Desktop runs a Linux VM internally
- Kubernetes runs inside that VM
- You interact using kubectl from macOS

### Ansible will:
- Run locally (connection: local)
- Configure Docker
- Validate Kubernetes
- Deploy test resources

## 📦 Prerequisites
**Install:**
- Docker Desktop
- kubectl
- Ansible

**Example (Homebrew):**
- brew install ansible kubectl

**Verify:**
- docker --version
- kubectl version --client
- ansible --version

## 📁 Project Structure
```
ansible-k8s-mac/
├── site.yml
├── roles/
│   ├── docker/
│   ├── k8s/
│   └── deploy/
```

### 1️⃣ site.yml (Entry Point)
```
---
- name: Configure Kubernetes on Docker Desktop (Mac)
  hosts: localhost
  connection: local
  gather_facts: yes

  roles:
    - docker
    - k8s
    - deploy
```

Everything runs locally.

### 2️⃣ Role: Docker Configuration

roles/docker/tasks/main.yml
```
---
- name: Ensure Docker Desktop is installed
  command: docker --version
  register: docker_check
  ignore_errors: yes

- name: Fail if Docker is not installed
  fail:
    msg: "Docker Desktop is not installed. Please install it first."
  when: docker_check.rc != 0

- name: Check Docker is running
  command: docker info
  register: docker_running
  ignore_errors: yes

- name: Fail if Docker is not running
  fail:
    msg: "Docker Desktop is not running."
  when: docker_running.rc != 0
```

We don’t install Docker automatically here — on macOS that’s typically user-managed.

### 3️⃣ Role: Enable Kubernetes in Docker Desktop

Docker Desktop stores settings in a JSON file.
```
**Location:**

~/Library/Group Containers/group.com.docker/settings.json
```

roles/k8s/tasks/main.yml
```
---
- name: Enable Kubernetes in Docker Desktop settings
  replace:
    path: "{{ lookup('env','HOME') }}/Library/Group Containers/group.com.docker/settings.json"
    regexp: '"kubernetesEnabled": false'
    replace: '"kubernetesEnabled": true'
  notify: Restart Docker
```

**Handler**

roles/k8s/handlers/main.yml
```
---
- name: Restart Docker
  command: killall Docker
```

After restart, Docker will boot Kubernetes.

### 4️⃣ Wait for Kubernetes to Be Ready

Add to roles/k8s/tasks/main.yml:
```
- name: Wait for Kubernetes API
  command: kubectl cluster-info
  register: k8s_status
  retries: 10
  delay: 15
  until: k8s_status.rc == 0
```

This avoids race conditions.

### 5️⃣ Verify Cluster Nodes
```
- name: Get Kubernetes nodes
  command: kubectl get nodes
  register: nodes_output

- debug:
    var: nodes_output.stdout
```

**Expected output:**
- docker-desktop   Ready   control-plane   ...

### 6️⃣ Deploy Sample NGINX App

Now we prove it works.

roles/deploy/tasks/main.yml
```
---
- name: Deploy NGINX
  command: kubectl create deployment nginx --image=nginx
  ignore_errors: yes

- name: Expose NGINX service
  command: kubectl expose deployment nginx --type=NodePort --port=80
  ignore_errors: yes
```

### 7️⃣ Verify Deployment
```
- name: Get pods
  command: kubectl get pods
  register: pods

- debug:
    var: pods.stdout
```

#### 🚀 Run Everything
- ansible-playbook site.yml

Now you have:
- Docker validated
- Kubernetes enabled
- Cluster verified
- Sample workload deployed

Reproducible. Clean.

### 🔥 Advanced: Using Ansible Kubernetes Modules

Instead of shell commands, use native modules:
```
- name: Create namespace
  kubernetes.core.k8s:
    api_version: v1
    kind: Namespace
    name: demo
    state: present
```

**Install collection:**
- ansible-galaxy collection install kubernetes.core

This avoids raw kubectl usage.

#### 🧪 Example: Apply YAML Manifest

Create nginx.yml:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

#### Apply via Ansible:
```
- name: Apply manifest
  kubernetes.core.k8s:
    state: present
    src: nginx.yml
```

#### 🛠 Cleanup Playbook

Add cleanup.yml:
```
---
- hosts: localhost
  connection: local

  tasks:
    - name: Delete deployment
      command: kubectl delete deployment nginx
      ignore_errors: yes

    - name: Delete service
      command: kubectl delete service nginx
      ignore_errors: yes
```

#### ⚡ Optional: Using Kind Instead of Docker Desktop K8s

If you prefer lightweight clusters:
- Use Kind (Kubernetes in Docker).

**Ansible example:**
```
- name: Create kind cluster
  command: kind create cluster --name dev
```

Kind is often faster and more CI-friendly.
