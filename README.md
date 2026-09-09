# ☁️ CloudRik — Enterprise Serverless PaaS Platform

## 📌 Project Overview
CloudRik is a highly scalable, AWS-native **Platform-as-a-Service (PaaS)** designed to automate the deployment, scaling, and management of modern web applications. The platform is built as a custom alternative to services such as Vercel and Netlify, providing automated deployments, custom domain routing, serverless backend execution, and CI/CD capabilities.

This project focuses on the **Cloud Architecture & DevOps Infrastructure** powering CloudRik. The infrastructure follows a decoupled and stateless compute model using **Amazon EC2, Auto Scaling Groups, Application Load Balancer, Amazon EFS, Amazon S3, AWS CodeBuild, AWS Lambda, PostgreSQL, Redis, Nginx, and Amazon CloudWatch**.

The architecture separates compute, storage, database, build, and user-workload execution responsibilities so that application traffic and user deployments can be handled independently. The documented implementation is based on the CloudRik infrastructure documentation provided for this project.

---

## 🏗️ Architecture Diagram
*(Paste your CloudRik network and infrastructure architecture diagram here in the GitHub repo)*

![CloudRik AWS Architecture Diagram](./architecture-diagram.png)

---

## 🛠️ Infrastructure Components & Details

### 1️⃣ Compute & Edge Routing Layer

| Component | Implementation | Purpose |
| :--- | :--- | :--- |
| **Amazon EC2** | API + Nginx Edge Proxy | Runs the core CloudRik application and reverse-proxy layer |
| **Auto Scaling Group** | `cloudrik-api-asg` | Automatically manages API/edge compute capacity |
| **Golden AMI** | `cloudrik-api-golden-ami` | Provides a standardized machine image for new ASG instances |
| **Launch Template** | ASG Launch Template | Defines the configuration used when new compute nodes are launched |
| **Application Load Balancer** | ALB | Distributes incoming requests across available ASG nodes |
| **Nginx** | Edge Reverse Proxy | Handles SSL termination and dynamic routing for hosted domains |

### 2️⃣ Stateful Database & Caching Layer

| Component | Implementation | Purpose |
| :--- | :--- | :--- |
| **PostgreSQL** | `zenith_db` | Stores persistent application and deployment metadata |
| **Redis** | Dedicated Redis service | Provides caching and pub/sub functionality |
| **Dedicated EC2 Node** | Private subnet | Hosts PostgreSQL and Redis separately from the stateless compute layer |
| **Security Group** | Internal VPC access | Restricts database and cache connectivity to authorized internal compute resources |

### 3️⃣ Shared Network Storage

| Component | Implementation | Purpose |
| :--- | :--- | :--- |
| **Amazon EFS** | Mounted at `/mnt/efs` | Provides shared persistent filesystem access |
| **Nginx Configuration** | Stored on EFS | Keeps routing configuration available across ASG nodes |
| **Let's Encrypt Certificates** | Stored on EFS | Makes dynamically generated SSL certificates available across the cluster |

### 4️⃣ CI/CD & Build Engine

| Component | Implementation | Purpose |
| :--- | :--- | :--- |
| **GitHub Webhooks** | Push-triggered workflow | Notifies the CloudRik API when source code changes |
| **AWS CodeBuild** | Programmatically provisioned builds | Compiles user applications in isolated build environments |
| **Amazon S3** | Build artifact destination | Stores compiled frontend deployment artifacts |

### 5️⃣ Serverless Compute Layer

| Component | Implementation | Purpose |
| :--- | :--- | :--- |
| **AWS Lambda** | `cloudrik-fn-*` functions | Executes user backend workloads independently from core platform servers |
| **AWS SDK** | Node.js `@aws-sdk` v3 | Programmatically provisions and manages user functions |
| **Lambda Function URLs** | Direct function endpoint | Provides an endpoint that the Nginx proxy can route API requests to |
| **Amazon CloudWatch** | Execution log source | Collects Lambda execution logs for platform monitoring |

### 6️⃣ Static Hosting & Delivery Layer

| Component | Implementation | Purpose |
| :--- | :--- | :--- |
| **Amazon S3** | Frontend artifact storage | Stores compiled React, Vue, and static Next.js applications |
| **Nginx** | Dynamic reverse proxy | Intercepts hosted-domain requests and retrieves static assets from S3 |
| **Cloudflare** | DNS/edge integration | Forms part of the external domain and traffic-management layer |

---

## 🚀 Deep-Dive Implementation Steps

