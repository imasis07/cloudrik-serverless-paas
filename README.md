# ☁️ CloudRik — Enterprise Serverless PaaS Platform

## 📌 Project Overview

**CloudRik** is a highly scalable, AWS-native **Platform-as-a-Service (PaaS)** designed to automate the deployment, scaling, and management of modern web applications.

The platform is designed as a custom alternative to platforms such as **Vercel** and **Netlify**, providing automated application deployment, custom-domain routing, CI/CD workflows, serverless backend execution, and cloud resource lifecycle management.

This project focuses primarily on the **Cloud Architecture & DevOps Infrastructure** behind the platform, demonstrating how independently scalable compute, storage, build, database, caching, and serverless components can be combined into a decoupled cloud architecture.

The infrastructure is built around **Amazon Web Services (AWS)** and integrates services including **Amazon EC2, Auto Scaling Groups, Application Load Balancer, AWS Lambda, AWS CodeBuild, Amazon S3, Amazon EFS, Amazon VPC, AWS IAM, CloudWatch, PostgreSQL, Redis, Nginx, Cloudflare, and Let's Encrypt**.

---

## 🎯 Project Objectives

The primary objectives of CloudRik are to:

- 🚀 Automate deployment of modern web applications.
- ⚡ Provide serverless execution for user backend workloads.
- 📦 Store and serve compiled frontend application artifacts.
- 🔨 Automate application builds through a dedicated CI/CD build environment.
- 📈 Support horizontally scalable application infrastructure.
- 🌐 Provide custom-domain and application routing.
- 🔐 Secure cloud resources using private networking and IAM-based access control.
- 📁 Synchronize shared Nginx configuration and SSL certificates across compute nodes.
- 🧹 Automatically clean up application-specific cloud resources when projects are deleted.
- ☁️ Demonstrate production-oriented AWS cloud architecture and DevOps practices.

---

## 🏗️ High-Level Architecture

CloudRik follows a **decoupled and stateless compute architecture** in which the primary application layer is separated from persistent data, shared configuration, build workloads, and dynamically provisioned user workloads.

The platform uses an **EC2 Auto Scaling Group** behind an **Application Load Balancer** for the core API and Nginx edge-proxy layer. Stateful services are separated from the scalable compute layer, while AWS CodeBuild handles resource-intensive application builds and AWS Lambda provides isolated serverless execution for user backend workloads.

![CloudRik Architecture Diagram](./architecture-diagram.png)

---

## 🛠️ Infrastructure Components & Details

| Architecture Layer | Component | Primary Responsibility |
| :--- | :--- | :--- |
| **Edge & Routing** | Nginx | Reverse proxy, SSL termination, domain and application routing |
| **Traffic Distribution** | Application Load Balancer | Distributes incoming requests across EC2 compute nodes |
| **Compute** | Amazon EC2 | Hosts CloudRik API and Nginx edge-proxy instances |
| **Scaling** | Auto Scaling Group | Automatically manages and scales EC2 application nodes |
| **Serverless Compute** | AWS Lambda | Executes dynamically deployed user backend workloads |
| **Build Engine** | AWS CodeBuild | Builds user applications in isolated build environments |
| **Object Storage** | Amazon S3 | Stores compiled frontend deployment artifacts |
| **Shared Storage** | Amazon EFS | Stores shared Nginx configuration and SSL certificates |
| **Database** | PostgreSQL | Stores persistent CloudRik application metadata |
| **Caching / Messaging** | Redis | Provides caching and real-time platform communication |
| **Networking** | Amazon VPC | Provides isolated private cloud networking |
| **Identity & Access** | AWS IAM | Controls access to AWS resources using IAM roles |
| **Monitoring** | Amazon CloudWatch | Collects and exposes application and serverless logs |
| **DNS / Edge Services** | Cloudflare | Handles external domain and traffic management |
| **SSL/TLS** | Let's Encrypt + Certbot | Provides automated SSL certificate generation |

---

## 🔄 Core Platform Workflow

The overall CloudRik deployment workflow can be represented as:

```text
Developer
    │
    ▼
GitHub Repository
    │
    │ Webhook
    ▼
CloudRik API
    │
    ├──────────────────────────┐
    │                          │
    ▼                          ▼
AWS CodeBuild              AWS Lambda
    │                          │
    │ Build Application        │ Deploy Backend
    ▼                          ▼
Build Artifacts            User API
    │
    ▼
Amazon S3
    │
    ▼
Frontend Application
