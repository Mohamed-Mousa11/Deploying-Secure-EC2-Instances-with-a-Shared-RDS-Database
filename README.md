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
### Create two **network interfaces**

Create two **network interfaces** - `nw-interface1` and `nw-interface2`.

Both of the interfaces should use `WebTrafficSG` as the security group, while the `nw-interface1` should use `AppSubnet1` and `nw-interface2` use `AppSubnet2` respectively.

```hcl
resource "aws_network_interface" "nw-interface1" {
  subnet_id = aws_subnet.AppSubnet1.id
  security_groups = [aws_security_group.WebTrafficSG.id]
  tags = {
    Name        = "nw-interface1"
  }  
}

resource "aws_network_interface" "nw-interface2" {
  subnet_id = aws_subnet.AppSubnet2.id
  security_groups = [aws_security_group.WebTrafficSG.id]
  tags = {
    Name        = "nw-interface2"
  }  
}

### Create `Internet Gateway`

Attach the network (AppVPC) to any `Internet Gateway`. Tag this gateway as `AppInternetGateway`.

Also, create a route table for the VPC `AppVPC`. Tag this table as `AppRouteTable`. Create an associated output for this ID named `route_table_ID`.

```hcl
resource "aws_internet_gateway" "AppIGW" {
  vpc_id = aws_vpc.AppVPC.id

  tags = {
    Name = "AppInternetGateway"
  }
}

resource "aws_route_table" "AppRouteTable" {
  vpc_id = aws_vpc.AppVPC.id
  tags = {
    Name = "AppRouteTable"
  }
}

output "route_table_ID" {
  value = aws_route_table.AppRouteTable.id
}
```


### Create Route Table

Create a route in your AWS infrastructure to allow internet access. The route should be associated with the route table named `AppRouteTable` and should direct traffic to the internet gateway named `AppInternetGateway`.

Set the destination CIDR block to `0.0.0.0/0` to allow all outbound traffic.

```hcl
resource "aws_route" "internet_access" {
  route_table_id         = aws_route_table.AppRouteTable.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.AppIGW.id
}
```

### Create Route Table Association

Associate two subnets, `AppSubnet1` and `AppSubnet2`, with the route table named `AppRouteTable` to ensure that the subnets use this route table for their traffic routing.

```hcl
resource "aws_route_table_association" "AppSubnet1_association" {
  subnet_id      = aws_subnet.AppSubnet1.id
  route_table_id = aws_route_table.AppRouteTable.id
}

resource "aws_route_table_association" "AppSubnet2_association" {
  subnet_id      = aws_subnet.AppSubnet2.id
  route_table_id = aws_route_table.AppRouteTable.id
}
```


### Create `Elastic IP (EIP)`

To ensure that our future EC2 instances get assigned a public IP address, create two `Elastic IP (EIP)` resources and attach to one network interface each - `nw-interface1` and`nw-interface2`

```hcl
resource "aws_eip" "public_ip1" {
  vpc = true
  network_interface = aws_network_interface.nw-interface1.id
}

resource "aws_eip" "public_ip2" {
  vpc = true
  network_interface = aws_network_interface.nw-interface2.id
}
```


### Create Create two EC2 instances


Create two EC2 instances within **AppVPC**, one in each subnet (`AppSubnet1` and `AppSubnet2`), using the `ami-06c68f701d8090592` AMI and `t2.micro` instance type.

Create a key-pair for the EC2 instances called `my-ec2-key`. Store it in `/root`. Use this key-pair for both the EC2 instances.

Tag the instances with `Name` as **WebServer1** (AppSubnet1) and **WebServer2** (AppSubnet2) respectively.

Attach the appropriate network interfaces to each instance according to their subnet ID.

To ensure that our future EC2 instances get assigned a public IP address, create two `Elastic IP (EIP)` resources and attach to one network interface each - `nw-interface1` and`nw-interface2`

First, run the following command in the terminal to create a key-pair:

```
aws ec2 create-key-pair --key-name my-ec2-key --query 'KeyMaterial' --output text > /root/my-ec2-key.pem
```

Change the permissions of the key so that the root user has read and write access to it:

```
chmod 600 /root/my-ec2-key.pem
```

append the EC2 instances configuration:
```hcl
resource "aws_instance" "WebServer1" {
  ami             = "ami-06c68f701d8090592"
  instance_type   = "t2.micro"

  network_interface {
    network_interface_id = aws_network_interface.nw-interface1.id
    device_index = 0
  }

  key_name = "my-ec2-key"

  tags = {
    Name = "WebServer1"
  }
}

