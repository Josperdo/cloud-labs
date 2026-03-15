# AWS Lab 2: EC2, Auto Scaling & Load Balancing

## Overview
This lab mirrors Azure Lab 2 (compute) but on AWS. You'll deploy EC2 instances, configure Auto Scaling Groups, and set up a Load Balancer. This is where your infrastructure *actually does work*.

**Estimated time**: 3–4 hours  
**Cost**: ~$1–$2 (t3.micro/t2.micro are cheap; tear down at the end)  
**Key difference from Azure**: AWS has Auto Scaling Groups (manages fleet of instances) instead of VMSS. Both scale automatically, different APIs.

---

## Objectives
- Launch EC2 instances in public and private subnets
- Configure security groups for EC2
- Create Auto Scaling Group with scaling policies
- Set up Application Load Balancer
- Understand how instances communicate with load balancer
- Compare AWS compute to Azure VMs

---

## Part 1: Create Key Pair for SSH Access

### Step 1: Generate SSH Key Pair
1. Go to **AWS Console** → Search **EC2** → Left sidebar → **Key pairs** → **Create key pair**
2. **Key pair settings:**
   - **Name**: `lab2-key`
   - **Key pair type**: `RSA`
   - **File format**: `.pem` (for Linux/Mac/OpenSSH)
