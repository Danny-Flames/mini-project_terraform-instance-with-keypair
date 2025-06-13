# Terraform EC2 Instance with Key Pair - Step-by-Step Guide

## Prerequisites Setup

### 1. Install Required Tools
- **AWS CLI**: Download and install from AWS official site
- **Terraform**: Download from HashiCorp's official site
- **Text Editor**: nano, vim, or VS Code

### 2. Configure AWS Credentials
```bash
aws configure
```
Enter my:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g., us-east-1)
- Default output format (json)

### 3. Verify AWS Configuration
```bash
aws sts get-caller-identity
```

## Phase 1: Project Setup

### Step 1: Create Project Directory
```bash
mkdir terraform-ec2-keypair
cd terraform-ec2-keypair
```

### Step 2: Generate SSH Key Pair (if I don't have one)
```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/terraform-key
```
- This creates `terraform-key` (private) and `terraform-key.pub` (public)

## Phase 2: Terraform Configuration

### Step 3: Create Enhanced main.tf
The provided template has issues. Here's a corrected version:

```hcl
# Provider configuration
provider "aws" {
  region = "us-east-1"  # Change to my preferred region
}

# Data source to get the latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Get default VPC
data "aws_vpc" "default" {
  default = true
}

# Create security group
resource "aws_security_group" "web_sg" {
  name_prefix = "terraform-web-sg"
  vpc_id      = data.aws_vpc.default.id

  # HTTP access
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # SSH access
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Restrict this to my IP for security
  }

  # All outbound traffic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "terraform-web-security-group"
  }
}

# Generate private key
resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

# Create AWS key pair
resource "aws_key_pair" "ec2_keypair" {
  key_name   = "terraform-ec2-key"
  public_key = tls_private_key.ec2_key.public_key_openssh

  tags = {
    Name = "terraform-ec2-keypair"
  }
}

# Save private key to local file
resource "local_file" "private_key" {
  content         = tls_private_key.ec2_key.private_key_pem
  filename        = "terraform-ec2-key.pem"
  file_permission = "0400"
}

# Create EC2 instance
resource "aws_instance" "web_server" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t2.micro"
  key_name              = aws_key_pair.ec2_keypair.key_name
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Hello World from $(hostname -f)</h1>" > /var/www/html/index.html
              echo "<p>Instance launched with Terraform!</p>" >> /var/www/html/index.html
              echo "<p>Current date: $(date)</p>" >> /var/www/html/index.html
              EOF

  tags = {
    Name = "terraform-web-server"
  }
}

# Outputs
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web_server.id
}

output "public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.web_server.public_ip
}

output "public_dns" {
  description = "Public DNS name of the EC2 instance"
  value       = aws_instance.web_server.public_dns
}

output "private_key_file" {
  description = "Path to the private key file"
  value       = "terraform-ec2-key.pem"
}

output "ssh_command" {
  description = "SSH command to connect to the instance"
  value       = "ssh -i terraform-ec2-key.pem ec2-user@${aws_instance.web_server.public_ip}"
}
```

### Step 4: Initialize Terraform
```bash
terraform init
```

### Step 5: Validate Configuration
```bash
terraform validate
```

### Step 6: Plan the Deployment
```bash
terraform plan
```

## Phase 3: Deployment

### Step 7: Apply Configuration
```bash
terraform apply
```
- Review the plan
- Type `yes` to confirm

### Step 8: Verify Outputs
After successful deployment, note the outputs:
- Instance ID
- Public IP address
- SSH command

## Phase 4: Testing and Verification

### Step 9: Test Web Server
```bash
# Using curl
curl http://YOUR_PUBLIC_IP

# Or open in browser
http://YOUR_PUBLIC_IP
```

### Step 10: SSH Access (Optional)
```bash
ssh -i terraform-ec2-key.pem ec2-user@YOUR_PUBLIC_IP
```

## Phase 5: Documentation and Cleanup

### Step 11: Document My Findings
Create a report including:
- Screenshots of successful deployment
- Output of `terraform show`
- Web server response
- Any challenges faced

### Step 12: Cleanup Resources
```bash
terraform destroy
```
- Type `yes` to confirm deletion

## Troubleshooting Common Issues

### Issue 1: AMI Not Found
- Update the AMI ID for my region
- Use the data source approach shown above

### Issue 2: Security Group Not Found
- Remove hardcoded security group IDs
- Create security group with Terraform

### Issue 3: Key Pair Issues
- Ensure SSH key exists at specified path
- Use the TLS provider approach for auto-generation

### Issue 4: Permission Denied
- Check AWS credentials
- Verify IAM permissions for EC2, VPC, and Key Pair operations

## Best Practices Applied

1. **Dynamic AMI Selection**: Uses data source instead of hardcoded AMI
2. **Security Group Creation**: Creates dedicated security group
3. **Key Pair Generation**: Automatically generates and saves key pair
4. **Proper Tagging**: Tags all resources for identification
5. **Comprehensive Outputs**: Provides all necessary connection information

## Files in My Project Directory
After completion, I should have:
- `main.tf` - Terraform configuration
- `terraform-ec2-key.pem` - Private key file
- `.terraform/` - Terraform working directory
- `terraform.tfstate` - State file

## Security Considerations
- The SSH access (port 22) is open to all IPs (`0.0.0.0/0`) for demonstration
- In production, restrict SSH access to my specific IP address
- Store private keys securely and never commit them to version control