resource "aws_instance" "WebServer2" {
  ami             = "ami-06c68f701d8090592"
  instance_type   = "t2.micro"

  network_interface {
    network_interface_id = aws_network_interface.nw-interface2.id
    device_index = 0
  }

  key_name = "my-ec2-key"

  tags = {
    Name = "WebServer2"
  }
}
```

### Create Create two outputs


Add two outputs to the configuration that contain the instance IDs of the created EC2 instances.

Name the outputs as `instance1_id` and `instance2_id` respectively.
```hcl
output "instance1_id" {
  value = aws_instance.WebServer1.id
}

output "instance2_id" {
  value = aws_instance.WebServer2.id
}
```

### Create a **database subnet group**


We will now be provisioning an `RDS` database instance. We want this instance to be accessible from the security group of the web servers.

Create a **database subnet group** called `app-db-subnet-group` which should include the subnets within the VPC `AppVPC`.
```hcl
resource "aws_db_subnet_group" "app_db_subnet_group" {
  name       = "app-db-subnet-group"
  subnet_ids = [aws_subnet.AppSubnet1.id, aws_subnet.AppSubnet2.id]  

  tags = {
    Name = "AppDBSubnetGroup"
  }
}
```

### Create RDS database instance


Now, provision an RDS instance in `AppVPC`. The database should be accessible from the WebServer security group and have the following specs:

- **Allocated storage:** `20`
- **Engine:** `mysql`
- **Engine version:** `8.0.33`
- **Instance class:** `db.t3.micro`
- **Database name:** `appdatabase`
- **Username:** `admin`
- **Password:** `db*pass123`
- **Database subnet group:** `app_db_subnet_group`
- **VPC security group ID:** ID of `WebTrafficSG`

Ensure that the database is publicly accessible. Tag the RDS instance with `Name` as `AppDatabase`.
```hcl
resource "aws_db_instance" "app_database" {
  allocated_storage    = 20
  engine               = "mysql"
  engine_version       = "8.0.33"  
  instance_class       = "db.t3.micro" 
  identifier           = "appdatabase"
  db_name              = "appdatabase"
  username             = "admin"
  password             = "db*pass123"  
  publicly_accessible     = true
  db_subnet_group_name = aws_db_subnet_group.app_db_subnet_group.name
  vpc_security_group_ids = [aws_security_group.WebTrafficSG.id]

  tags = {
    Name = "AppDatabase"
  }
}
```
### Connecting to the database


Now that both the `EC2 instances` and the `RDS database` have been created, lets use one of our EC2 instance to connect to the database.

From the **AWS Management Console**, grab the public IPv4 address of one of the EC2 instances - `WebServer1` or `WebServer2`.

In the `root` directory of your terminal, run the following command:

```
ssh -i my-ec2-key.pem ec2-user@<public_IP>
```

Replace `<public_IP>` with the IP of your instance.

Enter `yes` when you encounter this prompt:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```


![[ssh to ec2 done.jpg]]


Since, our AMI instance doesn't have `MySQL` pre-installed, run the following commands sequentially to install it:

```
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm

sudo dnf install mysql80-community-release-el9-1.noarch.rpm -y

sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023

sudo dnf install mysql-community-client -y
```

Once `MySQL` is installed, run the following command to connect to the database:

```
mysql -h <DB_endpoint> -P 3306 -u admin -p
```

Replace the `<DB_endpoint>` with the endpoint of your database instance that was created.

When prompted for password, enter `db*pass123` - the password you created via terraform.
![[connected to db.jpg]]

### Testing my RDS

Having logged into your RDS instance from one of the `EC2` instances - you should now be able to execute all SQL queries.

Run the following query:

```
SHOW DATABASES;
```

You should see a list of databases, one of them being `appdatabase` - the one we created via terraform.

![[DB working.jpg]]

This demonstrates that our RDS database is working as a shared database instance - accessible from both the web servers.




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

