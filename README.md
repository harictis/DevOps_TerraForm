# DevOps_Terraform
Deploying wordpress on AWS/Azure with RDS using Terraform, and Docker.

In this project, we will deploy a WordPress website on a virtual machine (VM) hosted in a public subnet. We will use Docker to containerize Apache webserver, PHP, and WordPress. Additionally, we will deploy a Relational Database Service (RDS) instance in a private subnet, connect the WordPress container to the RDS database, and then push the Docker image to Docker Hub.

## Prerequisites
  1. AWS/Azure Account
  2. Terraform installed locally
  3. Docker installed EC2
  4. Docker Hub Account
   
## AWS CLI installation
   In this we have used Ubuntu 20.04 so we need to install aws cli package to run the AWS cloud commands locally. After the installation we can configure the access tokens by using the following command.

       $ aws configure

  After that you can update the credentials by using

       $export AWS_ACCESS_KEY_ID=your_access_key_id
       $export AWS_SECRET_ACCESS_KEY=your_secret_access_key

 ## Terraform Installation
  To install terrraform locallly use the following command
  
        $sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
  After that
  
        $wget -O- https://apt.releases.hashicorp.com/gpg | \
        gpg --dearmor | \
        $sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
Verify the key's fingerprint.

         $gpg --no-default-keyring \
        --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
        --fingerprint
        
Download the package information from HashiCorp.

         $sudo apt update

Install Terraform from the new repository.

        $sudo apt-get install terraform
        
## Creating the Terraform Script
   The terraform script is used to provision the required resources and to maintain and monitor them. In this project we have used this for
       Creating a VM in a Public Subnet,
       Configuring Networking,
      Managing AWS Resources,
    Connecting to RDS.
