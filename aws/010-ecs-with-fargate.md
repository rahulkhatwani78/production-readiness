# AWS ECS (Elastic Container Service) with AWS Fargate (Beginner's Guide)

Deploying containerized applications inside AWS using ECS offers two launch types: EC2 and Fargate. While the EC2 launch type requires you to manage the underlying virtual servers yourself, AWS Fargate provides a **serverless** option that takes care of the infrastructure entirely.

This guide explains how **AWS Fargate** simplifies container deployment, compares it to the EC2 model, and provides a step-by-step setup guide.

---

> **Reference Material:** This guide is based on the YouTube tutorial [Deploying Docker Containers on AWS Elastic Container Service (ECS)](https://youtu.be/AiiFbsAlLaI?si=9z0-EIgGmjEWhwCr) by Piyush Garg.

---

## 1. What is AWS Fargate? (Layman's Terms)

AWS Fargate is a **serverless compute engine for containers**. 

*   In traditional setups, you must manage the servers (EC2 instances) where your containers run, including patching the OS, managing server storage, and scaling up the number of instances.
*   With Fargate, **there are no servers to manage**. You simply package your application in a container, define how much CPU and RAM it needs, and Fargate automatically provisions the servers, runs the container, and manages the infrastructure.

### The Cruise Ticket vs. Private Yacht Analogy

Imagine you want to go on an ocean vacation with your friends.

*   **EC2 Launch Type (Renting a Private Yacht):**
    You rent a private boat. You are responsible for navigating, engine maintenance, buying fuel, and hiring a captain. You can crowd as many friends (containers) onto the boat as can fit for one flat price.
    *   *Drawback:* High effort. If the engine breaks down, your trip stops. If you have too many friends, you must buy a second yacht.
*   **AWS Fargate Launch Type (Booking a Cruise Ticket):**
    You buy individual tickets on a cruise liner. The cruise company (AWS) manages the ship, engine, staff, fuel, and route. You simply pack your suitcase (Docker container), show up, and go to your room.
    *   *Benefit:* Zero effort. You pay exactly for the rooms you book. If you add more friends, you just buy more tickets, and the cruise line handles room availability.

---

## 2. AWS EC2 vs. AWS Fargate

When designing container workloads on ECS, use this comparison table to decide which launch type fits your needs:

| Feature | ECS with EC2 Launch Type | ECS with AWS Fargate |
| :--- | :--- | :--- |
| **Server Management** | **Yes.** You manage, patch, and scale EC2 instances. | **No (Serverless).** AWS handles all server management. |
| **Pricing Model** | Pay for the EC2 servers and EBS volumes (flat rate). | Pay for the exact CPU and memory consumed per second. |
| **Scaling Speed** | Slower (must launch new EC2 hosts first). | Incredibly fast (launches containers directly). |
| **Networking Mode** | Supports bridge, host, none, and awsvpc modes. | **Requires awsvpc** networking mode only. |
| **Use Case** | Best for steady, predictable, high-volume workloads. | Best for bursty workloads, microservices, and fast scaling. |

---

## 3. Core Concepts of ECS Fargate

Fargate requires a few different networking and sizing rules compared to standard EC2 deployments:

### A. awsvpc Network Mode (ENI per Task)
Unlike EC2 launch type where multiple containers share a single server's network card, **Fargate requires `awsvpc` network mode**.
*   Every single Task (container) launched on Fargate receives its own **Elastic Network Interface (ENI)** and private IP address directly inside your VPC.
*   To the rest of your network, the container looks and acts like a standalone virtual machine.

### B. Security Groups at the Task Level
Because each Fargate Task has its own ENI and private IP, you do not assign security groups to the host EC2 server (since there are no servers). Instead, you attach a **Security Group directly to the Task itself**. You can create fine-grained rules restricting access to individual containers.

### C. Task-Level CPU and Memory
In Fargate, you do not specify size in terms of EC2 types (like `t2.micro`). You define the CPU and Memory limits directly at the **Task level** in your Task Definition (e.g., `0.25 vCPU` and `0.5 GB RAM`). AWS uses these limits to determine how much to charge you.

---

## 4. Step-by-Step Fargate Deployment Walkthrough

Here is the flow to build, push, and run your container on Fargate.

### Step 1: Package your App (Dockerfile)
Prepare your standard Dockerfile for a Node.js web application:
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD [ "node", "index.js" ]
```

### Step 2: Build and Push to ECR
```bash
# 1. Log in to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# 2. Build and Tag
docker build -t my-fargate-app .
docker tag my-fargate-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-fargate-app:latest

# 3. Push to Registry
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-fargate-app:latest
```

### Step 3: Write a Fargate Task Definition (`fargate-task.json`)
```json
{
  "family": "my-fargate-task",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "web-app",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-fargate-app:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "hostPort": 3000
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-fargate-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "web"
        }
      }
    }
  ],
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole"
}
```
*   `cpu` & `memory`: Declared at the root task level (required for Fargate).
*   `hostPort: 3000`: Under `awsvpc` network mode, the `hostPort` must match the `containerPort`.

Register the task:
```bash
aws ecs register-task-definition --cli-input-json file://fargate-task.json
```

### Step 4: Create the Fargate Service
Because Fargate runs inside the VPC directly, you must specify the subnet and security groups when creating the service:
```bash
aws ecs create-service \
    --cluster my-fargate-cluster \
    --service-name my-fargate-service \
    --task-definition my-fargate-task:1 \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[subnet-012345,subnet-abcde6],securityGroups=[sg-887766],assignPublicIp=ENABLED}"
```

---

## 5. Logging & Auto Scaling

### A. Logging with CloudWatch (`awslogs`)
Because there are no underlying servers to log into, you cannot use standard tools to inspect console logs (stderr/stdout).
*   You must configure the **`awslogs` log driver** in your Task Definition.
*   This automatically streams all `console.log()` statements from your Node.js application to **Amazon CloudWatch Logs** for easy debugging.

---

### B. Task Auto Scaling
With Fargate, scaling is simple:
*   You create an **Application Auto Scaling** policy for your ECS Service.
*   You set target metrics (e.g., scale out if average CPU usage goes above 70%).
*   Fargate automatically handles the provisioning of the new compute space and spins up your new containers in seconds. You do not need to scale any underlying server clusters.

---

## 6. Fargate Production Checklist

- [ ] **Deploy in Private Subnets:** Keep your Fargate tasks in private subnets. Use an **Application Load Balancer (ALB)** in a public subnet to receive user requests and forward them to your private tasks.
- [ ] **Enable CloudWatch Container Insights:** Provides deep metrics on container resource usage and memory diagnostics.
- [ ] **Define Log Groups First:** Before launching a service with `awslogs` config, ensure you create the target Log Group in CloudWatch, or the task will fail to launch.
- [ ] **Utilize Fargate Spot:** For non-critical, fault-tolerant workloads (e.g., worker queues, staging environments), configure Fargate Spot to save up to 70% of standard pricing.
- [ ] **Limit Task IAM Policies:** Follow the principle of least privilege. Grant your `Task Execution Role` permissions to read ECR, and give your `Task Role` only the permissions required to access specific databases or S3.