### 🔹 Step 1: Design the Decoupled AWS Cloud Architecture
The CloudRik infrastructure is designed around separation of responsibilities instead of placing the complete application stack on a single server:
*   The **stateless application and Nginx edge layer** run on EC2 instances managed by an Auto Scaling Group.
*   The **stateful PostgreSQL and Redis services** are separated onto a dedicated EC2 node.
*   **Amazon EFS** provides shared filesystem state required by multiple Nginx nodes.
*   **AWS CodeBuild** handles resource-intensive application compilation outside the core API servers.
*   **AWS Lambda** executes dynamic user backend workloads independently from the platform's core compute layer.
*   **Amazon S3** stores compiled frontend deployment artifacts.

### 🔹 Step 2: Provision the Stateless Compute Layer with EC2 & Auto Scaling
The CloudRik API and Nginx Edge Proxy are deployed using an EC2-based Auto Scaling architecture. A Golden AMI and Launch Template provide a repeatable configuration for dynamically created nodes:
*   Configured the Auto Scaling Group as `cloudrik-api-asg`.
*   Created the Golden AMI as `cloudrik-api-golden-ami`.
*   Used a Launch Template to define the configuration for new compute instances.
*   Hosted the CloudRik API and Nginx Edge Proxy on the ASG-managed EC2 instances.
*   Designed the compute layer to remain stateless so additional nodes can be introduced without moving application state between servers.

### 🔹 Step 3: Introduce the Application Load Balancer
The Application Load Balancer provides a single ingress point for application traffic and distributes requests across the available EC2 compute nodes:
*   Positioned the **ALB** in front of the Auto Scaling Group.
*   Distributed incoming requests across the available CloudRik compute instances.
*   Used the load-balanced architecture to reduce dependence on any individual application server.
*   Routed traffic from the load-balancing layer toward the Nginx/application tier.

### 🔹 Step 4: Configure Nginx as the CloudRik Edge Proxy
Nginx acts as the central routing layer for hosted projects and custom domains. It terminates SSL and dynamically determines where incoming application traffic must be forwarded:
*   Configured Nginx as the Edge Reverse Proxy for the CloudRik platform.
*   Implemented wildcard subdomain routing for `*.cloudrik.com`.
*   Added support for custom-domain routing.
*   Configured routing paths for static frontend assets hosted in **Amazon S3**.
*   Configured `/api/*` traffic to be proxied toward **AWS Lambda Function URLs**.
*   Used Nginx as the point where dynamically managed SSL certificates and routing configurations are applied.

### 🔹 Step 5: Isolate PostgreSQL & Redis from the Stateless Compute Layer
Persistent state is separated from the dynamically scalable API/edge layer so that scaling application nodes does not require moving database or cache state:
*   Hosted PostgreSQL database `zenith_db` on a dedicated EC2 node.
*   Hosted Redis on the same dedicated stateful node.
*   Placed the database node inside a private subnet.
*   Restricted PostgreSQL traffic on port `5432` to authorized internal VPC sources.
*   Restricted Redis traffic on port `6379` to authorized internal VPC sources.
*   Prevented direct public exposure of the stateful database and cache services.

### 🔹 Step 6: Implement Shared Configuration Storage with Amazon EFS
Auto-scaled Nginx instances require access to the same routing rules and SSL certificate state. Amazon EFS provides a shared filesystem that can be mounted by multiple compute nodes:
*   Mounted the EFS filesystem at `/mnt/efs` on ASG instances.
*   Stored Nginx configuration files on the shared filesystem.
*   Stored Let's Encrypt SSL certificates on the shared filesystem.
*   When a new custom domain is mapped, the primary node updates the shared EFS state.
*   Made updated Nginx configuration and certificate files available to other running ASG instances through the shared filesystem.

### 🔹 Step 7: Automate GitHub-to-CodeBuild Application Builds
CloudRik moves resource-intensive compilation away from the main API servers and into isolated AWS CodeBuild environments:
*   Configured the deployment workflow around GitHub push events.
*   The GitHub webhook triggers the CloudRik API.
*   The CloudRik API programmatically provisions an AWS CodeBuild build environment.
*   CodeBuild clones the user's repository inside an isolated build environment.
*   Secure environment variables are injected into the build process.
*   Application build commands such as `npm run build` are executed inside CodeBuild.
*   Generated build artifacts are synchronized to Amazon S3.

