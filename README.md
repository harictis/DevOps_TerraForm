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
   ![Screenshot 2023-09-15 112749](https://github.com/harictis/DevOps_Terraform/assets/95492691/9d0b633f-b22e-4500-9d85-b1f8c541205d)

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

![Screenshot 2023-09-15 113641](https://github.com/harictis/DevOps_Terraform/assets/95492691/669a1a7a-3fb1-4f63-971d-2fe8a395783a)

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
# Terraform Plan 
Compile the script using 

      $terraform Plan
![Screenshot 2023-09-06 184923](https://github.com/harictis/DevOps_TerraForm/assets/95492691/ba70577e-2892-4436-8361-bd8df8db63d3)


![Screenshot 2023-09-15 123043](https://github.com/harictis/DevOps_Terraform/assets/95492691/fa64463e-2f37-4008-9540-cba6b9c1ec4d)

## Terraform Apply
  Run the script template to deploy resources

     $terraform apply


![Screenshot 2023-09-15 123139](https://github.com/harictis/DevOps_Terraform/assets/95492691/7132ba23-af14-4531-9025-5ca8c2e55222)
![Screenshot 2023-09-15 123222](https://github.com/harictis/DevOps_Terraform/assets/95492691/e1e3726d-99cc-4676-938d-542b2a3b704a)
![Screenshot 2023-09-15 124048](https://github.com/harictis/DevOps_Terraform/assets/95492691/44fb94d9-3e89-43d8-8950-7e9cf2819469)
![Screenshot 2023-09-15 124124](https://github.com/harictis/DevOps_Terraform/assets/95492691/bfc42c9c-7103-48b6-ab2f-613bc63dd37b)

# Check the Resources
![Screenshot 2023-09-15 131121](https://github.com/harictis/DevOps_Terraform/assets/95492691/d0cd0156-fc21-4080-bb6f-28f598da0b9d)

![Screenshot 2023-09-15 131211](https://github.com/harictis/DevOps_Terraform/assets/95492691/830ba4a6-23ba-44b5-b099-c3a1011be4a5)
![Screenshot 2023-09-15 131445](https://github.com/harictis/DevOps_Terraform/assets/95492691/b152d56b-651f-47a7-be2f-6bfa3ebe794a)

# Docker in EC2
 Initializing the docker engine in EC2 instance

 ![Screenshot 2023-09-15 160042](https://github.com/harictis/DevOps_Terraform/assets/95492691/8bab85cf-00c2-441c-903c-5a59f4d05f9a)

![Screenshot 2023-09-15 160504](https://github.com/harictis/DevOps_Terraform/assets/95492691/8a6034a2-10cd-41cc-a641-a80ea46d6190)

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
![Screenshot 2023-09-15 160645](https://github.com/harictis/DevOps_Terraform/assets/95492691/d778bb30-3e77-42b6-a759-7acb0e673623)
![Screenshot 2023-09-15 171128](https://github.com/harictis/DevOps_Terraform/assets/95492691/2ccffe35-a139-491b-ab4a-0297c2a31e13)
![Screenshot 2023-09-15 171227](https://github.com/harictis/DevOps_Terraform/assets/95492691/a1de18fa-6097-4857-a781-838c74741b87)
# Check the Docker Hub

![Screenshot 2023-09-15 171414](https://github.com/harictis/DevOps_Terraform/assets/95492691/f7fb6409-7bff-4840-9084-eaff4cb1b43d)

# Configure the Wordpress
Install the wordpress and configure it

![WhatsApp Image 2023-09-15 at 3 58 50 PM(3)](https://github.com/harictis/DevOps_Terraform/assets/95492691/47469e2f-7307-40a4-abef-c93fb1538695)
![WhatsApp Image 2023-09-15 at 3 58 50 PM(2)](https://github.com/harictis/DevOps_Terraform/assets/95492691/6a7bd9c1-6dca-4adb-9c06-469edbd7123f)
![WhatsApp Image 2023-09-15 at 3 58 50 PM(1)](https://github.com/harictis/DevOps_Terraform/assets/95492691/12bf39e4-5236-4375-89a6-e67b8c918e5b)
![WhatsApp Image 2023-09-15 at 3 58 50 PM](https://github.com/harictis/DevOps_Terraform/assets/95492691/8f467648-b407-44e1-b00c-9bdd14e4db6b)
![WhatsApp Image 2023-09-15 at 3 58 49 PM(1)](https://github.com/harictis/DevOps_Terraform/assets/95492691/8d3c1d1e-3cc8-4cba-b3f6-c1e4c1e376da)
# HOOREY!!
![WhatsApp Image 2023-09-15 at 3 58 49 PM](https://github.com/harictis/DevOps_Terraform/assets/95492691/95831325-15e5-499f-a091-fa1a70ce1474)

# THANK YOU
