# local-devops-lab
Zero cost — No surprise bills at the end of the month, Works offline, No security worries and Total control
---
# What You’ll Need:
1. A laptop with at least 8GB of RAM (16GB is better)
2. About 20GB of free disk space
3. Windows, macOS, or Linux operating system
4. Administrator access to your computer
5. Virtualization enabled in BIOS/UEFI
---
## Step 1: Setting Up Docker
### For Linux:
**Update your package lists**
- sudo apt-get update
  
**Install prerequisites**
- sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
  
**Add Docker's official GPG key**
- curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  
**Add the Docker repository**
- sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  
**Update package database with Docker packages**
- sudo apt-get update
  
**Install Docker**
- sudo apt-get install docker-ce
  
**Start and enable Docker**
- sudo systemctl start docker
- sudo systemctl enable docker
  
**Add your user to the docker group so you don't need sudo**
- sudo usermod -aG docker $USER

### For Mac:
- Download Docker Desktop for Mac from Docker’s [website](https://www.docker.com/products/docker-desktop)
- Open the downloaded .dmg file and drag Docker to your Applications folder
- Open Docker from your Applications folder
- Enter your password when prompted
- Wait until the whale icon in the menu bar stops animating
---
### Test Docker:
**Open a terminal or command prompt and type:**
- docker run hello-world
---
## Step 2: Setting Up Kubernetes 
- with [minikube](https://minikube.sigs.k8s.io)

### For Linux:
**Download Minikube**
- curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64# Make it executable
- chmod +x minikube-linux-amd64
  
**Move it to your path**
- sudo mv minikube-linux-amd64 /usr/local/bin/minikube

**Start Minikube**
- minikube start --driver=docker

### For Mac:
**Using Homebrew**
- brew install minikube
  
**Start Minikube**
- minikube start --driver=docker

### Testing Minikube:
**Check if your cluster is running**
- minikube status
  
**Open the Kubernetes dashboard in your browser**
- minikube dashboard --url
- minikube dashboard

- with [Kind](https://kind.sigs.k8s.io)

### For Linux:
**Download Kind**
- curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.11.1/kind-$(uname)-amd64
  
**Make it executable**
- chmod +x ./kind
  
**Move it to your path**
- sudo mv ./kind /usr/local/bin/kind

### For Mac:
- brew install kind

### Creating a cluster with Kind:
**Create a new cluster**
- kind create cluster --name my-local-lab
---
## Step 3: Installing kubectl 

### For Linux:
**Using curl**
- curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
- chmod +x kubectl
- sudo mv kubectl /usr/local/bin/kubectl

### For Mac:
**Using Homebrew**
- brew install kubectl
  
**Or using curl**
- curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
- chmod +x ./kubectl
- sudo mv ./kubectl /usr/local/bin/kubectl

### Testing kubectl:
**List all pods in all namespaces**
- kubectl get pods --all-namespaces

---
## Step 4: Installing Ansible
Ansible helps you automate tasks across multiple servers.

### For Linux:
**For Ubuntu/Debian**
- sudo apt update
- sudo apt install software-properties-common
- sudo apt-add-repository --yes --update ppa:ansible/ansible
- sudo apt install ansible

**For Red Hat/CentOS**
- sudo yum install ansible

### For Mac:
**Using Homebrew**
- brew install ansible

### Testing Ansible:
1. Open /etc/ansible in your editor and create a file called hosts with:

[local]
localhost ansible_connection=local

2. Create a file called playbook.yml :

```
- name: Test playbook
  hosts: local
  tasks:
    - name: Print a message
      debug:
        msg: "Ansible is working!"
```

3. Run it with:
 - ansible-playbook -i hosts playbook.yml

---
## Step 5: Installing Terraform

### For Linux:
**Download**
- wget https://releases.hashicorp.com/terraform/1.0.0/terraform_1.0.0_linux_amd64.zip

**Unzip**
- unzip terraform_1.0.0_linux_amd64.zip

**Move to a directory in your PATH**
- sudo mv terraform /usr/local/bin/

### For Mac:
**Using Homebrew**
- brew install terraform

### Testing Terraform:

1. Create a file called main.tf with:

```
resource "local_file" "hello" {
  content  = "Hello, Terraform!"
  filename = "${path.module}/hello.txt"
}
```

2. Run:

```
terraform init
terraform apply
```

---

## Using GitHub Actions

1. Create file: .github/workflows/ansible.yml
   
```
name: Ansible (local)

on:
  push:
  workflow_dispatch:

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install Ansible
        run: |
          python -m pip install --upgrade pip
          pip install ansible

      - name: Run playbook
        run: |
          ansible-playbook -i hosts playbook.yml
```

2. Workflow example (SSH to hosts)
   
```
name: Ansible (remote)

on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install Ansible
        run: |
          python -m pip install --upgrade pip
          pip install ansible

      - name: Load SSH key
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Run playbook
        env:
          ANSIBLE_HOST_KEY_CHECKING: "False"
        run: |
          ansible-playbook -i hosts playbook.yml
```

set ANSIBLE_HOST_KEY_CHECKING=False because CI runners are ephemeral; better is to manage known_hosts

Github Actions [marketplace](https://github.com/marketplace/actions/run-ansible-playbook)
---

**Source:**  https://open.substack.com/pub/stackinsight/p/package-a-compose-k8s-loki-stack

