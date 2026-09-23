# Linux, AWS & Kubernetes — Revision Notes

Quick revision notes for Linux user management, Python environments, AWS networking, and Kubernetes.

---

# 1. Linux User Management

## Create a User with a Non-Interactive Shell

Sometimes we need a Linux user that **should not be allowed to log in interactively**.

```bash
sudo useradd -s /sbin/nologin anita
```

### Verify

```bash
grep '^anita:' /etc/passwd
```

Example:

```text
anita:x:1102:1102::/home/anita:/sbin/nologin
```

### Why `/sbin/nologin`?

It prevents the user from getting an interactive shell.

Commonly used for:

* Service accounts
* Application users
* Users that only need file/process ownership
* Security-restricted accounts

Mental model:

```text
Normal user
    |
    +--> /bin/bash
           |
           +--> Can interactively log in


Service user
    |
    +--> /sbin/nologin
           |
           +--> No interactive shell
```

---

# 2. UID and GID

Linux identifies users and groups using numbers.

Example `/etc/passwd` entry:

```text
ravi:x:1101:1101::/var/www/ravi:/bin/bash
```

The format is:

```text
username : password : UID : GID : comment : home : shell
```

Therefore:

```text
username = ravi
password = x
UID      = 1101
GID      = 1101
home     = /var/www/ravi
shell    = /bin/bash
```

## UID

```text
UID = User Identifier
```

Identifies the user.

Example:

```text
ravi → UID 1101
```

## GID

```text
GID = Group Identifier
```

Identifies the user's primary group.

Example:

```text
ravi → Primary GID 1101
```

---

# 3. Create a User with a Specific UID

```bash
sudo useradd -u 1101 -d /var/www/ravi -m ravi
```

This creates:

```text
Username       → ravi
UID            → 1101
Home directory → /var/www/ravi
```

### Important `useradd` Options

| Option | Meaning                      |
| ------ | ---------------------------- |
| `-u`   | Specify UID                  |
| `-d`   | Specify home directory       |
| `-m`   | Create home directory        |
| `-s`   | Specify login shell          |
| `-g`   | Specify primary group        |
| `-G`   | Specify supplementary groups |

### Example

```bash
sudo useradd \
  -u 1101 \
  -d /var/www/ravi \
  -m \
  -s /bin/bash \
  ravi
```

---

# 4. AWS Security Group — GUI Method

## Goal

Create a Security Group and allow:

```text
HTTP → Port 80
SSH  → Port 22
```

Instead of remembering the AWS CLI commands, use the AWS Console flow.

---

## Step 1 — Open EC2

AWS Console:

```text
AWS Console
   ↓
EC2
   ↓
Security Groups
```

---

## Step 2 — Create Security Group

Click:

```text
Create security group
```

Provide:

```text
Security group name:
nautilus-sg

Description:
Security group for Nautilus App Servers
```

Select the required VPC.

If using the default VPC:

```text
EC2
  ↓
Security Groups
  ↓
Create security group
  ↓
Select Default VPC
```

---

## Step 3 — Add HTTP Rule

Under **Inbound rules**, add:

```text
Type       → HTTP
Protocol   → TCP
Port       → 80
Source     → 0.0.0.0/0
```

Meaning:

```text
Internet
   |
   | TCP : 80
   ↓
EC2
```

This allows HTTP traffic from anywhere.

---

## Step 4 — Add SSH Rule

Add another inbound rule:

```text
Type       → SSH
Protocol   → TCP
Port       → 22
Source     → Your IP
```

### Important

For learning, you may see:

```text
0.0.0.0/0
```

but opening SSH to the entire internet is generally not recommended.

Prefer:

```text
My IP
```

when connecting from your own machine.

---

## Security Group Mental Model

Think of a Security Group as a **virtual firewall attached to an EC2 instance/network interface**.

```text
                    Internet
                       |
              +--------+--------+
              |                 |
          HTTP : 80         SSH : 22
              |                 |
              +--------+--------+
                       |
                Security Group
                       |
                      EC2
```

### Key Point

Security Groups are:

```text
STATEFUL
```

If an inbound request is allowed, the response traffic is automatically allowed.

---

# 5. Python Virtual Environment

## Why Virtual Environments?

Different Python applications may require different package versions.

Without a virtual environment:

```text
System Python
     |
     +-- numpy
     +-- pandas
     +-- sklearn
     +-- application A
     +-- application B
```

Version conflicts can occur.

With virtual environments:

```text
System Python
     |
     +-- ml-env
     |     +-- numpy
     |     +-- pandas
     |     +-- sklearn
     |
     +-- another-env
           +-- different package versions
```

---

## Create Virtual Environment

```bash
python3 -m venv ml-env
```

---

## Activate

```bash
source ml-env/bin/activate
```

You should see something similar to:

```text
(ml-env) user@server:~$
```

---

## Install Packages

```bash
pip install numpy pandas scikit-learn matplotlib
```

---

## Generate `requirements.txt`

