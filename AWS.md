# AWS — Complete Notes (Basics → Advanced + Interview Prep)

*Note: VPC, subnets, routing, load balancers, DNS, and CDN are covered in depth in the separate Cloud Networking notes — this file focuses on compute, storage, databases, serverless, IAM/security, messaging, and operations, with light cross-references back to networking where relevant.*

---

## Table of Contents
1. [Introduction](#1-introduction)
2. [AWS Global Infrastructure](#2-aws-global-infrastructure)
3. [IAM — Identity & Access Management](#3-iam--identity--access-management)
4. [EC2 — Elastic Compute Cloud](#4-ec2--elastic-compute-cloud)
5. [Auto Scaling & Elastic Load Balancing](#5-auto-scaling--elastic-load-balancing)
6. [S3 — Simple Storage Service](#6-s3--simple-storage-service)
7. [EBS & EFS — Block & File Storage](#7-ebs--efs--block--file-storage)
8. [RDS & Aurora — Relational Databases](#8-rds--aurora--relational-databases)
9. [DynamoDB — NoSQL Database](#9-dynamodb--nosql-database)
10. [ElastiCache](#10-elasticache)
11. [Lambda & Serverless](#11-lambda--serverless)
12. [API Gateway](#12-api-gateway)
13. [Containers — ECS, EKS, Fargate, ECR](#13-containers--ecs-eks-fargate-ecr)
14. [Messaging & Integration — SQS, SNS, EventBridge, Kinesis](#14-messaging--integration)
15. [Step Functions](#15-step-functions)
16. [Networking Recap](#16-networking-recap)
17. [CloudWatch — Monitoring & Logging](#17-cloudwatch--monitoring--logging)
18. [CloudTrail & Config](#18-cloudtrail--config)
19. [Security Services](#19-security-services)
20. [Cost Management](#20-cost-management)
21. [AWS Well-Architected Framework](#21-aws-well-architected-framework)
22. [Infrastructure as Code on AWS](#22-infrastructure-as-code-on-aws)
23. [CI/CD on AWS](#23-cicd-on-aws)
24. [High Availability & Disaster Recovery](#24-high-availability--disaster-recovery)
25. [AWS CLI & SDK Basics](#25-aws-cli--sdk-basics)
26. [Troubleshooting](#26-troubleshooting)
27. [Best Practices Summary](#27-best-practices-summary)
28. [Cheat Sheet](#28-cheat-sheet)
29. [Interview Questions & Answers](#29-interview-questions--answers)

---

## 1. Introduction

**AWS (Amazon Web Services)** is the largest public cloud provider, offering 200+ on-demand services spanning compute, storage, databases, networking, machine learning, analytics, and more — accessible via console, CLI, SDKs, or Infrastructure as Code (CloudFormation, Terraform, CDK).

**Core value proposition:**
- Pay only for what you use (OpEx vs CapEx) — no upfront hardware investment
- Elastic — scale resources up/down on demand
- Global reach — deploy close to users worldwide in minutes
- Managed services reduce operational burden (e.g., RDS handles patching/backups/failover for you)
- Broad ecosystem — compute, storage, ML, IoT, analytics all integrate natively

---

## 2. AWS Global Infrastructure

| Concept | Description |
|---|---|
| **Region** | A geographic area (e.g., `us-east-1`, `eu-west-1`) containing multiple isolated Availability Zones |
| **Availability Zone (AZ)** | One or more discrete data centers within a region, with independent power/cooling/networking |
| **Edge Location** | Smaller sites used by CloudFront (CDN) and Route 53 for caching content/DNS closer to end users |
| **Local Zone** | An extension of a region placed closer to large population centers for ultra-low-latency needs |

*(See the Cloud Networking notes for full detail on VPCs, subnets, and how resources are distributed across AZs.)*

**Choosing a region** typically depends on: latency to your users, data residency/compliance requirements, service availability (not every service launches in every region simultaneously), and pricing (costs vary by region).

---

## 3. IAM — Identity & Access Management

IAM controls **authentication** (who can sign in) and **authorization** (what they're allowed to do) across your AWS account — free, global (not region-scoped), and foundational to AWS security.

**Core entities:**
| Entity | Description |
|---|---|
| **User** | Represents a person or application with long-term credentials |
| **Group** | A collection of users — policies attached to a group apply to all members |
| **Role** | An identity with temporary credentials, assumed by users, applications, or AWS services (e.g., an EC2 instance assuming a role to access S3) — **preferred over long-term credentials** |
| **Policy** | A JSON document defining permissions (allow/deny specific actions on specific resources) |

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

**Policy types:**
- **Identity-based policies** — attached to users/groups/roles
- **Resource-based policies** — attached directly to a resource (e.g., an S3 bucket policy), can grant access to principals in *other* accounts
- **Permissions boundaries** — set the maximum permissions an identity-based policy can grant, regardless of what's otherwise attached
- **Service Control Policies (SCPs)** — applied at the AWS Organizations level, set permission guardrails across entire accounts/OUs

```bash
aws iam create-user --user-name jane
aws iam attach-user-policy --user-name jane --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam create-role --role-name MyEC2Role --assume-role-policy-document file://trust-policy.json
```

**IAM Roles vs Users (very common interview question):** Users have long-term credentials (access keys, password) tied to a specific identity. Roles have **no long-term credentials at all** — they're assumed temporarily (via STS, generating short-lived credentials), commonly used for EC2 instances, Lambda functions, or cross-account access, so you never need to store/rotate long-lived secrets on the resource itself. **Best practice: prefer roles over hardcoded access keys wherever possible.**

**Principle of least privilege:** Grant only the specific permissions actually needed — start with no access and add explicit allows, rather than starting broad and trying to restrict later. AWS Access Analyzer and the IAM policy simulator help validate this in practice.

**MFA (Multi-Factor Authentication):** Should be enabled on the root account at minimum, and ideally enforced for all human users via an IAM policy condition.

---

## 4. EC2 — Elastic Compute Cloud

**EC2** provides resizable virtual server instances ("VMs") in the cloud.

**Key concepts:**
| Concept | Description |
|---|---|
| **AMI (Amazon Machine Image)** | A template containing the OS, application server, and configuration used to launch an instance |
| **Instance Type** | Defines vCPU/memory/network/storage profile (e.g., `t3.micro`, `m5.large`, `c5.xlarge`) |
| **Instance Family prefixes** | `t` = burstable general purpose, `m` = balanced general purpose, `c` = compute-optimized, `r` = memory-optimized, `i`/`d` = storage-optimized, `p`/`g` = GPU/accelerated computing |
| **Key Pair** | SSH key pair used to securely access the instance (public key injected at launch, private key kept by you) |
| **Security Group** | Instance-level virtual firewall (see Cloud Networking notes) |
| **Elastic IP** | A static public IP you can attach/detach/reassign between instances |
| **User Data** | A script that runs automatically on first boot (e.g., to bootstrap configuration) |

```bash
aws ec2 run-instances --image-id ami-xxxx --instance-type t3.micro \
  --key-name mykey --security-group-ids sg-xxxx --subnet-id subnet-xxxx

aws ec2 describe-instances
aws ec2 start-instances --instance-ids i-xxxx
aws ec2 stop-instances --instance-ids i-xxxx
aws ec2 terminate-instances --instance-ids i-xxxx
```

**Purchasing options:**
| Option | Description | Best for |
|---|---|---|
| **On-Demand** | Pay per second/hour, no commitment | Unpredictable, short-term, or spiky workloads |
| **Reserved Instances (RI)** | 1 or 3-year commitment for a significant discount (up to ~72%) | Steady-state, predictable workloads |
| **Savings Plans** | Commit to a $/hour spend over 1-3 years, more flexible than RIs (covers EC2, Fargate, Lambda) | Predictable spend with some flexibility in instance type/family |
| **Spot Instances** | Bid on unused EC2 capacity for up to ~90% discount; can be reclaimed by AWS with 2-minute warning | Fault-tolerant, flexible workloads — batch jobs, CI runners, stateless workers |
| **Dedicated Hosts/Instances** | Physical server dedicated to you | Compliance/licensing requirements needing physical isolation |

**Spot Instance interview point:** AWS can reclaim a Spot Instance with only a 2-minute warning (via the instance metadata service) when it needs the capacity back — applications must be designed to tolerate sudden termination (e.g., checkpointing state, using a fleet where losing one instance doesn't lose work) to safely use Spot for cost savings.

**EC2 instance lifecycle:** `pending` → `running` → (`stopping` → `stopped`, or `shutting-down` → `terminated`). **Stopped** instances retain their EBS root volume (and its data) but release certain resources (e.g., the public IP unless using an Elastic IP); **terminated** instances are gone permanently (their root EBS volume is deleted by default, unless configured otherwise).

---

## 5. Auto Scaling & Elastic Load Balancing

**Auto Scaling Group (ASG):** Automatically launches/terminates EC2 instances to maintain a target capacity, based on demand or health.

```
Min: 2   Desired: 4   Max: 10
```

**Scaling policies:**
| Policy | Description |
|---|---|
| **Target tracking** | Maintain a specific metric at a target value (e.g., keep average CPU at 50%) — the simplest, most commonly used |
| **Step scaling** | Add/remove a specific number of instances based on the magnitude of an alarm breach |
| **Scheduled scaling** | Scale at predictable times (e.g., scale up before a known daily traffic peak) |
| **Predictive scaling** | Uses ML to forecast traffic and scale ahead of time |

**Launch Template:** Defines what a new instance launched by the ASG looks like (AMI, instance type, security groups, user data) — the modern replacement for the older "Launch Configuration."

**Health checks:** An ASG can use EC2 status checks or, more commonly, **ELB health checks** — if an instance fails health checks, the ASG terminates it and launches a replacement automatically, providing self-healing.

*(Load balancer types — ALB/NLB, L4 vs L7 — are covered in depth in the Cloud Networking notes.)*

---

## 6. S3 — Simple Storage Service

**S3** is AWS's object storage service — virtually unlimited, highly durable (11 nines — 99.999999999%) storage for any type of file (objects), accessed via a flat key-value namespace within "buckets" (not a traditional hierarchical filesystem, though prefixes simulate folders).

```bash
aws s3 mb s3://my-bucket                          # create bucket
aws s3 cp file.txt s3://my-bucket/                  # upload
aws s3 cp s3://my-bucket/file.txt ./                  # download
aws s3 sync ./local-dir s3://my-bucket/prefix/           # sync a directory
aws s3 ls s3://my-bucket/                                  # list contents
aws s3 rm s3://my-bucket/file.txt                            # delete an object
aws s3 rb s3://my-bucket --force                                # delete bucket + all contents
```

**Storage classes (cost/access-pattern trade-offs):**
| Class | Use Case | Retrieval |
|---|---|---|
| **S3 Standard** | Frequently accessed data | Milliseconds |
| **S3 Intelligent-Tiering** | Unknown/changing access patterns — automatically moves objects between tiers | Milliseconds |
| **S3 Standard-IA** (Infrequent Access) | Accessed less often but needs fast retrieval when needed | Milliseconds |
| **S3 One Zone-IA** | Same as Standard-IA but stored in only one AZ (cheaper, less durable against AZ loss) | Milliseconds |
| **S3 Glacier Instant Retrieval** | Archive data needing immediate access | Milliseconds |
| **S3 Glacier Flexible Retrieval** | Archive data, occasional access | Minutes to hours |
| **S3 Glacier Deep Archive** | Long-term archival, cheapest option | Hours (up to ~12h) |

**Key features:**
- **Versioning** — keeps multiple versions of an object, protecting against accidental overwrite/deletion
- **Lifecycle policies** — automatically transition objects between storage classes or expire them over time (e.g., move to Glacier after 90 days, delete after 1 year)
- **Bucket policies / ACLs** — control access at the bucket or object level (resource-based IAM policies)
- **Block Public Access** — account/bucket-level setting that overrides any other configuration to prevent accidental public exposure — a critical security default
- **Encryption** — server-side encryption (SSE-S3, SSE-KMS, SSE-C) or client-side
- **Static website hosting** — S3 can directly serve a static website
- **Cross-Region Replication (CRR)** — automatically replicate objects to a bucket in another region
- **Pre-signed URLs** — generate a temporary, time-limited URL granting access to a private object without making it public or requiring the requester to have AWS credentials

**S3 consistency model:** S3 now provides **strong read-after-write consistency** for all operations (a change from its earlier eventual-consistency model) — a `GET` immediately after a successful `PUT` will always return the latest data.

---

## 7. EBS & EFS — Block & File Storage

| Service | Type | Scope | Use Case |
|---|---|---|---|
| **EBS (Elastic Block Store)** | Block storage | Attached to a single EC2 instance, tied to one AZ | Instance root volumes, databases — needs a traditional filesystem/block device |
| **EFS (Elastic File System)** | Network file system (NFS) | Shared across multiple instances/AZs simultaneously | Shared application data, content management, multiple instances needing the same files |
| **Instance Store** | Ephemeral block storage physically attached to the host | Tied to the instance's lifetime — **data lost on stop/terminate** | Temporary data, caches, buffers — very high IOPS but no persistence |

**EBS volume types:**
| Type | Best for |
|---|---|
| `gp3`/`gp2` (General Purpose SSD) | Most general workloads, boot volumes |
| `io2`/`io1` (Provisioned IOPS SSD) | High-performance databases needing consistent, very high IOPS |
| `st1` (Throughput Optimized HDD) | Big data, log processing — high throughput, sequential access |
| `sc1` (Cold HDD) | Infrequently accessed data, lowest cost |

```bash
aws ec2 create-volume --availability-zone us-east-1a --size 100 --volume-type gp3
aws ec2 attach-volume --volume-id vol-xxxx --instance-id i-xxxx --device /dev/sdf
aws ec2 create-snapshot --volume-id vol-xxxx     # point-in-time backup, stored in S3 (incrementally)
```

**EBS vs EFS vs S3 (very common interview question):** EBS is block storage attached to exactly one instance at a time, AZ-bound — like a virtual hard drive. EFS is a fully managed NFS file system that multiple instances across multiple AZs can mount **simultaneously** — like a shared network drive. S3 is object storage accessed via API (HTTP), not mountable as a traditional filesystem (without extra tooling), designed for storing and retrieving discrete files/objects at massive scale rather than as a live, POSIX-compliant filesystem.

---

## 8. RDS & Aurora — Relational Databases

**RDS (Relational Database Service)** is a managed relational database service supporting MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, and Aurora — AWS handles patching, backups, and (optionally) failover.

```bash
aws rds create-db-instance --db-instance-identifier mydb \
  --db-instance-class db.t3.micro --engine postgres \
  --master-username admin --master-user-password secret \
  --allocated-storage 20
```

**Key features:**
- **Multi-AZ deployment** — synchronously replicates to a standby instance in another AZ; on failure, RDS automatically fails over to the standby (improves **availability**, not read scaling)
- **Read Replicas** — asynchronously replicated copies that can serve read traffic, offloading the primary (improves **read scalability**, can also be promoted to a standalone primary for disaster recovery)
- **Automated backups** — daily snapshots + transaction logs, enabling point-in-time recovery within the retention window
- **Manual snapshots** — user-initiated, retained until explicitly deleted

**Multi-AZ vs Read Replica (very common interview question):** Multi-AZ is purely for **high availability** — the standby is not used for read traffic and only takes over automatically on a failure (synchronous replication). Read Replicas are primarily for **read scalability** — they actively serve read queries to offload the primary, use asynchronous replication (so there's some replication lag), and don't automatically fail over on primary failure (though they can be manually promoted).

**Aurora:** AWS's proprietary, MySQL/PostgreSQL-compatible relational database engine, re-architected for the cloud — storage is automatically replicated 6 ways across 3 AZs, decoupled from compute, offers higher throughput and faster failover than standard RDS engines, plus features like **Aurora Serverless** (auto-scaling capacity based on load) and a "Global Database" option for low-latency cross-region reads.

---

## 9. DynamoDB — NoSQL Database

**DynamoDB** is a fully managed, serverless key-value and document NoSQL database, designed for single-digit millisecond latency at virtually any scale.

**Core concepts:**
| Concept | Description |
|---|---|
| **Table** | A collection of items (rows) |
| **Item** | A single record (like a row), composed of attributes |
| **Primary Key** | Either a simple **Partition Key** alone, or a **Composite Key** (Partition Key + Sort Key) |
| **Partition Key** | Determines which physical partition stores the item — should have high cardinality for even distribution |
| **Sort Key** | Allows multiple items per partition key, sorted/queryable by this key |
| **Global Secondary Index (GSI)** | An alternate partition/sort key combination for different query patterns, can have different throughput |
| **Local Secondary Index (LSI)** | Same partition key as the base table, different sort key — must be created at table creation time |

```bash
aws dynamodb create-table --table-name Users \
  --attribute-definitions AttributeName=UserId,AttributeType=S \
  --key-schema AttributeName=UserId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

aws dynamodb put-item --table-name Users --item '{"UserId": {"S": "u123"}, "Name": {"S": "Jane"}}'
aws dynamodb get-item --table-name Users --key '{"UserId": {"S": "u123"}}'
aws dynamodb query --table-name Users --key-condition-expression "UserId = :id" --expression-attribute-values '{":id": {"S": "u123"}}'
```

**Capacity modes:** **On-demand** (pay per request, auto-scales instantly, simplest) vs **Provisioned** (specify read/write capacity units in advance, cheaper at predictable steady-state load, supports auto-scaling on top).

**DynamoDB Streams:** Captures a time-ordered sequence of item-level changes (insert/update/delete) — commonly used to trigger Lambda functions for event-driven processing (e.g., updating a search index whenever an item changes).

**DynamoDB vs RDS (very common interview question):** DynamoDB is schema-less NoSQL, scales horizontally and near-infinitely with consistent single-digit-millisecond performance, but has limited query flexibility (designed around knowing your access patterns upfront — no arbitrary joins, limited ad-hoc querying) and eventual consistency by default (strong consistency is available per-request at higher cost). RDS is a traditional relational database with full SQL support, joins, and ACID transactions, but doesn't scale horizontally as seamlessly and requires more capacity planning. Choose DynamoDB for massive scale with simple, well-known access patterns; choose RDS when you need complex queries, joins, or strict relational integrity.

---

## 10. ElastiCache

A managed in-memory caching service supporting **Redis** or **Memcached** — used to reduce database load and dramatically speed up read-heavy applications by caching frequently accessed data in memory.

**Redis vs Memcached (interview point):** Redis supports rich data structures (lists, sets, sorted sets, hashes), persistence, replication, pub/sub, and clustering — generally the more feature-rich and commonly chosen option today. Memcached is simpler, purely in-memory key-value caching with multi-threading support, no persistence or replication — suitable for simple, pure caching needs without any data durability requirement.

**Common use cases:** Session storage, database query result caching, leaderboards (Redis sorted sets), rate limiting, pub/sub messaging.

---

## 11. Lambda & Serverless

**Lambda** runs code in response to events **without provisioning or managing servers** — you pay only for actual compute time consumed (billed per millisecond), and AWS handles all scaling automatically.

```bash
aws lambda create-function --function-name myFunction \
  --runtime python3.12 --handler app.handler \
  --role arn:aws:iam::123456789:role/lambda-role \
  --zip-file fileb://function.zip

aws lambda invoke --function-name myFunction output.json
```

```python
# Example handler (Python)
def handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Hello from Lambda!'
    }
```

**Key concepts:**
- **Trigger/Event source** — what invokes the function: API Gateway, S3 events, DynamoDB Streams, SQS, EventBridge schedules, direct invocation, etc.
- **Cold start** — the latency penalty when Lambda must initialize a new execution environment (download code, start the runtime) before running your function for the first time (or after scaling up) — subsequent "warm" invocations reuse the environment and are much faster
- **Concurrency** — Lambda automatically scales out by running many instances of your function in parallel for concurrent events; **Reserved Concurrency** caps/guarantees a specific function's concurrency; **Provisioned Concurrency** keeps execution environments pre-initialized to eliminate cold starts for latency-sensitive workloads
- **Timeout** — max execution duration (up to 15 minutes)
- **Layers** — reusable packages of code/dependencies shared across multiple functions

**Lambda vs EC2/containers (very common interview question):** Lambda is fully serverless — no server management, automatic scaling to zero (no cost when idle), billed per-millisecond of actual execution, but with execution time limits (15 min max), potential cold-start latency, and less control over the runtime environment. EC2/containers give you full control over the OS/runtime and no execution time limits, but you manage scaling/patching/capacity yourself (or via ECS/EKS) and pay for idle capacity unless carefully managed. Lambda fits event-driven, intermittent, or unpredictable workloads well; EC2/containers fit long-running, steady-state, or highly customized workloads better.

---

## 12. API Gateway

A fully managed service for creating, publishing, and securing APIs (REST, HTTP, or WebSocket) that typically front Lambda functions or other backends.

**Key features:**
- Request/response transformation and validation
- Authentication/authorization (IAM, Cognito, custom Lambda authorizers, API keys)
- Throttling and rate limiting per client
- Caching responses to reduce backend load
- Stage-based deployments (dev/staging/prod) with independent configuration

**REST API vs HTTP API (interview-relevant, cost/feature trade-off):** HTTP APIs are newer, cheaper, and lower-latency, covering the most common use cases (simple proxying to Lambda, JWT authorization). REST APIs offer the full, more mature feature set (request validation, API keys, usage plans, more extensive caching options) at a higher cost — choose HTTP API by default unless you specifically need a REST-API-only feature.

---

## 13. Containers — ECS, EKS, Fargate, ECR

| Service | Description |
|---|---|
| **ECR (Elastic Container Registry)** | Managed Docker image registry, integrates natively with IAM for access control |
| **ECS (Elastic Container Service)** | AWS's own proprietary container orchestrator — simpler than Kubernetes, tightly integrated with other AWS services |
| **EKS (Elastic Kubernetes Service)** | Managed Kubernetes — AWS runs the control plane; you run worker nodes (or use Fargate) |
| **Fargate** | Serverless compute engine for containers — works with both ECS and EKS, removing the need to provision/manage the underlying EC2 instances |

**ECS core concepts:** **Task Definition** (a blueprint, like a Pod spec — defines container image, CPU/memory, ports, env vars), **Task** (a running instance of a task definition), **Service** (maintains a desired number of running tasks, integrates with load balancers, similar to a Kubernetes Deployment).

```bash
aws ecs create-cluster --cluster-name my-cluster
aws ecs register-task-definition --cli-input-json file://task-def.json
aws ecs create-service --cluster my-cluster --service-name my-service \
  --task-definition my-task --desired-count 3 --launch-type FARGATE
```

**ECS vs EKS (very common interview question):** ECS is AWS-proprietary, simpler to learn/operate, tightly integrated with the AWS ecosystem, but locks you into AWS. EKS runs standard, portable Kubernetes (same API/tooling as any K8s cluster, including on-prem or other clouds) — more powerful and portable, but steeper learning curve and more operational complexity. Choose ECS for AWS-only simplicity; choose EKS if you need Kubernetes portability, are already invested in the K8s ecosystem, or need its more advanced orchestration features.

**EC2 launch type vs Fargate launch type (within ECS/EKS):** EC2 launch type runs tasks on EC2 instances you provision and manage (more control, potentially cheaper at scale, but you handle instance patching/scaling). Fargate is fully serverless — AWS manages the underlying compute entirely, you just specify CPU/memory per task — simpler operations at a per-task pricing premium.

---

## 14. Messaging & Integration

| Service | Type | Use Case |
|---|---|---|
| **SQS (Simple Queue Service)** | Message queue (pull-based) | Decouple producers/consumers, buffer work, ensure reliable delivery with retry |
| **SNS (Simple Notification Service)** | Pub/Sub messaging (push-based) | Fan-out a single message to multiple subscribers (email, SMS, Lambda, SQS queues) |
| **EventBridge** | Event bus | Route events between AWS services, SaaS apps, and custom applications based on rules; supports scheduled events (cron-like) |
| **Kinesis** | Real-time streaming data | High-throughput, ordered streaming data (e.g., clickstream analytics, log/metric ingestion, real-time processing) |

**SQS queue types:** **Standard** (at-least-once delivery, best-effort ordering, virtually unlimited throughput) vs **FIFO** (exactly-once processing, strict ordering, lower throughput limits).

**SQS vs SNS (very common interview question):** SQS is a **queue** — messages are pulled by consumers and persist until processed/deleted, naturally decoupling producer and consumer pace (good for buffering/load leveling and ensuring no message is lost if a consumer is temporarily down). SNS is **pub/sub** — messages are pushed immediately to all current subscribers (fan-out); it doesn't durably store/queue messages for offline subscribers on its own (though SNS commonly fans out to SQS queues to combine both patterns — durable buffering for each individual consumer).

**Dead Letter Queue (DLQ):** A separate queue where messages that repeatedly fail processing (exceeding a configured retry count) are automatically routed — preventing a single problematic message from blocking the queue indefinitely and enabling later inspection/reprocessing.

**Kinesis vs SQS (interview point):** Kinesis is built for high-throughput, ordered, **replayable** streaming data — multiple consumers can independently read the same stream at their own pace, and data persists for a configurable retention window (replay capability). SQS is built for **work distribution** — once a consumer processes and deletes a message, it's gone; it's not designed for multiple independent consumers to each see every message (without additional fan-out via SNS).

---

## 15. Step Functions

A serverless orchestration service for coordinating multiple AWS services (especially Lambda functions) into visual, stateful **workflows** defined as state machines (using Amazon States Language, JSON-based).

```json
{
  "StartAt": "ProcessOrder",
  "States": {
    "ProcessOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:ProcessOrder",
      "Next": "CheckInventory"
    },
    "CheckInventory": {
      "Type": "Choice",
      "Choices": [
        {"Variable": "$.inStock", "BooleanEquals": true, "Next": "ShipOrder"}
      ],
      "Default": "BackorderNotify"
    },
    "ShipOrder": { "Type": "Task", "Resource": "...", "End": true },
    "BackorderNotify": { "Type": "Task", "Resource": "...", "End": true }
  }
}
```

**Why use it instead of chaining Lambdas manually:** Built-in error handling/retries per step, visual execution history for debugging, support for parallel branches, wait states, human-approval-style pauses, and avoiding the complexity/cost of one Lambda function awaiting/orchestrating another (which wastes execution time billing while waiting).

---

## 16. Networking Recap

*(Full detail in the Cloud Networking notes — quick reference here.)*
- **VPC** — your isolated virtual network; public/private subnets determine internet reachability
- **Security Groups** (stateful, instance-level) vs **NACLs** (stateless, subnet-level)
- **ALB** (L7, content-based routing) vs **NLB** (L4, extreme performance/static IP) vs **Internet/NAT Gateway**
- **Route 53** — DNS + advanced routing policies (latency-based, geolocation, failover, weighted)
- **CloudFront** — CDN, edge caching, integrates with WAF and Lambda@Edge for edge compute
- **VPC Peering / Transit Gateway** — connecting multiple VPCs
- **Direct Connect** — dedicated private connectivity to on-premises

---

## 17. CloudWatch — Monitoring & Logging

**CloudWatch** is AWS's native monitoring and observability service.

| Component | Purpose |
|---|---|
| **Metrics** | Numeric time-series data (CPU utilization, request count, custom application metrics) |
| **Logs** | Centralized log collection from EC2, Lambda, ECS, and other services (via the CloudWatch Logs agent or native integration) |
| **Alarms** | Trigger actions (SNS notification, Auto Scaling action, etc.) when a metric crosses a threshold |
| **Dashboards** | Custom visual views combining multiple metrics |
| **Events / EventBridge** | (See messaging section) react to operational changes/schedules |
| **CloudWatch Logs Insights** | Query language for searching/analyzing log data |

```bash
aws cloudwatch put-metric-alarm --alarm-name high-cpu \
  --metric-name CPUUtilization --namespace AWS/EC2 \
  --statistic Average --period 300 --threshold 80 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:...:my-topic

aws logs tail /aws/lambda/myFunction --follow
```

**Custom metrics:** Applications can publish their own business/application-level metrics (e.g., "orders processed per minute") via the CloudWatch API/SDK, not just infrastructure-level metrics.

---

## 18. CloudTrail & Config

| Service | Purpose |
|---|---|
| **CloudTrail** | Records **every API call** made in your AWS account (who did what, when, from where) — the foundational audit/compliance log; essential for security investigations |
| **AWS Config** | Tracks **configuration changes** to resources over time, and can evaluate resources against compliance rules (e.g., "flag any S3 bucket that becomes publicly accessible") |

**CloudTrail vs CloudWatch Logs (interview point):** CloudTrail logs **API activity** (management-plane actions — who called `CreateBucket`, who modified a security group). CloudWatch Logs typically captures **application/system-level output** (your app's log lines, Lambda execution logs). They're complementary — CloudTrail answers "who changed this configuration," CloudWatch Logs answers "what did my application do."

---

## 19. Security Services

| Service | Purpose |
|---|---|
| **KMS (Key Management Service)** | Managed creation/control of encryption keys, integrates natively with most AWS services for encryption at rest |
| **Secrets Manager** | Securely store, rotate, and retrieve secrets (DB credentials, API keys) — supports automatic rotation |
| **Systems Manager Parameter Store** | Simpler, often free/cheaper alternative to Secrets Manager for configuration values and secrets (without automatic rotation by default) |
| **WAF (Web Application Firewall)** | L7 filtering against common web exploits (SQL injection, XSS), deployed in front of CloudFront/ALB/API Gateway |
| **Shield** | DDoS protection — Standard (free, automatic, basic) vs Advanced (paid, more sophisticated mitigation + cost protection) |
| **GuardDuty** | Managed threat detection — analyzes CloudTrail, VPC Flow Logs, DNS logs for malicious/anomalous activity |
| **Inspector** | Automated vulnerability scanning for EC2 instances and container images |
| **Macie** | Discovers and protects sensitive data (PII) stored in S3 using ML |
| **ACM (Certificate Manager)** | Free, managed SSL/TLS certificates for use with CloudFront, ALB, API Gateway |

**Secrets Manager vs Parameter Store (interview point):** Secrets Manager offers built-in automatic rotation (e.g., rotating a database password on a schedule, with Lambda-based rotation logic) and is purpose-built for secrets, at a per-secret cost. Parameter Store is more general-purpose (configuration + secrets), cheaper (a free tier covers standard parameters), but lacks native automatic rotation — many teams use Parameter Store for simple config and Secrets Manager specifically when rotation is required.

---

## 20. Cost Management

| Tool | Purpose |
|---|---|
| **Cost Explorer** | Visualize and analyze spending trends over time, by service/tag/account |
| **AWS Budgets** | Set spend/usage thresholds and get alerted when approaching/exceeding them |
| **Cost and Usage Report (CUR)** | Most granular, comprehensive billing data export for detailed analysis |
| **Trusted Advisor** | Automated recommendations across cost optimization, security, performance, fault tolerance |
| **Compute Optimizer** | ML-based recommendations for right-sizing EC2/EBS/Lambda based on actual usage patterns |

**Common cost levers:** Reserved Instances/Savings Plans for steady-state workloads, Spot Instances for fault-tolerant workloads, S3 lifecycle policies to move cold data to cheaper tiers, right-sizing instances based on actual utilization, deleting unused/orphaned resources (unattached EBS volumes, idle load balancers, old snapshots), and being mindful of data transfer costs (cross-AZ, cross-region, NAT Gateway processing — see Cloud Networking notes).

---

## 21. AWS Well-Architected Framework

A set of six pillars AWS recommends evaluating any architecture against — a very common interview/system-design topic.

| Pillar | Focus |
|---|---|
| **Operational Excellence** | Running and monitoring systems to deliver business value, continuously improving processes |
| **Security** | Protecting data, systems, and assets through risk assessment and mitigation |
| **Reliability** | Ability to recover from failures, scale to meet demand, and consistently perform as expected |
| **Performance Efficiency** | Using computing resources efficiently, adapting as requirements/technology evolve |
| **Cost Optimization** | Avoiding unnecessary costs, understanding spending over time |
| **Sustainability** | Minimizing environmental impact of running workloads |

This framework is commonly referenced in system-design interviews as a structured way to evaluate trade-offs in any proposed architecture.

---

## 22. Infrastructure as Code on AWS

| Tool | Description |
|---|---|
| **CloudFormation** | AWS-native IaC, JSON/YAML templates describing the full stack of resources |
| **AWS CDK (Cloud Development Kit)** | Define infrastructure using real programming languages (TypeScript, Python, Java) that synthesize into CloudFormation templates |
| **Terraform** | Third-party, multi-cloud IaC (see separate Terraform notes) — widely used as an AWS-agnostic alternative to CloudFormation |
| **SAM (Serverless Application Model)** | CloudFormation extension simplifying serverless (Lambda/API Gateway/DynamoDB) resource definitions |

```yaml
# Minimal CloudFormation example
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-unique-bucket-name
```

**CloudFormation vs Terraform (interview point):** CloudFormation is AWS-native, deeply integrated (immediate support for new AWS features), and free — but AWS-only. Terraform is cloud-agnostic (manages AWS, Azure, GCP, and hundreds of other providers with one consistent workflow/syntax), often preferred in multi-cloud organizations or by teams wanting one tool across their entire infrastructure, with a typically large and active community-driven module ecosystem.

---

## 23. CI/CD on AWS

| Service | Purpose |
|---|---|
| **CodeCommit** | Managed Git repository hosting (note: AWS has been steering customers toward GitHub/GitLab/Bitbucket integrations for source control in recent years) |
| **CodeBuild** | Managed build service — compiles code, runs tests, produces artifacts |
| **CodeDeploy** | Automates application deployment to EC2, on-prem, Lambda, or ECS, supporting blue/green and rolling deployment strategies |
| **CodePipeline** | Orchestrates the full release pipeline — source → build → test → deploy stages |

**Deployment strategies (interview-relevant, applies broadly beyond just AWS):**
| Strategy | Description |
|---|---|
| **In-place/Rolling** | Gradually replace instances with the new version |
| **Blue/Green** | Run the new version ("green") alongside the old ("blue") fully, then switch traffic over (e.g., via load balancer or DNS) — enables instant rollback by switching back |
| **Canary** | Route a small percentage of traffic to the new version first, gradually increasing if healthy |

---

## 24. High Availability & Disaster Recovery

**HA principles:** Deploy across multiple AZs (minimum), use managed services with built-in Multi-AZ support (RDS Multi-AZ, ELB spans AZs natively), design for automatic failover and self-healing (Auto Scaling Groups, health checks).

**DR strategies (ordered by cost vs recovery speed trade-off — common interview topic):**
| Strategy | RTO/RPO | Cost | Description |
|---|---|---|---|
| **Backup & Restore** | Hours | Lowest | Just keep backups (snapshots) in another region; restore on disaster |
| **Pilot Light** | Minutes-Hours | Low | Core infrastructure (e.g., a minimal database) always running in DR region, scale up the rest on failover |
| **Warm Standby** | Minutes | Medium | A scaled-down, fully functional version of the full environment always running in the DR region, scaled up on failover |
| **Multi-Site (Active-Active)** | Near-zero | Highest | Full production environment running simultaneously in multiple regions, handling live traffic in both |

**RTO vs RPO (very common interview definition question):** **RTO (Recovery Time Objective)** — how long can the system be down before it's restored (a *time* measure). **RPO (Recovery Point Objective)** — how much data loss (measured in time) is acceptable, i.e., how far back does your most recent usable backup go. A lower RTO/RPO requires more investment in redundancy/replication infrastructure.

---

## 25. AWS CLI & SDK Basics

```bash
aws configure                  # set up access key, secret key, default region/output format
aws sts get-caller-identity      # verify which identity you're currently authenticated as
aws s3 ls                          # example command
aws ec2 describe-instances --query 'Reservations[].Instances[].InstanceId' --output table

# Using named profiles for multiple accounts/roles
aws configure --profile prod
aws s3 ls --profile prod

# Assuming a role
aws sts assume-role --role-arn arn:aws:iam::123456789:role/MyRole --role-session-name session1
```

**SDKs** (boto3 for Python, AWS SDK for JS/Java/Go/etc.) let applications interact with AWS APIs programmatically — the same underlying APIs the CLI and console use.

```python
import boto3
s3 = boto3.client('s3')
s3.upload_file('local.txt', 'my-bucket', 'remote.txt')
```

---

## 26. Troubleshooting

| Symptom | Common cause / where to look |
|---|---|
| EC2 instance unreachable | Security group rules, NACL, route table, instance status checks (`aws ec2 describe-instance-status`) |
| `AccessDenied` errors | Missing IAM permission — check the policy attached to the role/user; use IAM Policy Simulator |
| Lambda timing out | Function timeout setting too low, or a downstream dependency (DB, API) is slow — check CloudWatch Logs for the function |
| High Lambda cold-start latency | Consider Provisioned Concurrency, reduce package size, choose a faster-starting runtime |
| S3 `403 Forbidden` | Bucket policy, IAM policy, Block Public Access settings, or object ACL misconfiguration |
| RDS connection refused | Security group not allowing the app's source, RDS not in the same/reachable VPC/subnet, or instance not publicly accessible when it needs to be |
| Unexpected high bill | Check Cost Explorer by service; common culprits: NAT Gateway data processing, cross-AZ transfer, forgotten/idle resources, unattached EBS volumes |

```bash
aws ec2 describe-instance-status --instance-ids i-xxxx
aws logs tail /aws/lambda/myFunction --since 1h
aws iam simulate-principal-policy --policy-source-arn <ARN> --action-names s3:GetObject --resource-arns <ARN>
```

---

## 27. Best Practices Summary

- Never use the root account for day-to-day work; enable MFA on it and lock away its credentials
- Use IAM roles instead of long-lived access keys wherever possible
- Apply least-privilege IAM policies; review with Access Analyzer periodically
- Enable CloudTrail and centralize logs for audit/security visibility
- Use Multi-AZ deployments for anything production-critical
- Tag resources consistently for cost allocation and organization
- Use Infrastructure as Code (CloudFormation/CDK/Terraform) rather than manual console changes
- Set up billing alarms/budgets to catch unexpected cost spikes early
- Encrypt data at rest (KMS) and in transit (TLS) by default
- Use Auto Scaling + health checks for self-healing infrastructure
- Regularly review the Well-Architected Framework pillars against your architecture
- Use Secrets Manager/Parameter Store instead of hardcoding credentials anywhere

---

## 28. Cheat Sheet

```bash
# IAM
aws iam create-user / create-role / attach-user-policy / list-attached-user-policies

# EC2
aws ec2 run-instances / describe-instances / start-instances / stop-instances / terminate-instances

# S3
aws s3 cp / sync / ls / rm / mb / rb

# RDS
aws rds create-db-instance / describe-db-instances / create-db-snapshot

# DynamoDB
aws dynamodb create-table / put-item / get-item / query / scan

# Lambda
aws lambda create-function / invoke / update-function-code

# CloudWatch
aws cloudwatch put-metric-alarm / get-metric-statistics
aws logs tail /aws/lambda/fn --follow

# CLI setup
aws configure
aws sts get-caller-identity
```

---

## 29. Interview Questions & Answers

**Q1: What's the difference between an IAM user and an IAM role?**
A: A user has long-term credentials (password and/or access keys) tied to a specific identity. A role has no long-term credentials at all — it's temporarily *assumed* (via STS) by a user, application, or AWS service, generating short-lived credentials. Roles are the recommended way to grant permissions to EC2 instances, Lambda functions, or cross-account access, avoiding the need to store and rotate long-lived secrets.

**Q2: Explain the difference between Multi-AZ and Read Replicas in RDS.**
A: Multi-AZ is purely for high availability — a synchronously replicated standby in another AZ that RDS automatically fails over to on a primary failure, never serving read traffic itself. Read Replicas are for read scalability — asynchronously replicated copies that actively serve read queries to offload the primary, with some replication lag, and require manual promotion to take over as a writable primary.

**Q3: What's the difference between EBS, EFS, and S3?**
A: EBS is block storage attached to a single EC2 instance at a time, AZ-bound — like a virtual hard drive. EFS is a managed NFS file system that multiple instances across multiple AZs can mount simultaneously — like a shared network drive. S3 is object storage accessed via API, designed for storing/retrieving discrete objects at massive scale, not mountable as a traditional POSIX filesystem.

**Q4: When would you choose DynamoDB over RDS?**
A: Choose DynamoDB when you need massive horizontal scale with consistent low-latency performance and your access patterns are well-known and simple (key-based lookups, limited query flexibility needed) — it scales seamlessly without capacity planning headaches. Choose RDS when you need complex queries, joins across multiple tables, strict relational integrity/ACID transactions, or ad-hoc querying flexibility that a NoSQL key-value model doesn't support well.

**Q5: What's the difference between SQS and SNS?**
A: SQS is a queue — messages are pulled by consumers and persist until processed, naturally decoupling producer/consumer pace and buffering load. SNS is pub/sub — messages are pushed immediately to all current subscribers (fan-out) and aren't durably queued for later/offline subscribers by itself. A common pattern combines both: SNS fans out to multiple SQS queues, giving each consumer its own durable, independently-paced buffer.

**Q6: Explain cold starts in Lambda and how to mitigate them.**
A: A cold start is the added latency when Lambda must initialize a brand-new execution environment (download code, start the runtime, run any initialization code) before handling an invocation, versus reusing an already-warm environment for subsequent invocations. Mitigations include using Provisioned Concurrency (keeps environments pre-warmed), minimizing deployment package size, choosing faster-starting runtimes, and avoiding heavy initialization logic outside the handler when possible.

**Q7: What's the difference between ECS and EKS?**
A: ECS is AWS's own proprietary container orchestrator — simpler to learn and tightly integrated with the AWS ecosystem, but locks you into AWS-specific tooling/APIs. EKS runs standard, portable Kubernetes — more powerful and portable across clouds/on-prem with the same K8s API and tooling everyone already knows, but with a steeper learning curve and more operational complexity. Both support Fargate for serverless container compute, removing the need to manage underlying EC2 instances.

**Q8: What are the AWS Well-Architected Framework's six pillars?**
A: Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, and Sustainability — a structured framework AWS recommends using to evaluate trade-offs in any cloud architecture, commonly referenced in system-design discussions/interviews.

**Q9: What's the difference between RTO and RPO in disaster recovery?**
A: RTO (Recovery Time Objective) is how long the system can be down before it must be restored — a time-to-recovery measure. RPO (Recovery Point Objective) is how much data loss is acceptable, measured as how far back in time your most recent usable backup/replica goes. Lower RTO/RPO targets require more investment in redundancy and replication infrastructure (e.g., moving from simple backup-and-restore toward warm standby or multi-site active-active).

**Q10: What is the principle of least privilege, and how do you apply it with IAM?**
A: Granting only the specific permissions an identity actually needs to perform its job, nothing more — starting from no access and explicitly adding required allows, rather than starting broad and trying to restrict later. In practice: write narrowly-scoped IAM policies (specific actions and resource ARNs rather than wildcards), use roles instead of long-lived credentials, regularly audit with IAM Access Analyzer, and use permissions boundaries/SCPs as additional guardrails at scale.

**Q11: How does Auto Scaling work, and what's a target tracking policy?**
A: An Auto Scaling Group maintains a desired number of EC2 instances (between configured min/max bounds) by launching or terminating instances automatically, using a launch template to define what new instances look like. A target tracking scaling policy automatically adjusts the desired capacity to keep a chosen metric (like average CPU utilization) at a specified target value — the simplest and most commonly used scaling policy type, conceptually similar to a thermostat.

**Q12: What's the difference between a Security Group and S3 bucket policy in terms of how they grant access — and what is a resource-based policy?**
A: A Security Group is an identity-agnostic network-level firewall controlling traffic to a resource by IP/port. An S3 bucket policy is a **resource-based IAM policy** — attached directly to the resource (the bucket) rather than to a user/role, defining which principals (which can include users from *other* AWS accounts) can perform which actions on it. Resource-based policies are one of the few ways to grant cross-account access without needing the other account to assume a role in yours.

**Q13: What's the difference between CloudTrail and CloudWatch?**
A: CloudTrail records API-level activity across your AWS account — who called which API action, when, and from where — primarily for audit, security investigation, and compliance. CloudWatch is the broader monitoring/observability service, covering metrics, alarms, dashboards, and application/system-level logs (e.g., your Lambda function's print statements or an EC2 instance's application logs). CloudTrail tells you who changed something; CloudWatch tells you how your systems are performing and what they're outputting.

**Q14: Explain blue/green deployment versus canary deployment.**
A: Blue/green runs the new version ("green") fully alongside the old ("blue"), then switches all traffic over at once (e.g., via load balancer or DNS swap) — enabling instant rollback by switching back if something goes wrong. Canary deployment gradually shifts a small percentage of traffic to the new version first, monitoring for issues, and progressively increasing that percentage if it remains healthy — reducing blast radius of a bad deployment compared to switching all traffic at once.

**Q15: What's the difference between a Spot Instance and an On-Demand Instance, and what's the risk of using Spot?**
A: On-Demand instances are billed at standard rates with no commitment and are never reclaimed by AWS. Spot Instances bid on unused EC2 capacity at up to ~90% discount, but AWS can reclaim them with only a 2-minute warning when it needs that capacity back for On-Demand customers — so Spot is only suitable for fault-tolerant, flexible, or stateless workloads (batch processing, CI runners) that can handle sudden interruption, not for workloads requiring guaranteed continuous availability.

**Q16: How would you design a highly available, scalable web application architecture on AWS?**
A: Deploy across at least two Availability Zones; use an Application Load Balancer distributing traffic to an Auto Scaling Group of stateless application instances (or containers on ECS/EKS with Fargate); use RDS Multi-AZ (or Aurora) for the database with read replicas if read-heavy; use S3 + CloudFront for static assets; use ElastiCache to reduce database load for frequently-accessed data; put a WAF in front for application-layer protection; use Route 53 health checks for DNS-level failover if multi-region; and monitor everything with CloudWatch alarms feeding into SNS notifications.

**Q17: What's the difference between Lambda and a container running on Fargate?**
A: Lambda is event-driven, fully serverless with automatic per-millisecond billing, scales to zero with no idle cost, but has a hard 15-minute execution limit and can experience cold starts. Fargate runs containers without managing underlying servers, but typically for longer-running or always-on services (e.g., a continuously running API), without Lambda's strict execution time cap, and generally with a more predictable (though potentially higher when idle) cost profile for steady workloads compared to Lambda's pure pay-per-invocation model.

**Q18: What is the purpose of a Dead Letter Queue (DLQ)?**
A: A separate queue that messages are automatically routed to after they've failed processing a configured number of times — preventing a single repeatedly-failing message from blocking or endlessly retrying against the main queue, and allowing engineers to later inspect and potentially reprocess those failed messages after fixing the underlying issue.

**Q19: How does AWS billing work for data transfer, and what's a common cost surprise?**
A: Data transfer **into** AWS is generally free; data transfer **out** to the internet is charged and is often one of the largest networking line items on a bill. A very common surprise is cross-AZ data transfer charges (traffic between Availability Zones within the same region is billed, even though it feels "internal"), and NAT Gateway per-GB data processing charges in addition to the underlying data transfer cost — both are easy to overlook during architecture design but can add up significantly at scale.

**Q20: What's the difference between CloudFormation and Terraform?**
A: CloudFormation is AWS-native IaC — free, deeply integrated with AWS (typically gets day-one support for new AWS features), but AWS-only. Terraform is a third-party, cloud-agnostic IaC tool supporting AWS, Azure, GCP, and hundreds of other providers through one consistent workflow and syntax (HCL) — often preferred by teams managing multi-cloud environments or wanting a single IaC tool across their entire infrastructure, backed by a large community module ecosystem.

---

### Final interview tip
Be ready to **design a basic 3-tier highly-available architecture on a whiteboard** (ALB → ASG of app servers across 2+ AZs → RDS Multi-AZ), clearly explain the **trade-offs between serverless (Lambda) and container/EC2-based compute**, and confidently distinguish **SQS vs SNS** and **Multi-AZ vs Read Replicas** — these come up constantly. Also expect at least one cost-optimization or disaster-recovery scenario question (RTO/RPO trade-offs), since AWS interviews often blend technical depth with practical operational/cost judgment.
