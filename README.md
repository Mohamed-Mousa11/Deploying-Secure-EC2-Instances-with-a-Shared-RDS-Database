# 🚀 Deploying Secure EC2 Instances with a Shared RDS Database

In this guide, we will walk through the process of deploying secure EC2 instances connected to a shared database on AWS using Terraform. We will cover the setup of a Virtual Private Cloud (VPC), Elastic Compute Cloud (EC2) instances and a Relational Database Service (RDS) instance. Our focus will be on adhering to best practices for security, scalability, and maintainability.

![Diagram](https://res.cloudinary.com/dezmljkdo/image/upload/v1721286786/LBD/ilgmizw70wuipqb6r7ku.png)

## 💡 Introduction to Terraform on AWS

Terraform is an open-source infrastructure as code software tool that allows you to define and provision a cloud infrastructure using a high-level configuration language. It supports various cloud providers, including AWS, which we will be using for our project.

### Why Terraform?

- **Immutable Infrastructure:** Terraform encourages the creation of immutable infrastructure through declarative configuration files. This means your infrastructure can be versioned and treated as you would with application code.
- **Idempotency:** Terraform ensures that running the same configuration multiple times results in the same state, avoiding manual errors and inconsistencies.
- **Scalability:** With Terraform, scaling your infrastructure up or down becomes a matter of changing a few lines in your configuration file.

## 🧮 Setting Up Your AWS Environment with Terraform

Before diving into the specifics, ensure you have Terraform and AWS CLI installed and configured on your machine.

### Creating a Terraform Configuration File

Create a new directory for your project and within it, create a file named `main.tf`. This file will contain the configuration for your AWS resources.

```hcl
provider "aws" {
  region = "us-east-1"
}
```

This specifies that Terraform should use the AWS provider and sets the region where your resources will be created.

## 🧱 Building the Infrastructure

Our web application will need a VPC, EC2 instances, and an RDS instance. We will define each of these components in our `main.tf` file.

### Virtual Private Cloud (VPC)

A VPC is a virtual network dedicated to your AWS account. It is isolated from other virtual networks in the AWS cloud.

```hcl
resource "aws_vpc" "app_vpc" {
  cidr_block = "10.0.0.0/16"
  enable_dns_support = true
  enable_dns_hostnames = true
  tags = {
    Name = "AppVPC"
  }
}
```

### Subnets

Within the VPC, we create subnets. Each subnet resides in a different availability zone for high availability.

```hcl
resource "aws_subnet" "app_subnet_1" {
  vpc_id            = aws_vpc.app_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  tags = {
    Name = "AppSubnet1"
  }
}

resource "aws_subnet" "app_subnet_2" {
  vpc_id            = aws_vpc.app_vpc.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"
  tags = {
    Name = "AppSubnet2"
  }
}
```

### Security Groups

Security groups act as a virtual firewall for your instances to control inbound and outbound traffic.

```hcl
resource "aws_security_group" "app_sg" {
  name        = "app_security_group"
  description = "Allow web traffic"
  vpc_id      = aws_vpc.app_vpc.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "AppSecurityGroup"
  }
}
```

### Elastic Compute Cloud (EC2)

EC2 instances will host our web application. We will create an instance within our VPC and associate it with the security group we defined.

```hcl
resource "aws_instance" "app_instance" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.app_subnet_1.id
  security_groups = [aws_security_group.app_sg.name]

  tags = {
    Name = "AppInstance"
  }
}
```

### Relational Database Service (RDS)

For data persistence, we will set up an RDS instance. It's managed by AWS, which simplifies database administration tasks such as backups and patching.

```hcl
resource "aws_db_instance" "app_db" {
  allocated_storage    = 20
  storage_type         = "gp2"
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.t2.micro"
  name                 = "appdb"
  username             = "admin"
  password             = "yourpassword"
  parameter_group_name = "default.mysql8.0"
  db_subnet_group_name = aws_db_subnet_group.app_db_subnet_group.name
  vpc_security_group_ids = [aws_security_group.app_sg.id]

  tags = {
    Name = "AppDBInstance"
  }
}
```

## 💥 Deploying Your Infrastructure

With all components defined, you can now deploy your infrastructure:

```bash
terraform init
terraform plan
terraform apply
```

1. `terraform init` initializes the Terraform configuration, preparing your working directory for other commands.
2. `terraform plan` creates an execution plan, allowing you to review the changes Terraform will make to your infrastructure.
3. `terraform apply` applies the changes to create the defined resources.

## 🎉 Conclusion

By following this guide, you have learned how to use Terraform to deploy a scalable web application on AWS. This setup includes a VPC, EC2 instances for the web application, an RDS instance for data persistence, routes, gateways and much more.  
Remember, this is just the beginning. Terraform's power and flexibility allow you to manage a wide range of AWS services and resources efficiently.

