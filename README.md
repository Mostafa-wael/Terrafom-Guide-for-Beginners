# Terrafom-Guide

**General syntax**:

    provider "<provider_name>"
    {
        config options...
        access_key = "<your_access_key>"
        secret_key = "<your_secret_key>"
    }

    resource "<provider_name>_<resource_type>" "<name_for_terraform_file>"
    {
        config options...
        key_n = "value_n"
    }
---
**Main Commands**:

`terraform init`: initialize the project and download the required plugins.

`terraform plan`: show changes, don't apply them (`+` add new), (`~` update), (`-` remove).

`terraform apply`: apply the changes, hit `yes` after running or append `--auto-approve` to the command.

`terraform destroy`: destroy all our resources, hit `yes` after running or append `--auto-approve` to the command.

`terraform destroy -target <resource_name>`: destroy this target resource, same for apply.

`terraform state list`: list all resources in the state. 

`terraform state show <resource_name >`: list details about that resource instead of the provider console. 

`terraform output`: show all the outputs.

`terraform refresh`: refresh all stats without updating anything, will also show the outputs.

**Using Variables**:

To define a variable:
  
    variable "<variable_name>"{
    description = <optional_description>
    default = <optional_default_value>
    type = <optional_type> #e.g.: string, any, ...
    }

To reference a variable:
    `var.<variable_name>`

To initialize a variable:

    1. Pass it with the `apply` command like this: `terraform apply -var <variable_name>=<variable_value>`.
    
    2. Add values to the `terraform.tfvars` file like this: `<variable_name>=<variable_value>`. 
    You can optionally pass another file name like this:`terraform apply -var-file <file_name>.tfvars`. 
    
    3. In case of no default value assigned:
       - Terraform will ask the you to enter a value the `apply`/`destroy` command. 
       - You can enter dummy data with the `destroy` command.

**Outputs**:

To print outputs with the `apply` command, add the following snippet to the terraform script for each value:

    output "<value_name>"{
        value = <resource_type.resource_name.attribute>
    }

It can also be shown using `terraform output`

**Some Notes**:
- Terraform is written in a declarative manner, no matter how much time we apply the script, the final blueprint will be always the same. Running again will only check again if the final state was applied correctly.
- `terrafom.tfstate` file holds the current state of our terraform script to keep track of what was created by terraform and only modify it. In other words, it is the most critical and important file in our project.

**Illustrative Example**:
Create and deploy a server using terraform:

    1. Create Key-Pair:
        Go to EC2-> Key Pairs, it will allow us to connect to our server once created.
    
    2. Configure the provider, aws in this example:
        provider "aws"
        {
            access_key = "<your_access_key>"
            secret_key = "<your_secret_key>"
        }
    3. Create VPC.
        resource "aws_vpc" "prod-vpc" {
            cidr_block = "10.0.0.0/16"
            tags = {
                Name = "production"
            }
        }
    
    4. Create internet Gateway.
        resource "aws_internet_gateway" "gw"
        {
            vpc_id = aws_vpc_prod-vpc.id
            tags = { Name = "production"}
        } 
    
    5. Create Custom Route Table.
        resource "aws_route_table" "prod-route-table" {
            vpc_id = aws_vpc.example.id

            route {
                cidr_block = "0.0.0.0/0"
                gateway_id = aws_internet_gateway.gw.id
            }

            route {
                ipv6_cidr_block        = "::/0"
                gateway_id = aws_internet_gateway.gw.id
            }

            tags = {
                Name = "Prod"
            }
        }
    
    6. Create Subnet.
        resource "aws_subnet" "subnet-1" {
            vpc_id     = aws_vpc.prod-vpc.id
            cidr_block = "10.0.1.0/24"
            availability_zone = "us-east-1a"

            tags = {
            Name = "prod_subnet"
            }
        }
    
    7. Associate subnet with Route Table
        resource "aws_route_table_association" "a" {
            subnet_id = aws_subnet.subnet-1.id
            route_table_id  = aws_route_table.prod-route-table.id
        }
    
    8. Create Security Group to allow ports 22, 80, 443.
        resource "aws_security_group" "allow_web" {
        name        = "allow_web_traffic"
        description = "Allow web inbound traffic"
        vpc_id      =  aws_vpc.prod-vpc.id

        ingress {
            description      = "HTTPS"
            from_port        = 443
            to_port          = 443
            protocol         = "tcp"
            cidr_blocks      = ["0.0.0.0/0"]
            ipv6_cidr_blocks = ["::/0"]
        }
        ingress {
            description      = "HTTP"
            from_port        = 80
            to_port          = 80
            protocol         = "tcp"
            cidr_blocks      = ["0.0.0.0/0"]
            ipv6_cidr_blocks = ["::/0"]
        }
        ingress {
            description      = "SSH"
            from_port        = 22
            to_port          = 22
            protocol         = "tcp"
            cidr_blocks      = ["0.0.0.0/0"]
            ipv6_cidr_blocks = ["::/0"]
        }

        egress {
            from_port        = 0
            to_port          = 0
            protocol         = "-1"
            cidr_blocks      = ["0.0.0.0/0"]
            ipv6_cidr_blocks = ["::/0"]
        }

        tags = {
            Name = "allow_web"
        }
        }
    
    9.  Create a network interface with an ip in the subnet that was created in step 5.
        resource "aws_network_interface" "web-server-nic" {
            subnet_id       = aws_subnet.subnet-1.id
            private_ips     = ["10.0.1.50"]
            security_groups = [aws_security_group.allow_web.id]
        }
    
    10. Assign an elastic IP to the network interface created in step 8.
        resource "aws_eip" "one"{
            vpc = true
            network_interface = aws_network_interface.web-server-nic.id
            associate_with_private_ip = "10.0.1 .50"
            depends_on = [aws_internet_gateway.gw]
        }
    
    11. Create Ubuntu server and install/enable apache2.
        resource "aws_instance" "web-server-instance"{
            ami = "ami-085325f297f89fce1"
            instance_type = "t2.micro"
            availability_zone = "us-east-1a"
            key_name = "main_key" # that we created in step 1
            
            network_interface = {
                device_index = 0
                network_interface_id = aws_network_interface.web-server-nic.id
            }
            tags = { Name = "web-server"}
            
            user_data = <<-EOF
                        #!/bin/bash
                        sudo apt-get update - y
                        sudo apt-get install apache2 -y
                        sudo systemctl start apache2
                        sudo bash -c "echo Your web server" > /var/ww/html/index.html"
                        EOF
        }
    
    12. Go ot the "IPv1 Public IP" address shown in EC2 Console. You will see the "Your web server" message.

Adopted from: [The FreeCodeCampVideo](https://www.youtube.com/watch?v=SLB_c_ayRMo&ab_channel=freeCodeCamp.org)


