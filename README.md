# Enterprise AWS 3-Tier Architecture

A hands-on AWS portfolio project demonstrating the design, deployment, security, and troubleshooting of a multi-AZ three-tier cloud architecture.

## Project Overview

This project implements an enterprise-style AWS environment with separate public, application, and database tiers. The application tier runs across two Availability Zones behind an Application Load Balancer, while the database tier uses a private Amazon RDS MySQL instance. Administrative access is handled through AWS Systems Manager rather than public IP addresses, and database credentials are stored in AWS Secrets Manager.

## Architecture

**Region:** US East (Ohio) — `us-east-2`

The environment includes:

- Custom VPC using `10.10.0.0/16`
- Six workload subnets across two Availability Zones
- Two public subnets for the internet-facing load-balancing tier
- Two private application subnets
- Two private database subnets
- Internet Gateway for public-tier connectivity
- NAT Gateway for outbound connectivity from private application resources
- Dedicated route tables for public, application, and database tiers
- Application Load Balancer
- Target group with two healthy EC2 application servers
- Two Amazon Linux EC2 application servers distributed across AZ1 and AZ2
- Private Amazon RDS MySQL database
- AWS Secrets Manager for database credentials
- IAM role and AWS Systems Manager Session Manager for private EC2 administration
- Security groups enforcing tier-to-tier access

## Traffic Flow

```text
Internet
   |
   v
Application Load Balancer
   |
   +-------------------+
   |                   |
   v                   v
EC2 App Server AZ1   EC2 App Server AZ2
   |                   |
   +---------+---------+
             |
             v
        RDS MySQL
             |
             v
     Private DB Tier
```

The Application Load Balancer distributes HTTP requests between application servers in separate Availability Zones. The application servers have no public IPv4 addresses. Database access is restricted to the application tier, and the RDS instance is not publicly accessible.

## AWS Services Used

| Service | Purpose |
|---|---|
| Amazon VPC | Network isolation and subnet segmentation |
| EC2 | Application servers |
| Application Load Balancer | Distributes application traffic across AZs |
| Target Groups | Health monitoring and backend registration |
| Amazon RDS for MySQL | Managed relational database |
| NAT Gateway | Outbound internet access for private app servers |
| Internet Gateway | Internet connectivity for public resources |
| IAM | Instance permissions and least-privilege access |
| AWS Systems Manager | Secure EC2 administration without public SSH exposure |
| AWS Secrets Manager | Secure storage/retrieval of database credentials |
| Security Groups | Stateful network access control |

## Network Segmentation

```text
Public Tier
10.10.1.0/24  - AZ1
10.10.2.0/24  - AZ2

Application Tier
10.10.11.0/24 - AZ1
10.10.12.0/24 - AZ2

Database Tier
10.10.21.0/24 - AZ1
10.10.22.0/24 - AZ2
```

This design separates internet-facing infrastructure from application workloads and database resources.

## Security Design

- The ALB security group accepts application traffic and forwards HTTP traffic to the application tier.
- The application security group allows application traffic from the ALB security group rather than exposing EC2 instances directly to the internet.
- The EC2 instances do not require public IPv4 addresses.
- The database security group allows MySQL TCP/3306 from the application security group.
- RDS is configured as not publicly accessible.
- AWS Systems Manager Session Manager provides management access to private EC2 instances.
- The EC2 IAM role includes `AmazonSSMManagedInstanceCore` and a scoped policy for retrieving the required RDS secret.

## Secrets Management

Database credentials were moved from application configuration into AWS Secrets Manager. The application retrieves credentials at runtime using its IAM role, eliminating the need to hard-code the database password in application files.

## Database Validation

The project database contains an `app_requests` table with successful connection records from both application servers:

```text
CM-App-Server-AZ1 -> Database connection successful from AZ1
CM-App-Server-AZ2 -> Database connection successful from AZ2
```

This demonstrates successful application-to-database communication across the private architecture.

## Load Balancing Validation

Both application servers were registered with the target group and reached a **Healthy** state. Refreshing the Application Load Balancer endpoint demonstrated requests being served by both AZ1 and AZ2 application servers.

## Troubleshooting Case Study

`CM-App-Server-AZ2` initially failed its Application Load Balancer health checks.

### Symptoms and Investigation

1. The Amazon Linux package repository timed out during the original user-data execution.
2. Apache (`httpd`) therefore was not installed as expected.
3. After installing and starting Apache, the target returned HTTP `403` health-check responses.
4. The web root did not contain the expected application page.

### Resolution

I connected to the private instance using AWS Systems Manager, installed and enabled Apache, created the application page under `/var/www/html/`, verified HTTP service operation, and retested the target group.

The final target-group state showed:

```text
2 Total Targets
2 Healthy
0 Unhealthy
```

### Key Lesson

A successful EC2 instance status check does not guarantee application availability. Infrastructure health, operating-system health, web-service health, network reachability, and ALB application health are separate troubleshooting layers.

## Skills Demonstrated

- AWS VPC design and IPv4 subnetting
- Multi-AZ and public/private subnet architecture
- Route tables, Internet Gateway, and NAT Gateway
- EC2 deployment and Linux administration
- Application Load Balancer and target-group health checks
- Apache web server deployment
- Amazon RDS MySQL and database connectivity testing
- Security-group referencing and network segmentation
- IAM roles and least-privilege policies
- AWS Systems Manager Session Manager
- AWS Secrets Manager
- Cloud troubleshooting
- Three-tier application architecture

## Project Outcome

The finished environment demonstrates a functioning AWS three-tier architecture in which internet traffic enters through an Application Load Balancer, requests are distributed across two private application servers in separate Availability Zones, and the application tier communicates with a private RDS MySQL database. Database credentials are retrieved securely from AWS Secrets Manager, and private EC2 administration is performed through Systems Manager.

## Future Enhancements

Potential next iterations include HTTPS with AWS Certificate Manager, Route 53 DNS, Auto Scaling, CloudWatch alarms and dashboards, AWS WAF, RDS Multi-AZ deployment, infrastructure as code with Terraform or AWS CloudFormation, and CI/CD automation.

---

**Built as an AWS cloud/network engineering portfolio project.**