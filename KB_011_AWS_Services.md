Knowledge Base: AWS Services Deep Dive

Compute, Networking, Storage, Databases, Serverless, Security, Messaging & Production Architecture

⸻

PART 1: AWS GLOBAL INFRASTRUCTURE

1.1 Regions, AZs and Edge Locations

Region:
  Physical geographic area
Examples:
  us-east-1 → Virginia
  us-west-2 → Oregon
  eu-west-1 → Ireland
  ap-south-1 → Mumbai
Each region contains:
Region (Mumbai)
│
├── AZ-A
│    Multiple datacenters
│
├── AZ-B
│    Independent power/network
│
└── AZ-C
     Separate fault domain
Availability Zone Benefits:
  ✓ High availability
  ✓ Fault isolation
  ✓ Low latency (<2ms)
Edge Locations:
  Used by CloudFront
  400+ worldwide
Purpose:
  Cache content close to users
Example:
User (Japan)
     ↓
CloudFront Edge
     ↓
Origin S3 Bucket (Mumbai)
Result:
  Faster delivery

⸻

1.2 Shared Responsibility Model

AWS manages:
✓ Physical servers
✓ Networking
✓ Hypervisor
✓ Datacenter security
✓ Hardware failures
Customer manages:
✓ IAM permissions
✓ Application security
✓ Encryption
✓ Operating system (EC2)
✓ Patching (EC2)
✓ Data
Service Responsibility:
EC2:
  Customer manages OS
RDS:
  AWS manages DB server
Lambda:
  AWS manages everything except code
S3:
  AWS manages storage layer
The more managed the service,
the less operational burden.

⸻

PART 2: COMPUTE SERVICES

⸻

2.1 EC2 Deep Dive

Elastic Compute Cloud
Provides virtual machines.
Instance Types:
General Purpose:
  t3, t4g, m7i
Compute Optimized:
  c7g
Memory Optimized:
  r7g
Storage Optimized:
  i4i
GPU:
  p5
Lifecycle:
Launch EC2
     ↓
AMI selected
     ↓
Instance starts
     ↓
Application deployed
     ↓
Traffic served
Storage:
EBS Volume
Persistent
Instance Store
Ephemeral
Scaling:
Manual
Auto Scaling Group
Target:
CPU >70%
Add instance
CPU <30%
Remove instance
Benefits:
✓ Full control
✓ Custom software
✓ Long-running workloads
Use Cases:
Web servers
Microservices
Game servers
Legacy applications

⸻

2.2 Auto Scaling Group

ALB
 ↓
EC2 Instances
 ↓
ASG
Current:
4 instances
Traffic spikes
CPU=85%
ASG:
Launch 2 more
Now:
6 instances
Traffic drops
Terminate extra instances
Benefits:
✓ Elastic scaling
✓ Fault tolerance
✓ Cost optimization

⸻

2.3 Elastic Load Balancer

ALB (Application Load Balancer)

