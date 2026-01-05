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



<h1>Prerequisite</h1>

Have a Code Editor installed on your machine (e.g VS Code)

Have Terraform installed on your machine (Ubuntu distribution for this tutorial)

Basic understanding and working knowledge about different services like EC2, VPC, Security groups, Route53, Load Balancer, Auto Scaling groups and Amazon Certificate Manager.


### STEP 1: Configuring Keys

Terraform needs IAM Access and secret access keys in order to connect to AWS and utilize the services. You can either establish a new IAM user or supply a role (we'll be defining an IAM user) and generate the keys for this tutorial. Get the keys by downloading them.

Check the profile for that key when it has been downloaded. The profile will often be [default] and it will be saved in the following directory.

```
cd home/.aws/credentials
```

### STEP 2: Folder Structure

Make two subdirectories called main and modules in the working directory. The modules will hold the individual modules for every service, whereas the main will hold our primary configuration file.



Now, make three files inside the main folder: `terraform.tfvars` to store the variables, and `main.tf` to configure the `modules.tf` for the variables' declaration.

Additionally, each module will have `main.tf` for the variables and module setup.tf to declare the outputs and variables needed in that `module.tf` that are suitable for usage as module variables.

Within the primary directory:

`Main.tf`

```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "us-east-1"
}

# --- VPC ---
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.environment}-vpc"
  }
}

# --- Internet Gateway ---
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-igw"
  }
}

# --- Public Subnets (e.g., for NAT Gateways, ALBs) ---
resource "aws_subnet" "public" {
  count                   = length(var.availability_zones)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr_block, 8, count.index) # Example: /24 subnets from a /16 VPC
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true # Required for NAT Gateway in public subnet

  tags = {
    Name = "${var.environment}-public-subnet-${count.index}"
  }
}

# --- Private Subnets (for Tier 2 application instances) ---
resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr_block, 8, count.index + length(var.availability_zones)) # Offset for private subnets
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.environment}-private-subnet-${count.index}"
  }
}

# --- Elastic IPs for NAT Gateways ---
resource "aws_eip" "nat" {
  count = length(var.availability_zones)
  vpc   = true # Associate with VPC

  tags = {
    Name = "${var.environment}-nat-eip-${count.index}"
  }
}

# --- NAT Gateways ---
resource "aws_nat_gateway" "main" {
  count         = length(var.availability_zones) # One NAT Gateway per AZ for resilience
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${var.environment}-nat-gateway-${count.index}"
  }
}

# --- Route Table for Public Subnets ---
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.environment}-public-rt"
  }
}

# --- Associate Public Subnets with Public Route Table ---
resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# --- Route Tables for Private Subnets (each with its own NAT Gateway) ---
resource "aws_route_table" "private" {
  count  = length(var.availability_zones)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "${var.environment}-private-rt-${count.index}"
  }
}

# --- Associate Private Subnets with Private Route Tables ---
resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id # Each private subnet uses the NAT in its AZ
}

# --- Security Group for Tier 2 Instances ---
resource "aws_security_group" "tier2_sg" {
  vpc_id = aws_vpc.main.id
  name   = "${var.environment}-tier2-sg"
  description = "Security group for Tier 2 application instances"

  # Example: Allow inbound traffic from the ALB security group (Tier 1)
  ingress {
    from_port       = 8080 # Example application port
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id] # Reference ALB SG (defined elsewhere)
    description     = "Allow HTTP/app traffic from ALB"
  }

  # Example: Allow SSH from a bastion host (Tier 1 utility)
  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion_sg.id] # Reference Bastion SG (defined elsewhere)
    description     = "Allow SSH from Bastion Host"
  }

  # Allow all outbound traffic for updates/dependencies
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1" # All protocols
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.environment}-tier2-sg"
  }
}

# --- EC2 Instances for Tier 2 ---
resource "aws_instance" "tier2_app" {
  count         = var.tier2_instance_count
  ami           = data.aws_ami.ubuntu.id # Example: Ubuntu AMI
  instance_type = var.tier2_instance_type
  subnet_id     = element(aws_subnet.private[*].id, count.index % length(aws_subnet.private)) # Distribute across private subnets
  vpc_security_group_ids = [aws_security_group.tier2_sg.id]
  key_name      = var.ssh_key_pair
  user_data     = file("install_app.sh") # Example: Script to install your app

  tags = {
    Name        = "${var.environment}-tier2-app-${count.index}"
    Environment = var.environment
    Tier        = "2"
  }
}

# --- ALB, Target Group (for Tier 1 to Tier 2 communication) ---
# This part would typically be defined in a "tier1" or "loadbalancer" module,
# but included here for context of how it connects to Tier 2.

resource "aws_lb" "application" {
  name               = "${var.environment}-tier2-alb"
  internal           = true # Internal ALB, accessed from Tier 1 (e.g., public ALB, web servers)
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id] # SG for the ALB
  subnets            = aws_subnet.public[*].id # ALBs typically in public subnets (even internal ones)

  tags = {
    Name = "${var.environment}-tier2-alb"
  }
}

resource "aws_lb_target_group" "tier2_app" {
  name     = "${var.environment}-tier2-app-tg"
  port     = 8080 # Application port
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  target_type = "instance" # Or "ip" if using Fargate/EKS

  health_check {
    path = "/health" # Your application's health check endpoint
    port = "traffic-port"
    protocol = "HTTP"
    matcher = "200"
  }

  tags = {
    Name = "${var.environment}-tier2-app-tg"
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.application.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tier2_app.arn
  }
}

resource "aws_lb_target_group_attachment" "tier2_app_attach" {
  count            = var.tier2_instance_count
  target_group_arn = aws_lb_target_group.tier2_app.arn
  target_id        = aws_instance.tier2_app[count.index].id
  port             = 8080
}

# --- Placeholder Security Group for ALB (would be in a separate module/file) ---
resource "aws_security_group" "alb_sg" {
  vpc_id = aws_vpc.main.id
  name   = "${var.environment}-alb-sg"
  description = "Security group for the Application Load Balancer"

  # Example: Allow HTTP from web tier (public ALB) or specific IPs
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Be more restrictive in production
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# --- Placeholder Security Group for Bastion Host (would be in a separate module/file) ---
resource "aws_security_group" "bastion_sg" {
  vpc_id = aws_vpc.main.id
  name   = "${var.environment}-bastion-sg"
  description = "Security group for the Bastion Host"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["YOUR_OFFICE_IP_CIDR"] # Restrict to known IPs
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}


# --- Data Source for latest Ubuntu AMI ---
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# --- Outputs (optional, but good practice) ---
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs for Tier 2 instances"
  value       = aws_subnet.private[*].id
}

output "tier2_security_group_id" {
  description = "The ID of the security group for Tier 2 instances"
  value       = aws_security_group.tier2_sg.id
}
```



##`Variables.tf`

```
variable "aws_region" {
  description = "The AWS region to deploy resources into."
  type        = string
  default     = "us-east-1" # Or your desired region
}

variable "environment" {
  description = "The environment name (e.g., dev, stage, prod)."
  type        = string
  default     = "dev"
}

variable "vpc_cidr_block" {
  description = "The CIDR block for the VPC."
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "List of Availability Zones to use for subnets."
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"] # Adjust for your region
}

variable "tier2_instance_type" {
  description = "The instance type for Tier 2 application servers."
  type        = string
  default     = "t3.medium"
}

variable "tier2_instance_count" {
  description = "Number of Tier 2 application instances."
  type        = number
  default     = 2
}

variable "ssh_key_pair" {
  description = "The name of the SSH key pair to use for EC2 instances."
  type        = string
  default     = "my-ec2-key" # Replace with your key pair name
}
```

`install_app.sh`

```
#!/bin/bash
sudo apt update -y
sudo apt install -y nginx # Example: Install Nginx as your app
sudo systemctl start nginx
sudo systemctl enable nginx
echo "<h1>Hello from Tier 2 Application Server!</h1>" | sudo tee /var/www/html/index.nginx-debian.html
```

### Deployment Steps:
Initialize Terraform:
```Bash
terraform init
```

### Review the Plan:
```Bash

terraform plan
```

### Apply the Configuration:
```Bash

terraform apply
```

