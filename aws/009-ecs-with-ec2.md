# AWS ECS (Elastic Container Service) with EC2 Launch Type (Beginner's Guide)

When running a production backend application, deploying code directly to raw virtual machines (like bare EC2 instances) can lead to dependency conflicts, configuration drift, and scaling issues.

To solve this, modern development relies on **Containers (Docker)**. On AWS, **Amazon ECS (Elastic Container Service)** is the container orchestrator that runs, manages, and scales those containers across a cluster of servers.

---

> **Reference Material:** This guide is based on the article [AWS Part 6 — Step-by-Step Guide to Setting Up AWS ECS Using the EC2 Launch Type with Auto Scaling](https://medium.com/@shivambhadani_/aws-part-6-step-by-step-guide-to-setting-up-aws-ecs-using-the-ec2-launch-type-with-auto-scaling-cb21bf8e7a37) by Shivam Bhadani.

---

## 1. Docker & Container Basics (Layman's Terms)

### The Standardized Cargo Container Analogy

Imagine you are shipping items (toys, clothing, liquid oil) across the ocean on a cargo ship.

*   **Traditional Deployment (Loose Shipping):** You throw all the toys, boxes, and oil drums loose onto the deck. The oil leaks and ruins the toys. The crew struggles to stack different-shaped boxes, and unloading takes days because every box is unique.
*   **Containerized Deployment (Docker):** You pack the toys into a standard steel cargo container. You pack the oil drums into a separate liquid-proof cargo container. All cargo containers have the **exact same dimensions** and locking mechanisms.
    *   The cargo ship's crane easily stacks them.
    *   The items inside do not interfere with each other.
    *   It does not matter what is inside the container; the ship can carry it.

In software, **Docker** packages your code, its dependencies (`node_modules`), and the operating system configurations into a single, standardized **container**. This container runs identically on your local machine, staging servers, and AWS production servers.

---

## 2. What is Amazon ECR (Elastic Container Registry)?

Once you build your Docker image, where do you store it before running it on AWS? You store it in **Amazon ECR**.

**Amazon ECR** is a secure, private cloud registry where you upload (push) and store your Docker images. When AWS needs to run a new copy of your application, it connects to ECR and downloads (pulls) your image.

---

## 3. What is Amazon ECS (Elastic Container Service)?

### The Ship Captain Analogy

**Amazon ECS** is the **Ship Captain** or the Port Authority. Its job is to manage the containers: placing them on the ships, monitoring their health, and replacing them if they fall overboard.

AWS offers two ways (Launch Types) to run ECS:
1.  **Fargate (Serverless Launch Type):** You book a cruise ticket. AWS manages the ship, engine, staff, and routes. You only pay for your cabin. (Zero server management, but more expensive).
2.  **EC2 Launch Type:** You rent the physical cargo ship (EC2 instances). You are responsible for the fuel, engine maintenance, and steering. But you can pack as many containers onto the ship as can physically fit for a flat fee. (More control, cheaper for steady workloads).

*This guide focuses on the **EC2 Launch Type**.*

---

## 4. ECS Core Components

```
+-----------------------------------------------------------------+
|                        ECS CLUSTER                              |
|                       (Fleet of EC2s)                           |
|                                                                 |
|  +---------------------------+     +--------------------------+ |
|  |     ECS Task (Container)  |     |     ECS Task (Container)  | |
|  |  [ Web App Port 8080 ]    |     |  [ Web App Port 8081 ]    | |
|  +---------------------------+     +--------------------------+ |
|                                                                 |
|  +------------------------------------------------------------+ |
|  |                        ECS SERVICE                         | |
|  |  (Maintains 2 Tasks & Connects them to the Load Balancer)  | |
|  +------------------------------------------------------------+ |
+-----------------------------------------------------------------+
```

1.  **Task Definition (The Blueprint):** A JSON text file that describes how your container should run (Image URL, CPU/RAM, environment variables, port mappings).
2.  **Task (The Running Container):** A single active instance of your Task Definition.
3.  **Service (The Supervisor):** A manager that runs and maintains a specified number of Tasks (e.g. "Make sure 2 instances of the API are running"). If a task crashes, the service automatically replaces it.
4.  **Cluster (The Host Group):** A logical grouping of EC2 instances running the **ECS Container Agent**. The agent connects the EC2 servers to the ECS master coordinator.

---

## 5. Under the Hood: ECS Container Agent & Dynamic Port Mapping

*   **ECS Container Agent:** A lightweight daemon (running inside a Docker container itself) on each EC2 host instance. It communicates with the ECS API, reports host status, pulls images from ECR, starts/stops containers, and sends health statuses back to ECS.
*   **Dynamic Port Mapping:** If you run two copies of your Node app on a single EC2 server, they will both try to bind to Port 3000, causing port conflicts.
    *   By setting `hostPort: 0` in your Task Definition, ECS automatically assigns a **random high port** (e.g., `32768` and `32769`) on the EC2 host for each container.
    *   The ECS agent registers these random ports with the ALB Target Group, and the ALB dynamically routes public incoming traffic (Port 80) to these specific ports.

---

## 6. Task Placement: Strategies and Constraints

When launching tasks on an EC2 cluster, you can control *how* tasks are distributed across hosts:

### A. Placement Strategies (Cost vs. Redundancy)
*   **Binpack (Cost Optimization):** Place tasks on the host with the least remaining CPU or memory. This packs tasks as tightly as possible on a single EC2 instance, allowing you to shut down empty EC2 hosts to save money.
*   **Spread (High Availability):** Distribute tasks evenly across different Availability Zones or EC2 hosts. This ensures that if one AZ or EC2 server crashes, the other instances of your task are unaffected.
*   **Random:** Places tasks randomly across hosts.

### B. Placement Constraints
Hard rules that dictate host selection.
*   *Example:* `memberOf(attribute:ecs.instance-type == t3.medium)` ensures a task is only launched on a `t3.medium` EC2 host.

---

## 7. Step-by-Step Guide: Deploying ECS EC2 in the Console

### Step 1: Write a Dockerfile
Create a `Dockerfile` in your Node.js application root folder:
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD [ "npm", "start" ]
```

### Step 2: Push the Image to ECR
1. Go to **Elastic Container Registry (ECR)** in the AWS Console -> Click **Create repository** -> Set name `my-node-app` -> Click Create.
2. Click on the repository name, click **View push commands** in the top right, and execute the listed commands in your local terminal to build and push your image.
   ```bash
   aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
   docker build -t my-node-app .
   docker tag my-node-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-node-app:latest
   docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-node-app:latest
   ```

### Step 3: Create an ECS Task Definition
1. Go to **ECS Dashboard** -> Click **Task definitions** in the left menu -> **Create new task definition**.
2. Select **EC2** as the launch type compatibility. Click Next.
3. Configure details:
   * **Task definition family:** `my-web-task`
   * **Infrastructure requirements:** Set Task size (optional, can define at container level).
   * **Container details:**
     * **Name:** `web-container`
     * **Image URI:** Paste your ECR image URI (e.g. `123456789012.dkr.ecr.us-east-1.amazonaws.com/my-node-app:latest`).
     * **Port mappings:** Set *Container port* to `3000` and *Host port* to `0` (Enables Dynamic Port Mapping).
     * **Resource limits:** Set *Memory hard limit* to `512` MB and *CPU* to `256`.
4. Click **Create**.

### Step 4: Create the ECS Cluster and Capacity Provider
1. In the ECS Console, click **Clusters** -> **Create cluster**.
2. Set cluster parameters:
   * **Cluster name:** `my-ecs-cluster`
   * **Infrastructure:** Check **Amazon EC2 instances**.
     * **Operating system/Architecture:** Amazon Linux 2 (ECS-optimized).
     * **EC2 instance type:** `t2.micro` (or `t3.micro`).
     * **Desired capacity:** Minimum: `1`, Maximum: `3`.
     * **SSH Key Pair:** Select your key pair.
3. Click **Create**. This automatically creates an Auto Scaling Group behind the scenes and registers the EC2 instances with your cluster.

### Step 5: Deploy the ECS Service
1. Click into your new cluster `my-ecs-cluster`, and under the **Services** tab, click **Create**.
2. Configure settings:
   * **Environment:** Launch type: **EC2**.
   * **Deployment configuration:**
     * Task definition Family: `my-web-task` (select version).
     * Service name: `my-web-service`
     * Desired tasks: `2` (Runs two containers).
   * **Load balancing:** Select **Application Load Balancer**.
     * Select your ALB, and target group (registers tasks dynamically).
3. Click **Create**.

---

## 8. Capacity Providers: ECS Managed Auto Scaling

In production, scaling tasks requires scaling the host servers. **ECS Capacity Providers** automate this integration:
1. When ECS needs to scale out your service and run more tasks, it checks if the EC2 host instances have enough CPU/RAM capacity.
2. If host resources are exhausted, the Capacity Provider signals the underlying EC2 Auto Scaling Group to launch more EC2 instances.
3. Once the new EC2 hosts boot up and the ECS Agent registers, the Capacity Provider launches the pending ECS tasks onto the new hardware.

---

## 9. Troubleshooting: ECS EC2 Common Errors

### A. Tasks Stuck in `PENDING` Status
*   **Causes:**
    *   **No EC2 hosts available:** Your cluster has 0 registered container instances, or they are all stopped.
    *   **Resource Exhaustion:** Your active EC2 instances do not have enough unallocated CPU or memory to fit the new task limits.
*   **Fixes:**
    *   Check EC2 instances -> Launch more hosts or upgrade instance sizes.
    *   Reduce container CPU/Memory requirements inside the Task Definition.

### B. Tasks Crash and Restart (Status `Stopped`)
*   **Causes:**
    *   The container command failed.
    *   The application crashed due to missing environment variables or failed DB connections.
*   **Fix:** Check container logs. Go to ECS -> Tasks -> Click your stopped task -> Expand the container details -> Check the **Stopped Reason** or view the CloudWatch log stream.

---

## 10. ECS EC2 Production Checklist

- [x] **Configure Task Execution Role vs. Task Role:**
    *   **Task Execution Role:** Grants permissions to the ECS agent to pull the image from ECR and send logs to CloudWatch.
    *   **Task Role:** Grants permissions to the code running *inside* your container.
- [x] **Use Health Checks:** Define health checks inside the Task Definition to let ECS automatically restart crashed or deadlocked containers.
- [x] **Always Use Dynamic Port Mapping (`hostPort: 0`):** Maximizes EC2 host utilization by letting multiple containers share a single host.
- [x] **Set Resource Limits:** Always define hard and soft CPU/memory limits in the Task Definition to prevent a buggy container from hogging all the EC2 host resources (memory leaks).
- [x] **Configure Capacity Providers:** Set up ECS Capacity Providers to handle auto-scaling of the EC2 cluster host nodes automatically when task capacity runs low.
- [x] Use `spread` task placement strategies for production services to distribute containers across multiple Availability Zones.
- [x] Utilize ECR image tag immutability and scan-on-push to prevent container vulnerabilities.