HTTP/HTTPS
Supports:
Path Routing
/api/* → Service A
/admin/* → Service B
Host Routing
api.company.com → API
shop.company.com → Shop
Features:
✓ SSL termination
✓ Sticky sessions
✓ WebSockets
✓ WAF integration

NLB

Layer 4
TCP/UDP
Millions of requests/sec
Ultra low latency
Use:
Gaming
Databases
High throughput

⸻

PART 3: STORAGE SERVICES

⸻

3.1 S3 Deep Dive

Object storage
Unlimited scale
11 9's durability
Bucket
 │
 ├── image1.jpg
 ├── file.pdf
 └── logs/
Storage Classes:
Standard
Frequently accessed
Standard IA
Less frequent
Glacier
Archive
Deep Archive
Cheapest
Lifecycle:
30 days
↓
IA
90 days
↓
Glacier
365 days
↓
Delete
Benefits:
✓ Durable
✓ Cheap
✓ Highly scalable

⸻

3.2 Versioning

file.txt v1
file.txt v2
file.txt v3
Delete file?
Actually creates:
Delete marker
Can restore old versions.
Protection against:
✓ Accidental deletion
✓ Ransomware
✓ Human error

⸻

3.3 EBS vs EFS

EBS

Attached to one EC2
Persistent disk
SSD/HDD
Use:
Database
Application storage

EFS

Shared filesystem
Multiple EC2 instances
NFS protocol
Use:
Shared content
Containers
ML workloads

⸻

PART 4: NETWORKING

⸻

4.1 VPC Architecture

VPC
│
├── Public Subnet
│     ALB
│     Bastion Host
│
├── Private Subnet
│     EC2
│     ECS
│
└── Database Subnet
      RDS
Internet Gateway
↓
Public subnet
NAT Gateway
↓
Private subnet internet access
Benefits:
✓ Isolation
✓ Security
✓ Control

⸻

4.2 Security Groups

Stateful firewall
ALB SG:
Allow 80,443
EC2 SG:
Allow 8080 from ALB SG
DB SG:
Allow 3306 from EC2 SG
Traffic Flow:
Internet
↓
ALB
↓
EC2
↓
RDS
Layered security

⸻

PART 5: DATABASE SERVICES

⸻

5.1 RDS Deep Dive

Managed relational database
Supports:
MySQL
PostgreSQL
MariaDB
Oracle
SQL Server
Features:
Backups
Patching
Replication
Monitoring
Multi-AZ:
Primary
↓
Synchronous replication
↓
Standby
Primary fails
Automatic failover
Benefits:
✓ HA
✓ Managed operations
✓ Backups

⸻

5.2 Aurora

Cloud native database
Storage separated from compute
6 copies across 3 AZs
Writer Node
│
├── Reader 1
├── Reader 2
└── Reader 3
Read scaling
Failover:
30 seconds
5x MySQL performance
Use:
Mission-critical systems

⸻

5.3 DynamoDB

NoSQL
Partition Key
Sort Key
Example:
PK=user#123
SK=order#1
SK=order#2
Features:
Auto scaling
Global tables
Streams
TTL
Single digit ms latency
Use:
Gaming
IoT
Session store
High-scale APIs

⸻

PART 6: SERVERLESS

⸻

6.1 Lambda

Upload code
↓
Event occurs
↓
Lambda executes
↓
Returns response
Triggers:
API Gateway
S3
SNS
SQS
EventBridge
Benefits:
✓ No servers
✓ Auto scaling
✓ Pay per request
Problems:
Cold starts
Execution limits:
15 minutes
Use:
Microservices
Image processing
Automation

⸻

6.2 API Gateway

Client
 ↓
API Gateway
 ↓
Lambda
Features:
Authentication
Throttling
Caching
Monitoring
Supports:
REST
HTTP API
WebSocket

⸻

PART 7: CONTAINERS

⸻

7.1 ECS

AWS managed container orchestration
ALB
↓
ECS Cluster
│
├── Task 1
├── Task 2
└── Task 3
Launch Types:
EC2
Fargate
Benefits:
✓ Simpler than Kubernetes
✓ Native AWS integration

⸻

7.2 EKS

Managed Kubernetes
Control Plane
↓
Worker Nodes
Features:
Autoscaling
Service Mesh
Helm
Ingress
Use:
Large microservice ecosystems
Tradeoff:
More complex than ECS

⸻

PART 8: MESSAGING

⸻

8.1 SQS

Producer
↓
Queue
↓
Consumer
Decoupling
Visibility timeout
Dead Letter Queue
Standard Queue:
Unlimited throughput
At-least-once
FIFO Queue:
Ordered
Exactly-once processing
Use:
Async workloads
Background jobs

⸻

8.2 SNS

Publisher
↓
Topic
├── Email
├── Lambda
├── SQS
└── SMS
Pub/Sub
Fanout pattern
Use:
Notifications
Broadcast events

⸻

8.3 EventBridge

Producer
↓
Event Bus
↓
Rules
├── Lambda
├── ECS
└── SQS
Serverless event routing
Supports SaaS integrations
Use:
Event-driven architecture

⸻

PART 9: CACHING

⸻

9.1 ElastiCache

Redis

Application
↓
Redis Cache
↓
Database
Read latency:
Microseconds
Patterns:
Cache Aside
App
↓
Redis miss
↓
DB
↓
Redis update
Benefits:
✓ Faster reads
✓ Reduced DB load

⸻

PART 10: SECURITY

⸻

10.1 IAM

User
Role
Policy
Policy:
Allow S3 Read
Resource:
bucket/*
Actions:
GetObject
Principle:
Least privilege

⸻

10.2 KMS

Encryption key management
S3
EBS
RDS
Customer managed keys
Benefits:
✓ Secure encryption
✓ Rotation
✓ Auditing

⸻

10.3 Secrets Manager

Stores:
DB passwords
API keys
JWT secrets
Supports rotation
Application
↓
Secrets Manager
↓
Retrieve credentials

⸻

PART 11: MONITORING

⸻

11.1 CloudWatch

Metrics
CPU
Memory
Latency
Logs
Application logs
Alarms
CPU >80%
↓
SNS alert
Dashboard
System overview

⸻

11.2 X-Ray

Trace request flow
User
↓
API Gateway
↓
Lambda
↓
RDS
Find bottlenecks
Distributed tracing

⸻

PART 12: CDN

⸻

12.1 CloudFront

User
↓
Edge Location
↓
CloudFront Cache
↓
Origin S3
Benefits:
✓ Reduced latency
✓ DDoS protection
✓ Global delivery

⸻

PART 13: CI/CD

⸻

CodePipeline

GitHub
↓
CodeBuild
↓
CodeDeploy
↓
EC2/ECS/Lambda
Supports:
Blue-Green deployment
Canary deployment
Rollback automatically

⸻

PART 14: PRODUCTION ARCHITECTURE

Highly Available Web Application

Users
 ↓
CloudFront
 ↓
WAF
 ↓
ALB
 ↓
Auto Scaling Group
EC2 instances
 ↓
Redis Cache
 ↓
RDS Aurora
Logs → CloudWatch
Backups → S3
Secrets → Secrets Manager

⸻

Serverless Architecture

User
 ↓
API Gateway
 ↓
Lambda
 ↓
DynamoDB
Events
 ↓
EventBridge
 ↓
SQS
 ↓
Lambda Worker
Monitoring:
CloudWatch + X-Ray

⸻

Microservices Architecture

ALB
 ↓
ECS/EKS Cluster
 │
 ├── User Service
 ├── Order Service
 ├── Payment Service
 └── Inventory Service
Redis Cache
Aurora
SQS
SNS
CloudWatch
Secrets Manager

⸻

PART 15: COST OPTIMIZATION

EC2:
Reserved Instances
Savings Plans
S3:
Lifecycle policies
Lambda:
Pay per execution
CloudFront:
Reduce origin traffic
Auto Scaling:
Remove idle instances
Spot Instances:
70-90% cheaper
Monitor:
Cost Explorer
Budgets

⸻

KEY TAKEAWAYS

1. EC2 provides full control, Lambda provides serverless simplicity
2. S3 is the backbone of storage
3. RDS is managed SQL, DynamoDB is managed NoSQL
4. VPC + Security Groups provide network isolation
5. SQS decouples services, SNS broadcasts events
6. ECS is simpler, EKS provides Kubernetes flexibility
7. CloudWatch + X-Ray are essential for observability
8. Auto Scaling and ALB enable elasticity
9. CloudFront improves global performance
10. Design for HA across multiple AZs
11. Prefer managed services whenever possible
12. Monitor cost continuously

⸻

AWS is a collection of building blocks