### 🔹 Step 8: Deploy User Backend Workloads with AWS Lambda
CloudRik uses serverless functions to isolate dynamically deployed user APIs from the core platform compute layer:
*   The CloudRik backend uses the AWS SDK to programmatically create user Lambda functions.
*   User functions follow the `cloudrik-fn-*` naming convention.
*   Backend workloads execute independently from the main CloudRik API servers.
*   Lambda Function URLs provide the endpoint used by the Nginx Edge Proxy.
*   Nginx intercepts `/api/*` requests and forwards them to the appropriate Lambda Function URL.
*   Lambda execution logs are collected through Amazon CloudWatch.

### 🔹 Step 9: Implement Static Frontend Deployment with Amazon S3
Compiled frontend applications are separated from the compute layer and stored as deployment artifacts in Amazon S3:
*   Stores compiled React applications in S3.
*   Supports static Next.js and Vue application artifacts.
*   Uses S3 as the persistent origin for deployed frontend assets.
*   Nginx dynamically retrieves requested static assets from S3.
*   Keeps frontend deployment artifacts independent from the lifecycle of EC2 compute instances.

### 🔹 Step 10: Implement Dynamic SSL & Custom Domain Provisioning
Custom-domain support requires SSL certificates and routing rules to be generated and propagated across the Nginx cluster:
*   Uses **Let's Encrypt** certificates for HTTPS.
*   Uses **Certbot** for programmatic certificate generation.
*   Stores generated certificates on the shared EFS filesystem.
*   Generates and updates Nginx routing configuration for newly mapped domains.
*   Uses the shared EFS state so dynamically scaled Nginx nodes can access the same certificate and routing configuration.

### 🔹 Step 11: Apply IAM Least-Privilege Access
CloudRik uses IAM instance profiles to provide AWS permissions to the compute layer without embedding long-lived AWS credentials into the application:
*   Runs EC2 compute instances with IAM Instance Profiles.
*   Grants only the AWS permissions required by the platform.
*   Allows the compute layer to trigger CodeBuild builds.
*   Allows required access to specific S3 deployment resources.
*   Allows management of the Lambda functions required by CloudRik.
*   Keeps AWS resource operations controlled through IAM rather than hard-coded credentials.

### 🔹 Step 12: Stream Runtime Logs to the CloudRik Dashboard
CloudRik integrates AWS CloudWatch with Redis pub/sub and WebSockets to provide runtime visibility for user workloads:
*   Lambda execution logs are collected through **Amazon CloudWatch**.
*   Runtime log information is retrieved by the platform.
*   Redis pub/sub is used as the internal event-distribution mechanism.
*   WebSockets stream relevant runtime information toward the user's CloudRik dashboard.
*   This creates a real-time feedback path between serverless execution and the deployment dashboard.

### 🔹 Step 13: Implement Automated Resource Teardown
CloudRik includes an automated cleanup workflow to remove cloud resources and stored deployment state when a user deletes a project:
*   Project deletion triggers an asynchronous cleanup process.
*   The associated AWS Lambda function is destroyed.
*   Deployment artifacts are removed from the corresponding S3 storage.
*   PostgreSQL deployment metadata is purged.
*   Redis logs associated with the deployment are removed.
*   Nginx routing rules are cleaned from the shared EFS filesystem.
*   The lifecycle process reduces abandoned cloud resources and helps control infrastructure sprawl.

---

## 🧪 Comprehensive Platform Testing & Verification

The CloudRik architecture should be validated at each major infrastructure boundary rather than relying only on successful deployment from the dashboard. Verification should cover compute availability, domain routing, build execution, artifact delivery, Lambda invocation, logging, and resource cleanup.

### Test A: Load-Balanced Application Request
*   **Source:** External client
*   **Target Destination:** CloudRik Application Load Balancer
*   **Expected Flow:** ALB → Auto Scaling Group → Nginx Edge Proxy → CloudRik API
*   **Verification:** Confirm that application traffic reaches the API through the load-balancing layer and is served by an ASG-managed compute node.

### Test B: Custom Domain Routing
*   **Source:** User browser / external HTTP client
*   **Target Destination:** Configured CloudRik custom domain
*   **Expected Flow:** Domain → Nginx → configured project origin
*   **Verification:** Confirm that the requested custom domain resolves to the correct hosted project and that the corresponding Nginx routing configuration is active.

### Test C: Frontend Build & S3 Artifact Delivery
*   **Source:** GitHub repository push
*   **Target Destination:** AWS CodeBuild → Amazon S3
*   **Expected Flow:** GitHub Webhook → CloudRik API → CodeBuild → Build Artifact → S3
*   **Verification:** Confirm that the build completes successfully and generated frontend assets are available from the deployment origin.

