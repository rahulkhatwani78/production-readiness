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

*   **EC2 Launch Type (Renting a Private Yacht):** You rent a private boat. You are responsible for navigating, engine maintenance, buying fuel, and hiring a captain. You can crowd as many friends (containers) onto the boat as can fit for one flat price.
    *   *Drawback:* High effort. If the engine breaks down, your trip stops. If you have too many friends, you must buy a second yacht.
*   **AWS Fargate Launch Type (Booking a Cruise Ticket):** You buy individual tickets on a cruise liner. The cruise company (AWS) manages the ship, engine, staff, fuel, and route. You simply pack your suitcase (Docker container), show up, and go to your room.
    *   *Benefit:* Zero effort. You pay exactly for the rooms you book. If you add more friends, you just buy more tickets, and the cruise line handles room availability.

---

## 2. AWS EC2 vs. AWS Fargate

When designing container workloads on ECS, use this comparison table to decide which launch type fits your needs:

| Feature | ECS with EC2 Launch Type | ECS with AWS Fargate |
| :--- | :--- | :--- |
| **Server Management** | **Yes.** You manage, patch, and scale EC2 instances. | **No (Serverless).** AWS handles all server management. |
| **Pricing Model** | Pay for the EC2 servers and EBS volumes (flat rate). | Pay for the exact CPU and memory consumed per second. |
| **Scaling Speed** | Slower (must launch new EC2 hosts first). | **Incredibly fast (launches containers directly).** |
| **Networking Mode** | Supports bridge, host, none, and awsvpc modes. | **Requires awsvpc** networking mode only. |
| **Storage Limits** | Up to physical hard drive space on EC2. | Ephemeral storage up to 200 GB. |
| **Use Case** | Best for steady, predictable, high-volume workloads. | Best for microservices, APIs, and fast scaling. |

---

## 3. ECS Fargate Core Concepts

Fargate introduces specific networking and resource allocation rules:

### A. awsvpc Network Mode (ENI per Task)
Unlike the EC2 launch type where multiple containers share a single server's network card, **Fargate requires `awsvpc` network mode**.
*   Every single Task (container) launched on Fargate receives its own **Elastic Network Interface (ENI)** and private IP address directly inside your VPC.
*   The container behaves exactly like a standalone virtual machine inside your private network.

### B. Security Groups at the Task Level
Because each Fargate Task has its own ENI and private IP, you do not assign security groups to any host server. Instead, you attach a **Security Group directly to the Task itself**, letting you define precise inbound/outbound rules for individual containers.

### C. Task-Level CPU and Memory Settings
You must select matching allocations of CPU and RAM for your tasks. The pricing is billed based on these selected resource footprints:

| Task vCPU | Memory Configurations Allowed |
| :--- | :--- |
| **0.25 vCPU** | 0.5 GB, 1 GB, 2 GB |
| **0.5 vCPU** | 1 GB, 2 GB, 3 GB, 4 GB |
| **1.0 vCPU** | 2 GB to 8 GB (1 GB increments) |
| **2.0 vCPU** | 4 GB to 16 GB (1 GB increments) |
| **4.0 vCPU** | 8 GB to 30 GB (1 GB increments) |

---

## 4. Fargate Spot: Up to 70% Cost Savings

For non-critical, fault-tolerant workloads (e.g. staging environments, test suites, background queue workers), you can utilize **Fargate Spot**:
*   Fargate Spot runs your containers on spare AWS capacity at a **60-70% discount** compared to standard Fargate pricing.
*   *Catch:* If AWS needs the capacity back, your containers will be terminated. AWS sends a **2-minute warning notification** (via Amazon EventBridge) before shutting down the container.
*   *Best Practice:* Run a mix of standard Fargate (e.g. minimum 1-2 tasks for baseline reliability) and Fargate Spot (for scaling tasks).

---

## 5. Step-by-Step Guide: Deploying Fargate in the Console

### Step 1: Create a Task Definition
1. Open the **AWS Console** -> Go to **ECS Dashboard**.
2. Click **Task definitions** in the left menu -> Click **Create new task definition**.
3. Configure the settings:
   * **Task definition family:** `fargate-web-task`
   * **Launch type:** Select **AWS Fargate** (default).
   * **Task size:** Set *CPU* to `0.25 vCPU` and *Memory* to `0.5 GB`.
   * **Task roles:**
     * **Task role:** Select an IAM role allowing your code to access S3/DynamoDB (if needed).
     * **Task execution role:** Select `ecsTaskExecutionRole` (Allows ECS to pull ECR images and create CloudWatch logs).
   * **Container details:**
     * **Name:** `fargate-container`
     * **Image URI:** Enter your ECR image URL.
     * **Port mappings:** Set *Container port* to `3000` and *Host port* to `3000` (`awsvpc` mode requires host and container ports to match).
     * **Log configuration:** Keep enabled. This configures the `awslogs` driver to stream output to **CloudWatch Logs** automatically.
