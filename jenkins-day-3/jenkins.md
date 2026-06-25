# monolithic Services

Management Account
┌────────────────────────────┐
│ VPC: Management            │
│                            │
│ Jenkins                    │
│ Terraform                  │
│ SonarQube                  │
│ Nexus                      │
└─────────────┬──────────────┘
              │ IAM Role
              ▼
        AWS APIs
              │
   ┌──────────┴───────────┐
   │                      │
   ▼                      ▼
Dev Account           Prod Account
VPC-A                 VPC-B
EC2                   EC2
RDS                   RDS
ALB                   ALB

===============================================================================================================

                    Management AWS Account / VPC
                  +-------------------------------+
                  | Jenkins EC2                  |
                  | Git                          |
                  | Terraform                    |
                  | Ansible                      |
                  +---------------+--------------+
                                  |
                         AWS API (IAM Role/User)
                                  |
                                  ▼
                Production AWS Account / VPC
        +------------------------------------------------+
        | Terraform creates:                             |
        |                                                |
        |  VPC                                           |
        |  Public Subnets                                |
        |  Private Subnets                               |
        |  Internet Gateway                              |
        |  NAT Gateway                                   |
        |  Security Groups                               |
        |  ALB / NLB                                     |
        |  EC2 Instances                                 |
        |  RDS                                           |
        |  EKS / ECS                                     |
        |  S3 / IAM / Route53                            |
        +------------------------------------------------+



===============================================================================================================
# main flow 

                                   Developer
                                       │
                           git push develop / test / main
                                       │
                                       ▼
                              GitHub Repository
                                       │
                                GitHub Webhook
                                       │
                                       ▼
═══════════════════════════════════════════════════════════════
          Management AWS Account / Management VPC
═══════════════════════════════════════════════════════════════

                     Jenkins Controller (Master)
                   (Private Subnet - No Public IP)
                               │
                 Reads Jenkinsfile & schedules jobs
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
  Backend Agent         Frontend Agent        Infra/K8s Agent
 (Python Build)         (React Build)        (Terraform/K8s)
        │                      │                      │
        └──────────────┬───────┴──────────────┬───────┘
                       │                      │
               Build Docker Images     Push Images
                       │                      │
                       └──────────────┬───────┘
                                      │
                                      ▼
                                 Amazon ECR
                                      │
═══════════════════════════════════════════════════════════════
             Environment Selection (Based on Branch)
═══════════════════════════════════════════════════════════════

develop branch ─────────────► DEV

test branch ────────────────► TEST

main branch ────────────────► PROD

                                      │
                                      ▼
═══════════════════════════════════════════════════════════════
                         DEV Environment
═══════════════════════════════════════════════════════════════

Terraform Apply

Creates

DEV VPC
├── Public Subnets
│      ├── ALB
│      └── NAT Gateway
│
├── Private Subnets
│      ├── EKS Cluster
│      ├── Backend Pods
│      ├── Frontend Pods
│      └── RDS (Dev)

Deploy

Frontend Image (Dev)

Backend Image (Dev)

Smoke Tests

───────────────────────────────────────────────────────────────
                 Manual Approval / QA Approval
───────────────────────────────────────────────────────────────

                                      │
                                      ▼

═══════════════════════════════════════════════════════════════
                         TEST Environment
═══════════════════════════════════════════════════════════════

Terraform Apply

Creates

TEST VPC
├── Public Subnets
├── ALB
├── NAT Gateway
├── Private Subnets
├── EKS
├── Frontend Pods
├── Backend Pods
└── RDS (Test)

Integration Testing

Performance Testing

QA Validation

───────────────────────────────────────────────────────────────
               Change Approval / Release Approval
───────────────────────────────────────────────────────────────

                                      │
                                      ▼

═══════════════════════════════════════════════════════════════
                        PROD Environment
═══════════════════════════════════════════════════════════════

Terraform Apply

Creates

PROD VPC
├── Public Subnets
│      └── Application Load Balancer
│
├── Private Subnets
│      ├── EKS Worker Nodes
│      ├── Frontend Pods
│      ├── Backend Pods
│      └── Amazon RDS
│
└── Monitoring
       ├── CloudWatch
       ├── Prometheus
       └── Grafana

Deploy Latest Approved Images

Health Checks

Production Release

═══════════════════════════════════════════════════════════════

Traffic