### Test D: Serverless API Invocation
*   **Source:** Client request to `/api/*`
*   **Target Destination:** AWS Lambda Function URL
*   **Expected Flow:** Client → Nginx → Lambda Function URL → Lambda execution
*   **Verification:** Confirm that the correct user Lambda function executes and returns the expected API response.

### Test E: Runtime Log Visibility
*   **Source:** Lambda execution
*   **Target Destination:** CloudRik dashboard
*   **Expected Flow:** Lambda → CloudWatch → CloudRik log processing → Redis pub/sub → WebSocket → Dashboard
*   **Verification:** Confirm that runtime execution information becomes visible through the dashboard's real-time logging workflow.

### Test F: Automated Project Teardown
*   **Source:** Project deletion request
*   **Target Destination:** AWS resources associated with the project
*   **Expected Flow:** Delete Request → Cleanup Job → Lambda + S3 + PostgreSQL + Redis + EFS cleanup
*   **Verification:** Confirm that project-specific cloud resources and deployment metadata are removed after deletion.

---

## 🔐 Security & IAM Controls

CloudRik's documented architecture applies several security boundaries between public traffic, stateless compute, stateful services, and user workloads:

*   **VPC Isolation:** AWS resources are logically separated using Amazon VPC.
*   **Private Database Layer:** PostgreSQL and Redis operate on a dedicated node inside a private subnet.
*   **Restricted Database Access:** PostgreSQL `5432` and Redis `6379` access is restricted to internal VPC compute sources.
*   **IAM Instance Profiles:** EC2 workloads use IAM roles instead of requiring long-lived AWS credentials.
*   **Least Privilege:** IAM permissions are scoped to the AWS operations required by CloudRik, including CodeBuild, S3, and Lambda management.
*   **Dynamic SSL:** HTTPS certificates are generated through Let's Encrypt and managed with Certbot.
*   **Serverless Workload Isolation:** User backend workloads execute through AWS Lambda rather than directly inside the core CloudRik API process.

---

## 🔄 Automated Resource Lifecycle

CloudRik treats deployment resources as lifecycle-managed infrastructure rather than permanent cloud objects. When a user removes a project, the platform initiates asynchronous cleanup using the AWS SDK:

*   **Lambda:** Deletes the project's associated `cloudrik-fn-*` function.
*   **Amazon S3:** Removes deployment artifacts.
*   **PostgreSQL:** Removes project deployment metadata.
*   **Redis:** Removes deployment-related logs.
*   **Amazon EFS:** Removes Nginx routing configuration associated with the project.

This cleanup workflow is designed to reduce unused infrastructure, stale deployment state, and unnecessary AWS resource consumption.

---

## 📊 End-to-End Deployment Flow

| Stage | Platform Action | AWS / Infrastructure Component |
| :--- | :--- | :--- |
| **1. Source Push** | Developer pushes application code | GitHub |
| **2. Webhook** | GitHub notifies CloudRik | GitHub Webhook → CloudRik API |
| **3. Build** | Application is compiled | AWS CodeBuild |
| **4. Artifact Storage** | Build output is stored | Amazon S3 |
| **5. Frontend Routing** | Static assets are served | Nginx → S3 |
| **6. Backend Provisioning** | User API is deployed | AWS Lambda |
| **7. API Routing** | `/api/*` traffic is forwarded | Nginx → Lambda Function URL |
| **8. Runtime Logging** | Function execution logs are collected | CloudWatch |
| **9. Dashboard Streaming** | Runtime information is streamed | Redis Pub/Sub → WebSocket |
| **10. Resource Cleanup** | Deleted project resources are removed | Lambda + S3 + PostgreSQL + Redis + EFS |

---

## 🧩 Technology Stack

| Category | Technologies |
| :--- | :--- |
| **Cloud Provider** | Amazon Web Services (AWS) |
| **Compute & Scaling** | Amazon EC2, Auto Scaling Groups, Application Load Balancer |
| **Serverless Compute** | AWS Lambda, Lambda Function URLs |
| **Storage** | Amazon S3, Amazon EFS |
| **Database & Cache** | PostgreSQL, Redis |
| **CI/CD & Build** | AWS CodeBuild, GitHub Webhooks |
| **Networking & Proxy** | Amazon VPC, Nginx |
| **DNS / Edge Integration** | Cloudflare |
| **TLS / Certificates** | Let's Encrypt, Certbot |
| **Observability** | Amazon CloudWatch |
| **Application Runtime** | Node.js, PM2 |
| **AWS SDK** | `@aws-sdk` v3 |

