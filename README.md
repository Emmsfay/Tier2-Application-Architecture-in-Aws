<h1>Understanding Tier 2 Architecture</h2>

A "Tier 2" architecture typically refers to a mid-level service within a multi-tiered application. This could be an application server layer, a worker node layer, or a specialized data processing layer that interacts with a database (Tier 3) and is accessed by a web/load balancer layer (Tier 1). For an AWS VPC, this means setting up the necessary network isolation, routing, and compute resources.

**Core Components You'll Define in Terraform:**
**VPC (Virtual Private Cloud)**: Your isolated network in AWS.
Subnets:
- **Public Subnets**: For resources that need direct internet access (e.g., NAT Gateways, bastion hosts, public load balancers.

- **Private Subnets**: For your actual Tier 2 application instances, ensuring they are not directly accessible from the internet.
  
- **Internet Gateway (IGW)**: Allows communication between your VPC and the internet.
  
- **NAT Gateway**: Enables instances in private subnets to initiate outbound connections to the internet (e.g., for updates, downloading packages) without being publicly accessible.
 
- **Route Tables**: Control network traffic flow within your VPC and to/from the internet.
  
- **Security Groups**: Act as virtual firewalls to control inbound and outbound traffic for your instances.
  
- **EC2 Instances (or other compute)**: Your actual Tier 2 application servers.
  
- **Target Group & Application Load Balancer (ALB) (Optional but recommended)**: For distributing traffic to your Tier 2 instances and providing high availability.
  
### Terraform Structure:
It's good practice to organize your Terraform code into modules for reusability and maintainability.

```
.
├── provider.tf
├── variables.tf
├── vpc.tf
├── subnets.tf
├── nat.tf
├── routing.tf
├── security_groups.tf
├── ec2_tier2.tf
├── alb.tf
├── data.tf
├── outputs.tf
└── install_app.sh
```


<h1>Prerequisite</h1>

Have a Code Editor installed on your machine (e.g VS Code)

Have Terraform installed on your machine (Ubuntu distribution for this tutorial)

Basic understanding and working knowledge about different services like EC2, VPC, Security groups, Route53, Load Balancer, Auto Scaling groups and Amazon Certificate Manager.


<img width="1024" height="1024" alt="Generated Image January 05, 2026 - 1_42PM" src="https://github.com/user-attachments/assets/4c31e879-2c79-4946-b55e-76090c23dd47" />