Internet
     │
     ▼
Application Load Balancer
     │
     ▼
Ingress Controller
     │
     ▼
Frontend Service
     │
     ▼
Frontend Pods
     │
HTTP /api
     ▼
Backend Service
     │
     ▼
Backend Pods
     │
     ▼
Amazon RDS


#  Enterprise Microservices CI/CD Architecture

                                      Developers
                                           │
                               Feature Development
                                           │
                 ┌───────────────┬───────────────┬───────────────┐
                 │               │               │               │
                 ▼               ▼               ▼               ▼

         frontend-repo     user-service     product-service    order-service
         (React)           (Python)         (Python)           (Python)

                 ▼               ▼               ▼               ▼

         payment-service   inventory-service  notification-service
             (Python)          (Python)            (Python)

                              │
                              │
                              ▼

                  terraform-infra-repo
                  helm-charts-repo
                  kubernetes-manifests-repo

══════════════════════════════════════════════════════════════════════════════

                           GitHub Organization

    frontend-repo
    user-service-repo
    product-service-repo
    order-service-repo
    payment-service-repo
    inventory-service-repo
    notification-service-repo
    terraform-infra-repo
    helm-charts-repo

══════════════════════════════════════════════════════════════════════════════

                         GitHub Webhooks

                Any code push automatically triggers
                   the corresponding Jenkins job

══════════════════════════════════════════════════════════════════════════════
                Management AWS Account / Management VPC
══════════════════════════════════════════════════════════════════════════════

                    Private Subnet

              Jenkins Controller (Master)

        • Receives GitHub Webhooks

        • Reads Jenkinsfile

        • Stores Build History

        • Assigns Jobs

        • Does NOT build applications

                             │

         ┌───────────────────┼────────────────────┐
         │                   │                    │

         ▼                   ▼                    ▼

  Jenkins Agent-1     Jenkins Agent-2     Jenkins Agent-3

  Backend Builds      Frontend Builds     Infra/Kubernetes

  Git                 Git                 Git

  Python              NodeJS              Terraform

  Docker              Docker              kubectl

  SonarQube           SonarQube           Helm

  Trivy               Trivy               AWS CLI

══════════════════════════════════════════════════════════════════════════════
                     Individual Application Pipelines
══════════════════════════════════════════════════════════════════════════════

Frontend Pipeline

Git Checkout

↓

npm install

↓

npm test

↓

npm run build

↓

Docker Build

↓

Trivy Scan

↓

Push Image to Amazon ECR


------------------------------------------------------------

User Service Pipeline

Git Checkout

↓

pip install

↓

pytest

↓

SonarQube

↓

Docker Build

↓

Trivy Scan

↓

Push Image to Amazon ECR


------------------------------------------------------------

Product Service Pipeline

Git Checkout

↓

Build

↓

Test

↓

Docker Build

↓

Push Image to Amazon ECR


------------------------------------------------------------

Order Service Pipeline

Git Checkout

↓

Build

↓

Test

↓

Docker Build

↓

Push Image to Amazon ECR


------------------------------------------------------------

Payment Pipeline

↓

Docker Build

↓

Push Image


------------------------------------------------------------

Inventory Pipeline

↓

Docker Build

↓

Push Image


------------------------------------------------------------

Notification Pipeline

↓

Docker Build

↓

Push Image

══════════════════════════════════════════════════════════════════════════════
                      Infrastructure Pipeline
══════════════════════════════════════════════════════════════════════════════

terraform-infra-repo

↓

Git Checkout

↓

terraform fmt

↓

terraform validate

↓

terraform init

↓

terraform plan

↓

(terraform approval)

↓

terraform apply

↓

Creates AWS Infrastructure

══════════════════════════════════════════════════════════════════════════════
                        AWS Infrastructure
══════════════════════════════════════════════════════════════════════════════

DEV AWS Account / DEV VPC

├── Public Subnets
│      ├── Internet Gateway
│      ├── NAT Gateway
│      └── Application Load Balancer
│
├── Private Subnets
│      ├── Amazon EKS
│      ├── Amazon RDS
│      ├── Redis
│      └── Worker Nodes

--------------------------------------------------------------

TEST AWS Account / TEST VPC

├── Public Subnets
├── Private Subnets
├── Amazon EKS
├── Amazon RDS

--------------------------------------------------------------

PROD AWS Account / PROD VPC

