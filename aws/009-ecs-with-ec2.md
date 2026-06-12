# AWS ECS (Elastic Container Service) with EC2 Launch Type (Beginner's Guide)

When running a production backend application, deploying code directly to raw virtual machines (like bare EC2 instances) can lead to dependency conflicts, configuration drift, and scaling issues.

To solve this, modern development relies on **Containers (Docker)**. On AWS, **Amazon ECS (Elastic Container Service)** is the container orchestrator that runs, manages, and scales those containers across a cluster of servers.

---

> **Reference Material:** This guide is based on the article [AWS Part 6 — Step-by-Step Guide to Setting Up AWS ECS Using the EC2 Launch Type with Auto Scaling](https://medium.com/@shivambhadani_/aws-part-6-step-by-step-guide-to-setting-up-aws-ecs-using-the-ec2-launch-type-with-auto-scaling-cb21bf8e7a37) by Shivam Bhadani.

---

## 1. Docker & Container Basics (Layman's Terms)

Before running containers on AWS, we must understand what they are.

### The Standardized Cargo Container Analogy

Imagine you are shipping items (toys, clothing, liquid oil) across the ocean on a cargo ship.

*   **Traditional Deployment (Loose Shipping):**
    You throw all the toys, boxes, and oil drums loose onto the deck. The oil leaks and ruins the toys. The crew struggles to stack different-shaped boxes, and unloading takes days because every box is unique.
*   **Containerized Deployment (Docker):**
    You pack the toys into a standard steel cargo container. You pack the oil drums into a separate liquid-proof cargo container. All cargo containers have the **exact same dimensions** and locking mechanisms.
    *   The cargo ship's crane easily stacks them.
    *   The items inside do not interfere with each other.
    *   It does not matter what is inside the container; the ship can carry it.

In software, **Docker** packages your code, its dependencies (`node_modules`), and the operating system configurations into a single, standardized **container**. This container runs identically on your local machine, staging servers, and AWS production servers.

---

## 2. What is Amazon ECR (Elastic Container Registry)?

### The Shipping Container Storage Yard Analogy

Once you build your steel cargo container (Docker image), where do you store it before loading it onto the ship?

You store it in a **Storage Yard (Amazon ECR)**. 

**Amazon ECR** is a secure, private cloud registry where you upload (push) and store your Docker images. When AWS needs to run a new copy of your application, it connects to ECR and downloads (pulls) your image.

---

## 3. What is Amazon ECS (Elastic Container Service)?

### The Ship Captain Analogy

**Amazon ECS** is the **Ship Captain** or the Port Authority. Its job is to manage the containers: placing them on the ships, monitoring their health, and replacing them if they fall overboard.

AWS offers two ways (Launch Types) to run ECS:

1.  **Fargate (Serverless Launch Type):**
    You ask a third-party logistics company to ship your container. You don't care about the ship's engine, maintenance, or size. You only pay for the exact weight of your container. (Zero server management, but more expensive).
2.  **EC2 Launch Type:**
    You rent the physical cargo ship (EC2 instances). You are responsible for the fuel, engine maintenance, and steering. But you can pack as many containers onto the ship as can physically fit for a flat fee. (More control, cheaper for steady workloads).

*This guide focuses on the **EC2 Launch Type**.*

---

## 4. ECS Core Components

Inside ECS, you organize your container deployments using four core objects:

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

1.  **Task Definition (The Blueprint):** A JSON text file that describes how your container should run. It lists:
    *   Which Docker image to pull from ECR.
    *   How much CPU and memory (RAM) to allocate.
    *   Which ports to open.
    *   What environment variables to inject.
    *(Analogy: The shipping manifest listing instructions for a container).*
2.  **Task (The Running Container):** A single active instance of your Task Definition. (Analogy: The physical cargo container box sitting on the ship).
3.  **Service (The Supervisor):** A manager that runs and maintains a specified number of Tasks (e.g., *"Make sure 3 copies of our web app are always running"*). If a Task crashes, the Service automatically terminates it and launches a new one.
4.  **Cluster (The Host Group):** A logical grouping of EC2 instances running the ECS container agent. The agent connects your EC2 servers to the ECS service. (Analogy: The fleet of cargo ships).

---

## 5. Step-by-Step Deployment Walkthrough

Here is how you package and deploy a Node.js application to ECS using the EC2 Launch Type.

### Step 1: Write a Dockerfile
Create a file named `Dockerfile` in your Node.js application root folder:
```dockerfile
# 1. Use the official Node.js runtime image
FROM node:18-alpine

# 2. Set the working directory inside the container
WORKDIR /usr/src/app

# 3. Copy package files and install dependencies
COPY package*.json ./
RUN npm install --production

# 4. Copy the rest of the application code
COPY . .

# 5. Expose the port the app runs on
EXPOSE 3000

# 6. Command to start the application
CMD [ "npm", "start" ]
```

### Step 2: Push the Image to Amazon ECR
Create an ECR repository and run these commands to push your image:
```bash
# 1. Authenticate Docker with your ECR registry
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# 2. Build the Docker image locally
docker build -t my-node-app .

# 3. Tag your image to point to ECR
docker tag my-node-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-node-app:latest

# 4. Push the image to ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-node-app:latest
```

### Step 3: Create a Task Definition (`task-def.json`)
Create a JSON file describing the container setup:
```json
{
  "family": "my-web-task",
  "containerDefinitions": [
    {
      "name": "web-container",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-node-app:latest",
      "cpu": 256,
      "memory": 512,
      "portMappings": [
        {
          "containerPort": 3000,
          "hostPort": 0
        }
      ]
    }
  ],
  "requiresCompatibilities": ["EC2"],
  "networkMode": "bridge"
}
```
*   `"hostPort": 0`: This is a crucial setting for **Dynamic Port Mapping** (explained below).

Register this definition:
```bash
aws ecs register-task-definition --cli-input-json file://task-def.json
```

### Step 4: Deploy the ECS Service
Create an ECS Service linked to your Application Load Balancer (ALB) and target group:
```bash
aws ecs create-service \
    --cluster my-ecs-cluster \
    --service-name my-web-service \
    --task-definition my-web-task:1 \
    --desired-count 2 \
    --launch-type EC2 \
    --load-balancers targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/ecs-tg/xyz,containerName=web-container,containerPort=3000
```

---

## 6. Dynamic Port Mapping & Auto Scaling

### A. Dynamic Port Mapping
If you run two copies of your web app on a single EC2 server, they will both try to bind to Port 3000. This will crash the second container (port already in use).

By setting `hostPort: 0` in your Task Definition:
1.  ECS automatically assigns a **random high port** (e.g., `32768` and `32769`) on the EC2 host for each container.
2.  The container agent notifies the ALB Target Group: *"Container A is listening on host port 32768, and Container B is on 32769."*
3.  The ALB dynamically maps incoming public traffic (Port 80) to these random ports. 
4.  This allows you to pack multiple instances of the same application on a single EC2 host without port conflicts.

---

### B. Auto Scaling in ECS
To scale a production ECS cluster, you must configure two separate scaling loops:
*   **Task Auto Scaling (Scale containers):** If memory or CPU on your containers hits 70%, ECS Task Auto Scaling adds more Tasks (containers).
*   **EC2 Auto Scaling (Scale hosts):** If your EC2 instances run out of space to host new containers, ECS **Capacity Providers** automatically trigger the underlying EC2 Auto Scaling Group to launch another EC2 server so ECS has new hardware to place containers on.

---

## 7. ECS Production Checklist

- [ ] **Configure Task Execution Role vs. Task Role:**
    *   **Task Execution Role:** Grants permissions to the ECS agent to pull the image from ECR and send logs to CloudWatch.
    *   **Task Role:** Grants permissions to the code running *inside* your container (e.g., letting your Node.js app read from an S3 bucket).
- [ ] **Use Health Checks:** Define health checks inside the Task Definition to let ECS automatically restart crashed or deadlocked containers.
- [ ] **Always Use Dynamic Port Mapping (`hostPort: 0`):** Maximizes EC2 host utilization by letting multiple containers share a single host.
- [ ] **Set Resource Limits:** Always define hard and soft CPU/memory limits in the Task Definition to prevent a buggy container from hogging all the EC2 host resources (memory leaks).
- [ ] **Configure Capacity Providers:** Set up ECS Capacity Providers to handle auto-scaling of the EC2 cluster host nodes automatically when task capacity runs low.