3. Click **Create key pair**
4. Your browser downloads `lab2-key.pem`
5. **Move this file somewhere safe** (you'll need it for SSH)

**Why this matters?** EC2 uses key pairs instead of passwords. The private key is your access credential.

---

## Part 2: Create Launch Template (Instead of Creating VMs One-by-One)

### Step 2: Create Launch Template
1. **EC2** → Left sidebar → **Launch templates** → **Create launch template**
2. **Launch template details:**
   - **Launch template name**: `lab2-template`
   - **Template version description**: `Web server template for Lab 2`
   - **Quick start**: Search `Ubuntu` → Select **Ubuntu Server 22.04 LTS** (free tier eligible)

3. **Instance type**: `t3.micro` (free tier; cheap for learning)

4. **Key pair login**: Select `lab2-key` (the one you created)

5. **Network settings**:
   - **Security groups**: Click **Create new security group**
     - **Security group name**: `lab2-web-sg`
     - **Description**: `Security group for web servers`
     - **VPC**: `lab1-vpc` (from Lab 1)
     - **Inbound rules**:
       - Type: `HTTP`, Port: `80`, Source: `0.0.0.0/0`
       - Type: `HTTPS`, Port: `443`, Source: `0.0.0.0/0`
       - Type: `SSH`, Port: `22`, Source: `MY_IP` (your IP)
     - Click **Create security group**

6. **Advanced details**:
   - Scroll down → **User data** (this runs on first boot)
   - Paste this script:
     ```bash
     #!/bin/bash
     apt-get update
     apt-get install -y nginx
     systemctl start nginx
     systemctl enable nginx
     echo "<h1>Welcome to $(hostname -f)</h1>" > /var/www/html/index.html
     ```
   - This installs Nginx and creates a simple "Hello" page

7. Click **Create launch template**

**What you've done**: Created a template that says "When I launch instances, use Ubuntu 22.04, t3.micro, this key pair, this security group, and run this script."

---

## Part 3: Create Auto Scaling Group

### Step 3: Create Auto Scaling Group (ASG)
1. **EC2** → Left sidebar → **Auto Scaling groups** → **Create Auto Scaling group**
2. **Choose launch template or configuration**:
   - **Launch template**: Select `lab2-template`
   - Click **Next**

3. **Choose instance launch options**:
   - **VPC**: `lab1-vpc`
   - **Subnets**: Select both `public-subnet-1` (and if you have `public-subnet-2`, select it too; for this lab, just `public-subnet-1`)
   - Click **Next**

4. **Configure advanced options**:
   - Check **Attach to a new load balancer**
   - **Load balancer type**: `Application Load Balancer`
   - **Load balancer name**: `lab2-alb`
   - **Listeners and routing**:
     - Port: `80`, Protocol: `HTTP`
     - Click **Create a target group**
       - **Target group name**: `lab2-tg`
       - **Protocol/Port**: `HTTP` / `80`
       - **VPC**: `lab1-vpc`
       - Click **Next** → **Create target group**
   - Back on ASG page, select the newly created `lab2-tg` as the target group
   - Click **Next**

5. **Configure group size and scaling policies**:
   - **Desired capacity**: `2` (start with 2 instances)
   - **Minimum capacity**: `1` (never go below 1)
   - **Maximum capacity**: `4` (never go above 4)
   - **Scaling policies**: 
     - Select **Target tracking scaling policy**
     - **Metric type**: `Average CPU Utilization`
     - **Target value**: `70` (scale up if CPU > 70%)
   - Click **Next**

6. **Add notifications** (optional):
   - Skip for this lab
   - Click **Next**

7. **Add tags**:
   - **Key**: `Name`, **Value**: `lab2-asg-instance`
   - Click **Next**

8. **Review** → Click **Create Auto Scaling group**

**Wait ~5 minutes** for instances to launch.

---

## Part 4: Verify Auto Scaling Group and Load Balancer

### Step 4: Check ASG Status
1. **Auto Scaling groups** → Click `lab2-asg`
2. You should see:
   - **Desired capacity**: 2
   - **Current instances**: 2 (they'll go from 0 running → 2 running over a few minutes)
   - **Instances** tab → Lists the 2 EC2 instances

3. Click on one of the instances → Note its **Public IP address**

### Step 5: Check Load Balancer
1. **EC2** → Left sidebar → **Load balancers** → Click `lab2-alb`
2. You'll see:
   - **DNS name**: Something like `lab2-alb-123456.us-east-1.elb.amazonaws.com`
   - **Target groups** → Click `lab2-tg`
     - Should show 2 instances with status "Healthy" (after ~30 seconds)

### Step 6: Test Load Balancer (Web Access)
1. Copy the **DNS name** of the load balancer
2. Open a web browser → Paste the DNS name (e.g., `http://lab2-alb-123456.us-east-1.elb.amazonaws.com`)
3. You should see the Nginx welcome page with the instance hostname

4. **Refresh the page a few times** → Watch the hostname change
   - This proves the load balancer is distributing traffic across instances

**What's happening**: 
- Load Balancer receives traffic on port 80
- Distributes it to instances in the target group
- Instances run Nginx (started by user data script)

### Step 7: Test SSH to Instance
1. Open terminal → SSH into one of the instances:
   ```bash
   ssh -i path/to/lab2-key.pem ubuntu@<PUBLIC_IP>
   ```
2. Once connected:
   ```bash
   curl http://localhost
   ```
   You should see the Nginx page HTML
3. Type `exit` to disconnect

---

## Part 5: Understand Auto Scaling in Action

### Step 8: Trigger Scaling Up (Optional, Advanced)
1. SSH into one of the instances (from Step 7)
2. Run a CPU-intensive command to spike CPU:
   ```bash
   stress --cpu 2 --timeout 300
   ```
   (This uses `stress` tool; if not installed, skip this)

3. While it's running, go back to AWS Console → **Auto Scaling groups** → `lab2-asg`
   - **Activity** tab → Watch for scaling activity
   - After ~2–3 minutes, you should see a new instance launching if CPU stays > 70%

4. Stop the stress command (Ctrl+C)

5. After ~10 minutes of low CPU, the ASG should scale back down to 2 instances

**Key concept**: Auto Scaling Groups automatically adjust instance count based on demand. This is how Netflix handles traffic spikes.

---

## Part 6: Compare AWS EC2 to Azure VMs

### Step 9: Mental Model (Interview Prep)

**If someone asks: "What's the difference between AWS Auto Scaling Groups and Azure Virtual Machine Scale Sets (VMSS)?"**

Your answer:
- **Both** automatically launch/terminate instances based on scaling policies
- **AWS ASG**: Policy-driven (CPU > 70% → add instance). Can be attached to load balancers.
- **Azure VMSS**: Similar policy-driven approach. Can be attached to load balancers.
- **Key difference**: ASG is separate from the load balancer (you attach ASG to LB's target group). VMSS is more integrated with Azure's ecosystem.

**Comparison table**:

| Aspect | AWS | Azure |
|--------|-----|-------|
| **Resource** | EC2 instance | Virtual Machine |
| **Fleet management** | Auto Scaling Group (ASG) | Virtual Machine Scale Set (VMSS) |
| **Load balancing** | Attach ASG to ALB/NLB target group | Attach VMSS to load balancer backend pool |
| **Scaling trigger** | Metrics (CPU, custom) + policies | Similar |
| **Key pair** | SSH key pair (.pem file) | SSH key or password |
| **Security** | Security Groups (at instance level) | NSGs (at subnet or NIC level) |

---

## Part 7: Understanding Application Load Balancer (ALB)

### Step 10: Deep Dive on Load Balancer

**If someone asks: "How does a load balancer decide which instance to send traffic to?"**

Your answer:
1. Client sends HTTP request to ALB's DNS name
2. ALB checks the target group (list of healthy instances)
3. ALB uses a routing algorithm (round-robin by default) to pick an instance
4. ALB forwards the request to that instance
5. Instance responds, ALB sends response back to client

**Health checks**: ALB periodically sends HTTP requests to instances (path `/`). If instance responds with 200 OK, it's "Healthy." If not, ALB removes it from the target group.

**Why this matters**: If one instance crashes, ALB automatically stops sending traffic to it. Users don't notice.

---

## Part 8: Cleanup

1. **Delete Auto Scaling Group**:
   - **Auto Scaling groups** → Select `lab2-asg` → **Delete**
   - (This terminates all instances in the ASG)

2. **Delete Load Balancer**:
   - **Load balancers** → Select `lab2-alb` → **Delete**

3. **Delete Target Group**:
   - **Target groups** → Select `lab2-tg` → **Delete**

4. **Delete Security Group**:
   - **Security groups** → Select `lab2-web-sg` → **Delete**

5. **Delete Launch Template** (optional; doesn't cost anything):
   - **Launch templates** → Select `lab2-template` → **Delete**

6. **Delete Key Pair** (optional):
   - **Key pairs** → Select `lab2-key` → **Delete**

**Wait 2–3 minutes** for all resources to terminate.

---

## Key Concepts to Understand

| Concept | What It Does |
|---------|-------------|
| **EC2** | Elastic Compute Cloud; virtual machine in AWS |
| **Launch Template** | Blueprint for launching instances (image, size, key pair, user data) |
| **Auto Scaling Group** | Manages a fleet of instances; launches/terminates based on policies |
| **Application Load Balancer (ALB)** | Distributes incoming traffic across instances; understands HTTP/HTTPS |
| **Target Group** | Collection of instances that receive traffic from a load balancer |
| **Health Check** | ALB periodically verifies instances are healthy; removes unhealthy ones |
| **Scaling Policy** | Rule that triggers scaling (e.g., CPU > 70% → add instance) |
| **User Data** | Script that runs once at instance startup |

---

## Interview Prep: Common Questions

**Q: "How would you deploy a web application across multiple instances with automatic failover?"**
A: Create a Launch Template with your application code (via user data script). Create an Auto Scaling Group with a minimum of 2 instances. Attach an Application Load Balancer with health checks. If one instance dies, ALB stops routing traffic to it and ASG launches a replacement.

**Q: "What's the difference between ALB and NLB?"**
A: ALB (Application Load Balancer) understands HTTP/HTTPS and can route based on URL paths or hostnames. NLB (Network Load Balancer) is ultra-high-performance, for extreme throughput/latency requirements. Use ALB for web apps; NLB for gaming or financial trading.

**Q: "How does the load balancer know if an instance is healthy?"**
A: ALB sends HTTP requests to a target path (e.g., `/`) at regular intervals (every 30 seconds by default). If it gets a 200 OK response, the instance is healthy. If it times out or gets 5XX errors, the instance is marked unhealthy and removed from rotation.

**Q: "Can you have multiple load balancers for the same application?"**
A: Yes. You might have one ALB for US traffic and another for EU traffic, each with their own target groups and Auto Scaling Groups.

---

## Next Steps
- Deploy a multi-AZ (availability zone) Auto Scaling Group for higher availability
- Add a CloudWatch alarm that triggers an SNS notification when CPU > 80%
- Create a custom health check (point to an endpoint that checks database connectivity)
- Set up HTTPS/SSL on the load balancer
- Use AWS Systems Manager to run commands on instances in the ASG