├── Public Subnets
├── Private Subnets
├── Amazon EKS
├── Amazon RDS
├── Redis
├── CloudWatch

══════════════════════════════════════════════════════════════════════════════
                      Kubernetes Deployment Pipeline
══════════════════════════════════════════════════════════════════════════════

helm-charts-repo

↓

Git Checkout

↓

aws eks update-kubeconfig

↓

helm upgrade --install

↓

Deploy to DEV

↓

Smoke Test

↓

Approval

↓

Deploy to TEST

↓

Integration Test

↓

Approval

↓

Deploy to PROD

══════════════════════════════════════════════════════════════════════════════
                           Amazon EKS
══════════════════════════════════════════════════════════════════════════════

Namespaces

dev
test
prod

Each namespace contains

Frontend Deployment

User Service

Product Service

Order Service

Payment Service

Inventory Service

Notification Service

Services

Ingress

ConfigMaps

Secrets

══════════════════════════════════════════════════════════════════════════════
                            Traffic Flow
══════════════════════════════════════════════════════════════════════════════

Internet

↓

Application Load Balancer

↓

AWS Load Balancer Controller

↓

Ingress

↓

Frontend Service

↓

Frontend Pods

↓

API Routes

/users -----------> User Service

/products --------> Product Service

/orders ----------> Order Service

/payments --------> Payment Service

/inventory -------> Inventory Service

/notifications ---> Notification Service

↓

Amazon RDS

↓

Redis

══════════════════════════════════════════════════════════════════════════════
                              Monitoring
══════════════════════════════════════════════════════════════════════════════

Prometheus

↓

Grafana

↓

CloudWatch

↓

AlertManager

↓

Slack / Email Notifications






# =======================================================================

If asked:

"How do you know whether an application is monolithic or microservices?"

You can answer:

"If there is a single backend application that contains all business modules—such as users, products, orders, and payments—and it is built and deployed as one unit, it is a monolithic architecture. If each business capability is developed, built, deployed, and scaled independently as separate services with separate Docker images and CI/CD pipelines, it is a microservices architecture."

So your earlier project (React + one Flask backend + one RDS) is a monolithic application deployed on Kubernetes, not a microservices application.

# ======================================================================

Interview Answer

If asked:

"In microservices, do all services use the same GitHub repository?"

You can answer:

"There are two common approaches. In a polyrepo model, each microservice has its own Git repository, its own CI/CD pipeline, and can be deployed independently. This is common in large enterprises because it provides clear ownership and independent releases. In a monorepo model, all services are stored in a single repository, but the pipeline detects which services changed and builds only those. Both approaches are used in industry, depending on team size and organizational needs."

# ======================================================================

Interview Answer

If asked:

"In microservices, do all services use the same Kubernetes cluster?"

You can answer:

"No. In a polyrepo model, each microservice has its own Kubernetes cluster, its own CI/CD pipeline, and can be deployed independently. This is common in large enterprises because it provides clear ownership and independent releases. In a monorepo model, all services are stored in a single repository, but the pipeline detects which services changed and deploys only those. Both approaches are used in industry, depending on team size and organizational needs."

# ======================================================================
Option 1: Multiple Repositories (Most Common)

Each microservice has its own Git repository.

GitHub Organization

├── frontend-repo
├── user-service-repo
├── product-service-repo
├── order-service-repo
├── payment-service-repo
├── inventory-service-repo
├── notification-service-repo
├── terraform-infra-repo
└── helm-charts-repo

Each repository contains:

Source code
Dockerfile
Jenkinsfile
README
Tests

Example:

user-service-repo/

├── src/
├── tests/
├── Dockerfile
├── Jenkinsfile
└── requirements.txt

When a developer pushes to user-service-repo:

GitHub

↓

user-service-repo

↓

Webhook

↓

Only User Service Pipeline Runs

↓

Build User Image

↓

Deploy User Service

The Product Service and Order Service are not rebuilt.

Advantages
Independent deployments
Separate versioning
Different teams own different services
Faster CI/CD

This is the most common approach in enterprises.

# =======================================================================

Option 2: Monorepo

All services are stored in one repository.

ecommerce-platform/

├── frontend/
├── user-service/
├── product-service/
├── order-service/
├── payment-service/
├── inventory-service/
├── terraform/
├── helm/
└── Jenkinsfile

The pipeline detects which folders changed.