---

## 📸 Proof of Work (Deployment Verification)

Add your actual AWS Console, CloudRik dashboard, terminal, and deployment screenshots to the repository and replace the placeholder image paths below.

<table>
  <tr>
    <td width="50%" align="center">
      <b>🖥️ 1. CloudRik Platform / Dashboard</b><br><br>
      <img src="./screenshots/cloudrik-dashboard.png" alt="CloudRik dashboard" width="100%">
    </td>
    <td width="50%" align="center">
      <b>⚖️ 2. Application Load Balancer & Target Group</b><br><br>
      <img src="./screenshots/alb-target-group.png" alt="Application Load Balancer" width="100%">
    </td>
  </tr>
  <tr>
    <td width="50%" align="center">
      <b>📈 3. Auto Scaling Group & EC2 Instances</b><br><br>
      <img src="./screenshots/asg-ec2.png" alt="Auto Scaling Group and EC2 instances" width="100%">
    </td>
    <td width="50%" align="center">
      <b>💾 4. Amazon EFS Shared Storage</b><br><br>
      <img src="./screenshots/efs.png" alt="Amazon EFS" width="100%">
    </td>
  </tr>
  <tr>
    <td width="50%" align="center">
      <b>🔨 5. AWS CodeBuild Deployment</b><br><br>
      <img src="./screenshots/codebuild.png" alt="AWS CodeBuild" width="100%">
    </td>
    <td width="50%" align="center">
      <b>☁️ 6. AWS Lambda Functions</b><br><br>
      <img src="./screenshots/lambda.png" alt="AWS Lambda functions" width="100%">
    </td>
  </tr>
  <tr>
    <td width="50%" align="center">
      <b>🪣 7. Amazon S3 Deployment Artifacts</b><br><br>
      <img src="./screenshots/s3-artifacts.png" alt="Amazon S3 deployment artifacts" width="100%">
    </td>
    <td width="50%" align="center">
      <b>📊 8. CloudWatch Runtime Logs</b><br><br>
      <img src="./screenshots/cloudwatch.png" alt="Amazon CloudWatch logs" width="100%">
    </td>
  </tr>
  <tr>
    <td colspan="2" width="100%" align="center">
      <b>🌐 9. Custom Domain / HTTPS Deployment Verification</b><br><br>
      <img src="./screenshots/custom-domain-https.png" alt="Custom domain HTTPS deployment" width="100%">
    </td>
  </tr>
</table>

---

## 🚀 Recommended Production Enhancements

The following items are **recommended future improvements** and are not presented as currently implemented CloudRik infrastructure:

*   **Amazon RDS Multi-AZ:** Replace the single dedicated PostgreSQL EC2 node with a managed, highly available database architecture.
*   **Amazon ElastiCache:** Move Redis to a managed caching layer with improved availability and operational management.
*   **CloudFront + S3 Origin Access Control:** Introduce a dedicated CDN and private S3 origin architecture for static assets.
*   **AWS WAF:** Add web-application firewall protection in front of public application entry points.
*   **AWS Secrets Manager:** Store application secrets and database credentials in a managed secrets system rather than configuration files.
*   **CloudTrail:** Enable centralized AWS API activity auditing.
*   **CloudWatch Alarms:** Add infrastructure and application alarms for CPU, memory, request errors, build failures, Lambda failures, and capacity events.
*   **Infrastructure as Code:** Reproduce the complete AWS environment through Terraform or AWS CloudFormation.
*   **Multi-AZ State Layer:** Remove the documented single-node dependency for PostgreSQL and Redis to improve failure tolerance.
*   **Deployment Rollback:** Add versioned artifacts and automated rollback capabilities for failed deployments.
*   **Backup & Disaster Recovery:** Establish scheduled database backups, tested restoration procedures, and documented recovery objectives.
*   **Tenant Resource Controls:** Add per-user quotas, Lambda concurrency controls, build limits, and execution time restrictions to protect the shared platform from abusive workloads.

---

## 📚 Project Summary

CloudRik demonstrates how a modern serverless PaaS can be assembled from independently scalable AWS building blocks. The architecture separates the **edge-routing layer, stateless compute layer, stateful data layer, shared configuration storage, CI/CD build engine, serverless user workloads, static artifact storage, and observability pipeline**.

The project therefore demonstrates practical knowledge of **AWS cloud architecture, networking, compute scaling, reverse proxy design, serverless deployment automation, CI/CD, storage, IAM, observability, and automated cloud-resource lifecycle management**.