```bash
pip freeze > requirements.txt
```

Example:

```text
numpy==...
pandas==...
scikit-learn==...
matplotlib==...
```

Another machine can install the same dependencies using:

```bash
pip install -r requirements.txt
```

---

# 6. Kubernetes — Creating a Pod

## Simple Pod

You can quickly create a Pod using:

```bash
kubectl run pod-nginx \
  --image=nginx:latest \
  --labels=app=nginx_app
```

This creates:

```text
Pod
 |
 +-- Container
       |
       +-- nginx:latest
```

---

# 7. Kubernetes Problem Encountered

## Task Requirement

Suppose the task says:

```text
Create a Pod

Pod name:
pod-nginx

Container name:
nginx-container

Image:
nginx:latest
```

You might run:

```bash
kubectl run pod-nginx \
  --image=nginx:latest \
  --labels=app=nginx_app
```

The Pod is created successfully.

But there is a problem.

The container name may become:

```text
pod-nginx
```

while the task specifically requires:

```text
nginx-container
```

---

# 8. Why Did This Happen?

`kubectl run` is designed for quickly creating a Pod.

It does not give you the same level of control over every Pod specification field as writing the YAML yourself.

So:

```text
kubectl run
    ↓
Quick Pod creation
    ↓
Less control
```

Whereas:

```text
YAML
    ↓
Explicit Pod specification
    ↓
More control
```

---

# 9. Correct Kubernetes YAML

Create:

```text
pod.yaml
```

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: pod-nginx
  labels:
    app: nginx_app

spec:
  containers:
    - name: nginx-container
      image: nginx:latest
```

Apply it:

```bash
kubectl apply -f pod.yaml
```

Now the specification explicitly says:

```text
Pod name       → pod-nginx
Container name → nginx-container
Image          → nginx:latest
```

---

# 10. Verify the Pod

Check Pods:

```bash
kubectl get pods
```

Check detailed information:

```bash
kubectl describe pod pod-nginx
```

Check the container name:

```bash
kubectl get pod pod-nginx \
  -o jsonpath='{.spec.containers[*].name}'
```

Expected:

```text
nginx-container
```

---

# 11. Important Kubernetes Lesson

### `kubectl run`

Useful for:

```text
Quick testing
Temporary Pods
Learning
Troubleshooting
```

Example:

```bash
kubectl run nginx --image=nginx
```

### YAML

Better when the task specifies exact configuration.

For example:

```text
Pod name
Container name
Labels
Ports
Environment variables
Volumes
Resources
Probes
Security settings
```

Mental model:

```text
kubectl run
     ↓
Quick & simple
     ↓
Less explicit control


YAML
     ↓
Declarative configuration
     ↓
Precise control
```

---

# 12. Key Takeaways

## AWS

* VPC spans multiple Availability Zones.
* A subnet belongs to one Availability Zone.
* An EC2 instance is launched into a subnet.
* Security Groups are **stateful**.
* NACLs are **stateless**.
* Secrets Manager uses KMS for encryption capabilities.
* CloudTrail records AWS API activity.
* CloudFront accelerates content delivery.
* Global Accelerator accelerates application/network traffic.
* Fargate provides serverless compute for containers.
* Versioning and delete markers are different concepts.

---

## Linux

* `-u` → Specify UID.
* `-d` → Specify home directory.
* `-m` → Create home directory.
* `-s` → Specify shell.
* `-g` → Primary group.
* `-G` → Supplementary groups.
* `/bin/bash` → Interactive shell.
* `/sbin/nologin` → Prevent interactive login.
* UID identifies a user.
* GID identifies a group.

---

## Kubernetes

* `kubectl run` creates a Pod.
* `kubectl create deployment` creates a Deployment.
* Pod and container are different concepts.
* A Pod can contain one or more containers.
* YAML gives explicit control over Kubernetes configuration.
* If a task specifies an exact **container name**, YAML is usually the safest approach.

---

# Quick Revision Cheatsheet

```text
LINUX
────────────────────────────────
-u → UID
-d → Home directory
-m → Create home directory
-s → Shell
-g → Primary group
-G → Supplementary group

/bin/bash      → Interactive shell
/sbin/nologin  → Non-interactive shell


AWS
────────────────────────────────
VPC            → Regional network
Subnet         → One AZ
Security Group → Stateful firewall
NACL           → Stateless firewall
CloudTrail     → API activity
CloudFront     → Content delivery
Global Accel.  → Application/network acceleration
Fargate        → Serverless containers


PYTHON
────────────────────────────────
python3 -m venv ml-env
source ml-env/bin/activate
pip install <package>
pip freeze > requirements.txt
pip install -r requirements.txt


KUBERNETES
────────────────────────────────
kubectl run              → Quick Pod
kubectl create deployment → Deployment
kubectl apply -f file.yaml → Apply YAML
kubectl get pods         → List Pods
kubectl describe pod     → Detailed Pod information

Exact container name?
        ↓
Prefer YAML
```