4. Click **Create**.

### Step 2: Create a Fargate Cluster
1. In the ECS Console, click **Clusters** -> **Create cluster**.
2. Set cluster parameters:
   * **Cluster name:** `fargate-cluster`
   * **Infrastructure:** Keep default **AWS Fargate (serverless)** checked.
3. Click **Create**.

### Step 3: Create the Fargate Service
1. Click into your new cluster `fargate-cluster`. Under the **Services** tab, click **Create**.
2. Configure settings:
   * **Environment:** Launch type: **FARGATE** (select version `LATEST`).
   * **Deployment configuration:**
     * Task definition Family: `fargate-web-task` (select version).
     * Service name: `fargate-service`
     * Desired tasks: `2`.
   * **Networking:**
     * **VPC:** Select your VPC.
     * **Subnets:** Select your **private subnets** (Best practice).
     * **Security group:** Create a new group allowing inbound traffic on Port `3000` from your Load Balancer's security group.
     * **Public IP:** Keep **Enabled** if running in public subnets, or **Disabled** if running inside private subnets (NAT Gateway handles outbound traffic).
   * **Load balancing:** Select **Application Load Balancer**.
     * Select your ALB, and target group (registers containers dynamically).
3. Click **Create**.

---

## 6. Step-by-Step Guide: Managing Fargate via AWS CLI

### A. Register Fargate Task Definition (`fargate-task.json`)
```json
{
  "family": "fargate-web-task",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "fargate-app",
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
          "awslogs-group": "/ecs/fargate-app-logs",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "web"
        }
      }
    }
  ],
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole"
}
```
Run register command:
```bash
aws ecs register-task-definition --cli-input-json file://fargate-task.json
```

### B. Create Fargate Service
```bash
aws ecs create-service \
    --cluster fargate-cluster \
    --service-name fargate-service \
    --task-definition fargate-web-task:1 \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[subnet-012345,subnet-abcde6],securityGroups=[sg-887766],assignPublicIp=DISABLED}"
```

---

## 7. Troubleshooting: Fargate Startup Failures

### A. Error: `CannotPullContainerError: API error... Connection timed out`
*   **Cause:** Your Fargate tasks are deployed in a private subnet, but the subnet route table is missing a route to a NAT Gateway, preventing the container agent from connecting to ECR to download the Docker image.
*   **Fixes:**
    *   Add a route in the private subnet route table pointing `0.0.0.0/0` to a NAT Gateway in a public subnet.
    *   Or, configure **VPC Endpoints (AWS PrivateLink)** for ECR inside your VPC so ECR images are downloaded locally inside the private network.

### B. Error: `Essential container in task exited`
*   **Cause:** The application boot process crashed, or the CMD script exited immediately.
*   **Fix:**
    1. Ensure the CloudWatch Log Group `/ecs/fargate-app-logs` is created in CloudWatch. (If the Log Group does not exist, Fargate fails to start).
    2. Open the CloudWatch console, check the log stream for the container, and debug application-level startup exceptions (e.g. database connection failures, syntax errors).

---

## 8. Fargate Production Checklist

- [x] **Deploy in Private Subnets:** Keep your Fargate tasks in private subnets. Use an **Application Load Balancer (ALB)** in a public subnet to receive user requests and forward them to your private tasks.
- [x] **Enable CloudWatch Container Insights:** Provides deep metrics on container resource usage and memory diagnostics.
- [x] **Define Log Groups First:** Before launching a service with `awslogs` config, ensure you create the target Log Group in CloudWatch, or the task will fail to launch.
- [x] **Utilize Fargate Spot:** For non-critical, fault-tolerant workloads, configure Fargate Spot to save up to 70% of standard pricing.
- [x] **Limit Task IAM Policies:** Follow the principle of least privilege. Grant your `Task Execution Role` permissions to read ECR, and give your `Task Role` only the permissions required to access specific databases or S3.
- [x] Configure ephemeral storage size up to 200 GB in the Task Definition if your application requires caching large local data sets.
- [x] Set up auto-scaling targets based on memory utilization to auto-scale tasks during memory-intensive requests.
