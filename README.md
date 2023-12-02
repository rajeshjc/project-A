# project-A
Overview
In today's technological landscape, infrastructure automation is fundamental. While much focus is placed on software development processes, a robust infrastructure deployment strategy is equally crucial. Automating infrastructure not only enhances disaster recovery but also streamlines testing and development processes.

Your organization is transitioning towards DevOps methodology, emphasizing automated provisioning of infrastructure. The pivotal requirement is to establish a centralized server for Jenkins, a crucial component in the DevOps toolchain. Utilizing Terraform, a powerful provisioning tool, this project aims to set up the required infrastructure and subsequently install essential automation tools within it.

Project Objectives
Provision a central server for Jenkins using Terraform.
Establish connectivity to the provisioned server.
Install Jenkins along with essential dependencies like Java and Python within the instance.
Tools Required
Terraform
AWS account with appropriate security credentials
Keypair for secure access
Deliverables
Launch an EC2 Instance Using Terraform
Connect to the Provisioned Instance
Install Jenkins, Java, and Python within the Instance
Project Source Code - main.tf - https://github.com/paglipay/paglipay-terraform-jenkins.git
# The provided Terraform code focuses on provisioning infrastructure components for AWS using Terraform.

# Note: The current code successfully creates VPC, subnets, security groups, and an EC2 instance, and proceeds to install Jenkins, Java, Python, and other necessary tools.

##################################################################################
# PROVIDERS
##################################################################################

provider "aws" {
  region = "us-east-1"
}

##################################################################################
# DATA
##################################################################################

data "aws_ssm_parameter" "amzn2_linux" {
  name = "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"

}


##################################################################################
# RESOURCES
##################################################################################

# NETWORKING #
resource "aws_vpc" "app" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

}

resource "aws_internet_gateway" "app" {
  vpc_id = aws_vpc.app.id

}

resource "aws_subnet" "public_subnet1" {
  cidr_block              = "10.0.0.0/24"
  vpc_id                  = aws_vpc.app.id
  map_public_ip_on_launch = true
}

# ROUTING #
resource "aws_route_table" "app" {
  vpc_id = aws_vpc.app.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.app.id
  }
}

resource "aws_route_table_association" "app_subnet1" {
  subnet_id      = aws_subnet.public_subnet1.id
  route_table_id = aws_route_table.app.id
}

# SECURITY GROUPS #


# Nginx security group 
resource "aws_security_group" "nginx_sg" {
  name   = "nginx_sg"
  vpc_id = aws_vpc.app.id

  # HTTP access from anywhere
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTP access from anywhere
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTP access from anywhere
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTP access from anywhere
  ingress {
    from_port   = 8000
    to_port     = 8000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTP access from anywhere
  ingress {
    from_port   = 9001
    to_port     = 9001
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTP access from anywhere
  ingress {
    from_port   = 9443
    to_port     = 9443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTP access from anywhere
  ingress {
    from_port   = 8081
    to_port     = 8081
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTP access from anywhere
  ingress {
    from_port   = 5001
    to_port     = 5001
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # outbound internet access
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# INSTANCES # https://dev.to/aws-builders/installing-jenkins-on-amazon-ec2-491e
resource "aws_instance" "nginx1" {
  ami                    = nonsensitive(data.aws_ssm_parameter.amzn2_linux.value)
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.public_subnet1.id
  vpc_security_group_ids = [aws_security_group.nginx_sg.id]

  user_data = <<EOF
#! /bin/bash

#sudo yum update -y

sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

sudo yum upgrade -y
sudo amazon-linux-extras install -y java-openjdk11

sudo yum install -y jenkins
sudo service jenkins start

sudo yum install -y git python3


sudo amazon-linux-extras install -y nginx1
sudo service nginx start
sudo rm /usr/share/nginx/html/index.html
echo '<html><head><title>Taco Team Server</title></head><body style=\"background-color:#1F778D\"><p style=\"text-align: center;\"><span style=\"color:#FFFFFF;\"><span style=\"font-size:28px;\">You did it! Have a &#127790;</span></span></p></body></html>' | sudo tee /usr/share/nginx/html/index.html

sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install terraform

sudo cat /var/lib/jenkins/secrets/initialAdminPassword

sudo yum install -y docker
sudo yum install -y python3-pip
sudo pip3 install -y docker-compose
sudo systemctl enable docker.service
sudo systemctl start docker.service

sudo usermod -a -G docker ec2-user

sudo docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
sudo docker network create --driver overlay   portainer_agent_network
sudo docker service create   --name portainer_agent   --network portainer_agent_network   -p 9001:9001/tcp   --mode global   --constraint 'node.platform.os == linux'   --mount type=bind,src=//var/run/docker.sock,dst=/var/run/docker.sock   --mount type=bind,src=//var/lib/docker/volumes,dst=/var/lib/docker/volumes   portainer/agent:2.19.2
   
sudo docker run -it -d -p 8081:8080 -v /var/run/docker.sock:/var/run/docker.sock dockersamples/visualizer

EOF

}
