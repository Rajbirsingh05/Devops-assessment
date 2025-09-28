# Devops-assessment - End to end Summary of the assessment  
## Assumptions & consideration 
-  Two natGatway deployed per AZ, deviated from architecture 
-  ALB acess log setup, storage destination is s3
-  SSL confifured , used ACM service to generate public certificate
-  Entire network like VPC, subnet, IG, nat gateway, route table and associates created using terraform
-  Rest everything setup by using console 

## Network & Connectivity

- VPC created with CIDR block `10.0.0.0/16`
- Internet Gateway attached to the VPC
- 2 Elastic IPs provisioned for NAT Gateways (1 per AZ for HA)
- 6 Subnets:
  - 2 Public Subnets
  - 2 Private Application Subnets
  - 2 Private Database Subnets  
  *(Spread across `us-west-1a` and `us-west-1c`)*
- 2 NAT Gateways deployed in public subnets
- 3 Route Tables:
  - Public Route Table
  - Application Route Table (with NAT)
  - Database Route Table (no NAT)
- Route Table Associations:
  - Public subnets → Public route
  - Application subnets → NAT route
  - Database subnets → DB route (no internet)
- 4 Security Groups:
  - Database SG (allows DB port from Application SG only)
  - Application SG (allows port 80 & 443 from ALB SG)
  - Bastion Host SG ( allow SSH to private network)
- DNS Mapping:
  - A record created in existing hosted zone
  - Alias enabled → Routed traffic to ALB
  - Simple routing policy applied

---

## Database Setup (RDS MySQL)

- Multi-AZ RDS MySQL instance deployed
- DB Subnet Group created using private subnets (no internet)
- ECS application accesses DB via AWS Secrets Manager
- Secrets Manager handles password generation, rotation, and encryption
- Volume autoscaling enabled
- Automatic backups enabled (7-day retention)
- Detailed monitoring enabled:
  - Audit log
  - Error log
  - General log
  - Slow query log
- Maintenance window set to 00:00 (non-peak hours)

---

## Image Upload to ECR

- SSH into Bastion Host → SSH into private EC2
- Pulled nginx image using `docker pull nginx`
- Logged into ECR for authentication using ECR login command 
- Verified permissions for image push ( IAM  role attached)
- Tagged and pushed image using ECR "View Push Command" instructions
- iamge is encrpted using default KMS key 

---

## Application Load Balancer & Target Groups

- Internet-facing ALB created
- 2 public subnets attached for HA
- HTTP (80) and HTTPS (443) listeners added
- ACM public certificate attached
- ALB SG allows port 80/443
- Application SG allows traffic from ALB SG
- Target Group created:
  - Target type: IP address

---

## ECS Cluster, Task Definition & Service

- ECS Cluster created (EC2 launch type)
- Auto Scaling Group provisioned:
  - Instance type: `t3.medium`
  - Min: 2, Max: 3
- SSH key pair created
- Private application subnets and Application SG selected
- Cluster-level monitoring enabled (CPU, memory, network I/O)
- Task Definition created:
  - Launch type: EC2
  - Execution role: ECR pull + Secrets Manager access
  - Secrets injected via environment variables
  - Example: `os.environ.get("DB_PASSWORD")` in Python
  - Task-level CloudWatch metrics enabled
- ECS Service created:
  - Task Definition and Cluster selected
  - Replica count: 2
  - Load Balancing enabled → ALB, listener, target group selected
  - No volume added (not required for this assessment)
- Tasks deployed and running successfully

---

## Application Access

- Accessible via DNS record in Route 53
- Alternatively, use the ALB DNS name



 