## Creating a VM in a Public Subnet
   save this script in #provider.tf
  ![Screenshot 2023-09-15 112749](https://github.com/harictis/DevOps_TerraForm/assets/95492691/9a938754-e6f8-47ec-a760-89bd4fc650b3)

## For VPC
    $#vpc
    resource "aws_vpc" "this" {
      cidr_block = "10.100.0.0/16"
      tags = {
        Name = "upgrad-vpc"
      }
    }
    
    #public subnets
    resource "aws_subnet" "public1" {
      vpc_id     = aws_vpc.this.id
      cidr_block = "10.100.1.0/24"
      availability_zone = "us-east-1a"
    
      tags = {
        Name = "upgrad-public-1"
      }
    }
    resource "aws_subnet" "public2" {
      vpc_id     = aws_vpc.this.id
      cidr_block = "10.100.2.0/24"
      availability_zone = "us-east-1b"
    
      tags = {
        Name = "upgrad-public-2"
      }
    }
    
    #private subnets
    resource "aws_subnet" "private1" {
      vpc_id     = aws_vpc.this.id
      cidr_block = "10.100.3.0/24"
      availability_zone = "us-east-1a"
    
      tags = {
        Name = "upgrad-private-1"
      }
    }
    resource "aws_subnet" "private2" {
      vpc_id     = aws_vpc.this.id
      cidr_block = "10.100.4.0/24"
      availability_zone = "us-east-1b"
    
      tags = {
        Name = "upgrad-private-2"
      }
    }
![Screenshot 2023-09-15 113641](https://github.com/harictis/DevOps_TerraForm/assets/95492691/e293d58c-ef4f-406e-91f4-a34302720686)


## Internet Gateway

For creating IGW (Internet Gate Way), EIP (Elastic IP), NAT Gateway (Network Address Translation gateway) and Route tables for both public and private:

         #internet gateway
    resource "aws_internet_gateway" "igw" {
      vpc_id = aws_vpc.this.id
    
      tags = {
        Name = "upgrad-igw"
      }
    }
    
    #elastic ip
    resource "aws_eip" "eip" {
      domain = "vpc"
    }
    
    #nat gateway
    resource "aws_nat_gateway" "nat" {
      allocation_id = aws_eip.eip.id
      subnet_id     = aws_subnet.public1.id
    
      tags = {
        Name = "upgrad-nat"
      }
    }
    
    #public route table
    resource "aws_route_table" "public" {
      vpc_id = aws_vpc.this.id
    
      route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.igw.id
      }
    
      tags = {
        Name = "public-rt"
      }
    }
    
    #private route table
    resource "aws_route_table" "private" {
      vpc_id = aws_vpc.this.id
    
      route {
        cidr_block     = "0.0.0.0/0"
        nat_gateway_id = aws_nat_gateway.nat.id
      }
    
      tags = {
        Name = "private-rt"
      }
    }
## For creating Route table association and Security group
     #route table association
    resource "aws_route_table_association" "public1" {
      subnet_id      = aws_subnet.public1.id
      route_table_id = aws_route_table.public.id
    }
    resource "aws_route_table_association" "public2" {
      subnet_id      = aws_subnet.public2.id
      route_table_id = aws_route_table.public.id
    }
    resource "aws_route_table_association" "private1" {
      subnet_id      = aws_subnet.private1.id
      route_table_id = aws_route_table.private.id
    }
    resource "aws_route_table_association" "private2" {
      subnet_id      = aws_subnet.private2.id
      route_table_id = aws_route_table.private.id
    }
    
    #security group
    resource "aws_security_group" "allow_ssh" {
      name        = "allow_ssh"
      description = "Allow SSH inbound traffic"
      vpc_id      = aws_vpc.this.id
    
      ingress {
        description = "Allow SSH"
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }
    
      ingress {
        description = "Allow HTTP"
        protocol    = "tcp"
        from_port   = 80
        to_port     = 80
        cidr_blocks = ["0.0.0.0/0"]
      }
    
      egress {
        from_port        = 0
        to_port          = 0
        protocol         = "-1"
        cidr_blocks      = ["0.0.0.0/0"]
        ipv6_cidr_blocks = ["::/0"]
      }
    
      tags = {
        Name = "allow_ssh"
      }
    }
 # EC2 & RDS

      #EC2
      resource "aws_instance" "wordpress" {
        ami                         = "ami-053b0d53c279acc90"
        instance_type               = "t2.micro"
        key_name                    = "pem"
        subnet_id                   = aws_subnet.public1.id
        security_groups             = [aws_security_group.allow_ssh.id]
        associate_public_ip_address = true
      
      }
      
      #rds subnet
      resource "aws_db_subnet_group" "rds_subnet_group" {
        name       = "rds-subnet-group"
        subnet_ids = [aws_subnet.private1.id, aws_subnet.private2.id]
      }
      #RDS INSTANCE
      resource "aws_db_instance" "rds_instance" {
        engine                    = "mysql"
        engine_version            = "5.7"
        skip_final_snapshot       = true
        final_snapshot_identifier = "my-final-snapshot"
        instance_class            = "db.t2.micro"
        allocated_storage         = 20
        identifier                = "my-rds-instance"
        db_name                   = "wordpress_db"
        username                  = "mani"
        password                  = "mani123$"
        db_subnet_group_name      = aws_db_subnet_group.rds_subnet_group.name
        vpc_security_group_ids    = [aws_security_group.rds_security_group.id]
      
        tags = {
          Name = "RDS Instance"
        }
      }
      # RDS security group
      resource "aws_security_group" "rds_security_group" {
        name        = "rds-security-group"
        description = "Security group for RDS instance"
        vpc_id      = aws_vpc.this.id
      
        ingress {
          from_port   = 3306
          to_port     = 3306
          protocol    = "tcp"
          cidr_blocks = ["10.100.0.0/16"]
        }
      
        tags = {
          Name = "RDS Security Group"
        }
      }
# Terraform Plan 
Compile the script using 

      $terraform Plan
![Screenshot 2023-09-15 123023](https://github.com/harictis/DevOps_TerraForm/assets/95492691/9333d98c-121f-4557-ab48-e7721b27759f)


![Screenshot 2023-09-15 123043](https://github.com/harictis/DevOps_TerraForm/assets/95492691/3bafae6d-6251-4e7e-b63f-3596200a7be3)

## Terraform Apply
  Run the script template to deploy resources

     $terraform apply
![Screenshot 2023-09-15 123139](https://github.com/harictis/DevOps_TerraForm/assets/95492691/7b179478-b162-44eb-8281-4a36b3a307be)

![Screenshot 2023-09-15 123222](https://github.com/harictis/DevOps_TerraForm/assets/95492691/9a66d7f3-b473-4287-84b4-cf55dc359ed0)

![Screenshot 2023-09-15 123915](https://github.com/harictis/DevOps_TerraForm/assets/95492691/4570aa25-2ba5-4c19-a9d5-020e96f27bf0)
![Screenshot 2023-09-15 124048](https://github.com/harictis/DevOps_TerraForm/assets/95492691/a6700576-eb93-46bc-ab78-6f543255263a)
![Screenshot 2023-09-15 124124](https://github.com/harictis/DevOps_TerraForm/assets/95492691/c802b884-11ac-48b9-937c-57442428f07e)


# Check the Resources
![Screenshot 2023-09-15 131121](https://github.com/harictis/DevOps_TerraForm/assets/95492691/23cffc24-4bd5-4636-9bbb-d9657dfb7972)
![Screenshot 2023-09-15 131211](https://github.com/harictis/DevOps_TerraForm/assets/95492691/e3643716-6277-4199-b9a7-f759ea465a5f)
![Screenshot 2023-09-15 131445](https://github.com/harictis/DevOps_TerraForm/assets/95492691/1999015c-229d-4bb1-afc7-2b03de5d562f)


# Docker in EC2
 Initializing the docker engine in EC2 instance
![Screenshot 2023-09-15 160042](https://github.com/harictis/DevOps_TerraForm/assets/95492691/d74f11be-1f08-4c2a-bef1-95aff6eb310f)

 ![Screenshot 2023-09-15 160504](https://github.com/harictis/DevOps_TerraForm/assets/95492691/0bde52b9-f4aa-4522-9ad5-c4024a0806ab)



       $DockerFile
       
        FROM wordpress:latest

        WORKDIR /var/www/html
        
        RUN rm -rf *
        
        COPY . /var/www/html/    
PHP DockerFile

       $DockerFile

        FROM php:apache

        RUN a2enmod rewrite
        
        RUN docker-php-ext-install mysqli pdo pdo_mysql
        
        WORKDIR /var/www/html
![Screenshot 2023-09-15 160645](https://github.com/harictis/DevOps_TerraForm/assets/95492691/e9840369-1a52-476b-a4d5-0f6226b19885)
![Screenshot 2023-09-15 171128](https://github.com/harictis/DevOps_TerraForm/assets/95492691/b19f25ec-e12d-4033-b6c5-bf307f4eb9c1)
![Screenshot 2023-09-15 171227](https://github.com/harictis/DevOps_TerraForm/assets/95492691/f6196d7b-dc1c-4482-84a7-010961436f33)

# Check the Docker Hub

![Screenshot 2023-09-15 171414](https://github.com/harictis/DevOps_TerraForm/assets/95492691/8dd5b878-3704-4638-b829-3105943e3eda)


# Configure the Wordpress
Install the wordpress and configure it
![WhatsApp Image 2023-09-15 at 3 58 50 PM(3)](https://github.com/harictis/DevOps_TerraForm/assets/95492691/445492e5-ac5a-4e82-bc2b-b6fe50542af2)
![WhatsApp Image 2023-09-15 at 3 58 50 PM(2)](https://github.com/harictis/DevOps_TerraForm/assets/95492691/c3b59265-c4b4-4615-8e1d-5682e483d049)
![WhatsApp Image 2023-09-15 at 3 58 50 PM(1)](https://github.com/harictis/DevOps_TerraForm/assets/95492691/d13122a7-4172-4054-bd45-941b6b02791c)
![WhatsApp Image 2023-09-15 at 3 58 50 PM](https://github.com/harictis/DevOps_TerraForm/assets/95492691/ebf31f40-556f-419f-a60d-5bed409c3114)
![WhatsApp Image 2023-09-15 at 3 58 49 PM(1)](https://github.com/harictis/DevOps_TerraForm/assets/95492691/b6f93a9f-4d09-4370-a59f-67a3d7006db2)


# HOOREY!!
![WhatsApp Image 2023-09-15 at 3 58 49 PM](https://github.com/harictis/DevOps_TerraForm/assets/95492691/ddf4e45c-4573-45a8-9476-3ae9dbec6fe7)

# THANK YOU
