# Automate Infrastructure With IAC using Terraform Part 2
<p>In the previous part, we were able to to provision a VPC with public subnet </p>

<p>In this part we are going to finalize all of the infrastructure based on the below architectural digram</p>

![tooling_project_15](https://user-images.githubusercontent.com/76074379/127383520-f081e1da-44d3-464d-bf4b-8b0e28273147.png)


---
## Create Private Subnets

* According to the architecture we need to create 4 private subnets. let's create the first two .  

In the `main.tf` file which we created before, please add the the snippet of code below

```
resource "aws_subnet" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4 , count.index + 1)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```
As we did before with public subnet we are doing the same with private subnet 
- Define a variable in `Variables.tf` file called  `preferred_number_of_private_subnets_A`
```
  variable "preferred_number_of_private_subnets_A" {
      default = null
}
```
- Update `terraform.tfvars` with the value below: 
```
preferred_number_of_private_subnets_A = 2
```

---

**NOTE:**

We are going to create lots of resources so we need to <b>tag </b> all resources we will create. Tags enable you to categorize your AWS resources in different ways, for example, by purpose, owner, or environment. This is useful when you have many resources of the same type
```
tags = merge(
    var.additional_tags,
    {
      Name = format("PrivateSubnetA-%s", count.index)
    } 
  )
}
```
The `format` function produces a string by formatting a number of other values according to a specification string. It is similar to the **printf** function in C, and other similar functions in other programming languages.
`format(spec, values...)`.

We create everything with `multi AZ` so we need to differentiate between them. This will help us to create good tagging sting by adding for example the count of the resource.

---
Update your private subnet resource to be as below

```
resource "aws_subnet" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4 , count.index + 1)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = merge(
    var.additional_tags,
    {
      Name = format("PrivateSubnetA-%s", count.index)
    } 
  )
}
```
All we did was just to add tag block to be able to create the tag we want.

---
Update ***variables.tf*** file to define a variable called **additional_tags** since it will be used more than once as we will be tagging our other resources we need to tag.

```
variable "additional_tags" {
  default     = {}
  description = "Additional resource tags"
  type        = map(string)
}
```

Our full `main.tf` file after adding tags will be as below
```
# Get list of availability zones
data "aws_availability_zones" "available" {
state = "available"
}

provider "aws" {
  region = var.region
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support 
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink
  tags = {
    Name = "PublicSubnet"
    Environment = var.environment
  }
}

# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index +3)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  
}
resource "aws_subnet" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4 , count.index + 1)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = merge(
    var.additional_tags,
    {
      Name = format("PrivateSubnetA-%s", count.index)
    } 
  )
}

```
Now lets run command 
```
terraform plan
```
If we are okay with the output, then run
```
terraform apply -auto-aprrove
```


Check the AWS Console, we will find that aws has been updated with new subnets


---

## Replicate what did we do for the other private subnet 
- Make sure to update `variables.tf` file and `terraform.tfvars` file and then add the code below to `main.tf`
```
resource "aws_subnet" "private_B" {
  count = var.preferred_number_of_private_subnets_B == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_B
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4 , count.index + 7)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = merge(
    var.additional_tags,
    {
      Name = format("PrivateSubnetB-%s", count.index)
    } 
  )
}
```
---
**NOTE:** The main aim of using terraform is to convert our infrastructure to code and this gives us the opportunity to organize our resources and one of the best practices that terraform gives. <b>It gives us the power to organize our resources in seprate files.</b>

---

## Create internet gateway
- create a file called `internet_gateway.tf` and add the code below
```
resource "aws_internet_gateway" "ig" {
  vpc_id = aws_vpc.main.id
  tags = merge(
    var.additional_tags,
    {
      Name = format("%s-%s!", aws_vpc.main.id,"IG")
    } 
  )
}
```
## Create an Elastic IP(EIP) and AWS NATgateway and allocate the EIP to the NATgateway
create a new file called `natgateway.tf` and the following code to it 
```
resource "aws_eip" "nat_eip" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc        = true
  depends_on = [aws_internet_gateway.ig]
  tags = {
    Name        =  format("EIP-%s",var.environment)
    Environment = var.environment
  }
}

resource "aws_nat_gateway" "nat" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  allocation_id = aws_eip.nat_eip[count.index].id
  subnet_id     = element(aws_subnet.public.*.id, 0)
  depends_on    = [aws_internet_gateway.ig]
  tags = {
    Name        =  format("Nat-%s",var.environment)
    Environment = var.environment
  }
}
```

**NOTE**: As we can see in the code snippet above, we included a new variable called *Environment* in the tag block. So we will have to define that variable in the ***variable.tf*** file. Add the code below
```
variable "environment" {
  default = "for-private-subnets"
}
```

## Create AWS Routes 
### Create aws routes for public subnet
Create a file called `route_tables.tf` add the following code to it 

```
resource "aws_route_table" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets
  vpc_id = aws_vpc.main.id
  tags = {
    Name        = format("Public-RT-%s", var.environment)
    Environment = var.environment
  }
}
resource "aws_route" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets
  route_table_id         = aws_route_table.public[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.ig.id
}

resource "aws_route_table_association" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public[count.index].id
}

```
### Create aws routes for first private subnet
Add to the same file `route_tables.tf` . Add the following code to it 
```
resource "aws_route_table" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  vpc_id = aws_vpc.main.id
    tags = {
    Name        =  format("PrivateA-RT, %s!",var.environment)
    Environment = var.environment
  }
}

resource "aws_route" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  route_table_id         = aws_route_table.private[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.nat[count.index].id
}

resource "aws_route_table_association" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```  
### Create aws routes for second private subnet
Add to the same file `route_tables.tf` . Add the following code to it 
```
resource "aws_route_table" "private_B" {
  count = var.preferred_number_of_private_subnets_B == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_B
  vpc_id = aws_vpc.main.id
    tags = {
    Name        =  format("privateB-RT, %s!",var.environment)
    Environment = var.environment
  }
}

resource "aws_route" "private_B" {
  count = var.preferred_number_of_private_subnets_B == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_B
  route_table_id         = aws_route_table.private_2[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.nat[count.index].id
}

resource "aws_route_table_association" "private_B" {
  count = var.preferred_number_of_private_subnets_B == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_B
  subnet_id      = aws_subnet.private_2[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```
Now if you did `terraform plan` and `terraform apply` it will add the following in `multi az`:
- Our main vpc 
- 2 public subnets 
- 4 private subnets 
- 2 internet gateway 
- 2 natgateway
- 2 EIPS
- 6 routetables

All of them are tagged resources as we set in `format` function.

You may check AWS Console to confirm

---

##  Identity and Access Management 
We want to pass an IAM role to our EC2s to give them access on some specic resources so will need to do the following: 

1. Create Assume Role 
    
Assume Role uses Security Token Service (STS) API that returns a set of temporary security credentials that you can use to access AWS resources that you might not normally have access to. These temporary credentials consist of an access key ID, a secret access key, and a security token. Typically, you use AssumeRole within your account or for cross-account access.

Add the following code to a new file named ***security.tf*** 
```
    resource "aws_iam_role" "ec2_instance_role" {
    name = "ec2_instance_role"
    assume_role_policy = jsonencode({
        "Version": "2012-10-17",
        "Statement": [
            {
            "Action": "sts:AssumeRole",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Effect": "Allow",
            "Sid": ""
            }
        ]
    })
    tags = {
        Name = "aws assume role"
        Environment = var.environment
        }
    }
  ```
 In the code above, we are creating AssumeRole with AssumeRole policy. It grants to an entity, in our case it is an EC2, permissions to assume the role. 

2. Create IAM policy to this role 
    This is where we need to define the required policy (i.e. permissions) according to the necessities. For example, allowing the IAM role to describe the ec2
    ```
    resource "aws_iam_policy" "policy" {
    name        = "test-policy"
    description = "A test policy"
    policy = jsonencode({
        "Version": "2012-10-17",
        "Statement": [
            {
            "Action": [
                "ec2:Describe*"
            ],
            "Effect": "Allow",
            "Resource": "*"
            }
        ]
    })
    tags = {
        Name = "aws assume policy"
        Environment = var.environment
       }
    }
    ```
3. Attach thie Policy to this role 
    
    This is where, we’ll be attaching the policy which we wrote above, to the role we created in the first step.
    ```    
    resource "aws_iam_role_policy_attachment" "test-attach" {
        role       = aws_iam_role.test_role.name
        policy_arn = aws_iam_policy.policy.arn
    }
    ```

4. Create Instance Profile and interpolate the IAM role
    ```
    resource "aws_iam_instance_profile" "ip" {
        name = "aws_instance_profile_test"
        role =  aws_iam_role.test_role.name
    }
    ```

## Create resources inside public subnet
As per our architecture we need to create the following resources inside the public subnet:

1. Upload the ssh key to keypairs 
2. 2 Bastion Ec2s with security groups
2. ALB 
3. Auto Scaling Group    

### Upload SSH key-pair to aws keypairs 
In order to be able to ssh to your machines you need to upload your ssh-key to aws keypair 
to generate yours just run the following command: `ssh-keygen` and just follow the instructions and save your public and private key.

To Upload your public key to aws keypair add the following code to `keypair.tf` file
```
resource "aws_key_pair" "Dare" {
  key_name   = "Dare"
  public_key = file("C:/Users/Dare/.ssh/id_rsa.pub")
}
``` 
### Create EC2 instances
In order to create EC2 we need first to Create security group to allow the 80, 443 and 22 ports 

To do it create a `security_group.tf` file  and add the following code to it:
```
resource "aws_security_group" "bastion_sg" {
    name = "vpc_web_sg"
    description = "Allow incoming HTTP connections."

    ingress {
        description = "SSH"
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    ingress {
        description = "HTTPS"
        from_port   = 443
        to_port     = 443
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    ingress {
        description = "HTTP"
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

    vpc_id = aws_vpc.main.id

    tags = {
        Name = "Bastion-SG"
        Environment = var.environment
    }
}
```
Add the EC2 configuration to `ec2.tf` file as follows:
```
resource "aws_instance" "bastion" {
    count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
    key_name      = aws_key_pair.Dare.key_name
    ami           = var.ami
    instance_type = "t2.micro"
    vpc_security_group_ids = [
        aws_security_group.bastion_sg.id
    ]
    iam_instance_profile = aws_iam_instance_profile.ip.name
    subnet_id = element(aws_subnet.public.*.id,count.index)
    associate_public_ip_address = true
    source_dest_check = false
    user_data = << EOF
		  #! /bin/bash
      echo "Hello from the other side"
	EOF
    tags = {
        Name = "Bastion-Test${count.index}"
    }
}
```
This creates 2 EC2s one in each subnet using the instance profile which we created before
 
---

**NOTE:**


In the above code inside `ec2.tf` you can find something called `user_data`. 

In many cases we might want to pass commands to the machine once it has been created, `user_data` helps us to do that. We can pass to file path or inline bash script  

---

**NOTE:** 

We are using the ami id multiple times in ASG and ec2 so we need to create it as variable so you need to add it to your variables.tf file 

```
variable "ami" {
  default = "ami-06255644cfdfdb869"
}
```
---
### Create ALB 

Create a file called `alb.tf`

First of all we need to create a security group for this ALB in `security_groups.tf`

```
resource "aws_security_group" "my-alb-sg" {
  name   = "my-alb-sg"
  vpc_id = aws_vpc.main.id
}

resource "aws_security_group_rule" "inbound_http" {
  from_port         = 80
  protocol          = "tcp"
  security_group_id = aws_security_group.my-alb-sg.id
  to_port           = 80
  type              = "ingress"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "outbound_all" {
  from_port         = 0
  protocol          = "-1"
  security_group_id = aws_security_group.my-alb-sg.id
  to_port           = 0
  type              = "egress"
  cidr_blocks       = ["0.0.0.0/0"]
}
```

We need to Create ALB to work as load balancer between EC2s 
```
resource "aws_lb" "my-aws-alb" {
  name     = "my-test-alb"
  internal = false
  security_groups = [
    aws_security_group.my-alb-sg.id,
  ]
  subnets = [
    aws_subnet.public[0].id,
    aws_subnet.public[1].id
  ]
  tags = {
    Name = "my-test-alb"
  }
  ip_address_type    = "ipv4"
  load_balancer_type = "application"
}
```

To be able to let the ALB routes to  the EC2s we need to create <b>Target Group</b> to know its target 

```
resource "aws_lb_target_group" "my-target-group" {
  health_check {
    interval            = 10
    path                = "/"
    protocol            = "HTTP"
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }

  name        = "my-test-tg"
  port        = 80
  protocol    = "HTTP"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id
}
```
Then we will need to Create Listner to this target Group
```
resource "aws_lb_listener" "my-test-alb-listner" {
  load_balancer_arn = aws_lb.my-aws-alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.my-target-group.arn
  }
}
```

Add the following outputs to `output.tf` to print them on screen  
```
output "alb_dns_name" {
  value = aws_lb.my-aws-alb.dns_name
}

output "alb_target_group_arn" {
  value = aws_lb_target_group.my-target-group.arn
}
```
### Create ASG (Auto Scaling Group)

here we need to configure our ASG to be able to scale the EC2s Up and down depends on the application trrafic

1. As every new resource lets create resources group in `security_groups.tf`
```

resource "aws_security_group" "my-asg-sg" {
  name   = "my-asg-sg"
  vpc_id = aws_vpc.main.id
}

resource "aws_security_group_rule" "inbound_ssh-asg" {
  from_port         = 22
  protocol          = "tcp"
  security_group_id = aws_security_group.my-asg-sg.id
  to_port           = 22
  type              = "ingress"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "inbound_http-asg" {
  from_port         = 80
  protocol          = "tcp"
  security_group_id = aws_security_group.my-asg-sg.id
  to_port           = 80
  type              = "ingress"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "outbound_all-asg" {
  from_port         = 0
  protocol          = "-1"
  security_group_id = aws_security_group.my-asg-sg.id
  to_port           = 0
  type              = "egress"
  cidr_blocks       = ["0.0.0.0/0"]
}
```

2. we need to create the lunch configuration of this asg in `asg.tf` .  # Ask the students to refactor this code to use Launch templates.

```
resource "aws_launch_configuration" "my-test-launch-config" {
  image_id        = var.ami
  instance_type   = "t2.micro"
  security_groups = [aws_security_group.my-asg-sg.id]

  user_data = <<-EOF
              #!/bin/bash
              yum -y install httpd
              echo "Hello, from Terraform" > /var/www/html/index.html
              service httpd start
              chkconfig httpd on
              EOF

  lifecycle {
    create_before_destroy = true
  }
}
```
3. Lets create the ASG itself in `asg.tf`
```
resource "aws_autoscaling_group" "public_asg" {
  launch_configuration = aws_launch_configuration.my-test-launch-config.name
  vpc_zone_identifier  = [
    aws_subnet.public[0].id,
    aws_subnet.public[1].id
  ]
  target_group_arns    = [aws_lb_target_group.my-target-group.arn]
  health_check_type    = "EC2"

  min_size = 2
  max_size = 10

  tag {
    key                 = "Name"
    value               = "my-test-asg"
    propagate_at_launch = true
  }
}
```

## Repeat all of the creation of EC2, ALB, ASG for private subnet

As per our architecture we need to create the following resources inside the private subnet:

1. 2 Ec2s with security groups
2. ALB 
3. Auto Scaling Group

---
**Note:** 

We are goinf to use some created resources we created while creating resources in public subnet as there resources are genreic and doesn't request specifec subnet 

---
### Create 2 EC2s

as we did before inside the public subnet we will create 2 ec2s with security groups

lets add the following code to `security_groups.tf`

```
resource "aws_security_group" "private_sg" {
    name = "vpc_private"
    description = "Allow incoming HTTP connections."

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]

  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP"
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

    vpc_id = aws_vpc.main.id

    tags = {
        Name = "private-sg"
    }
}

```

Then we will be able to create the 2 EC2s by adding the following code to `ec2.tf`

```
resource "aws_instance" "private" {
    count  = var.preferred_number_of_private_subnets_1 == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_1   
    key_name      = aws_key_pair.Dare.key_name
    ami           = var.ami
    instance_type = "t2.micro"
    vpc_security_group_ids = [
        aws_security_group.private_sg.id
    ]
    subnet_id = element(aws_subnet.private.*.id,count.index)
    associate_public_ip_address = false
    source_dest_check = false
    tags = {
        Name = "private-Test${count.index}"
    }
}
```

### Create ALB 

As we did beofre in public subnet we will create new ALB but it will be private one add the following code to `alb.tf` file 

```
resource "aws_lb" "my-aws-alb-private" {
  name     = "my-test-alb-privare"
  internal = true
  security_groups = [
    aws_security_group.my-alb-sg.id,
  ]
  subnets = [
    aws_subnet.private[0].id,
    aws_subnet.private[1].id
  ]
  tags = {
    Name = "my-test-alb-private"
  }
  ip_address_type    = "ipv4"
  load_balancer_type = "application"
}
```
We used here the same security froup that we used for the public subnet one and for sure we can create new one 

We need to Create listner to this ALB and we will use same target group we used in public subnet, Add the following code to `alb.tf`

```
resource "aws_lb_listener" "my-test-alb-listner-private" {
  load_balancer_arn = aws_lb.my-aws-alb-private.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.my-target-group.arn
  }
}
```

### Create ASG 
As we did for the public one we will do the same for private one add the following code to `asg.tf`
```
resource "aws_autoscaling_group" "private_asg" {
  launch_configuration = aws_launch_configuration.my-test-launch-config.name
  vpc_zone_identifier  = [
    aws_subnet.private[0].id,
    aws_subnet.private[1].id
  ]
  target_group_arns    = [aws_lb_target_group.my-target-group.arn]
  health_check_type    = "EC2"
  min_size = 2
  max_size = 10

  tag {
    key                 = "Name"
    value               = "private_asg"
    propagate_at_launch = true
  }
}

```



## Create EFS

Inorder to create EFS you need to create KMS key.
AWS Key Management Service (KMS) makes it easy for you to create and manage cryptographic keys and control their use across a wide range of AWS services and in your applications.
Add the following code to `kms.tf`
```
resource "aws_kms_key" "kms" {
  description = "KMS key "
  policy = <<EOF
  {
  "Version": "2012-10-17",
  "Id": "kms-key-policy",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::${var.account_no}:root"},
      "Action": "kms:*",
      "Resource": "*"
    }
  ]
}
EOF
}
resource "aws_kms_alias" "alias" {
  name          = "alias/kms"
  target_key_id = aws_kms_key.kms.key_id
}
```
lets create EFS security group as every resource inside `security_groups.tf`

```
resource "aws_security_group" "SG" {
  vpc_id      = aws_vpc.main.id
  name        = "SG"
  description = "Allow Inbound Traffic"

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  # ingress {
  #   from_port   = 0
  #   to_port     = 65535
  #   protocol    = "tcp"
  #   cidr_blocks = ["${var.vpc_cidr}"]
  # }
    ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = "2049"
    to_port     = "2049"
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }
  egress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    cidr_blocks = [var.vpc_cidr]
  }
  
  egress {
    from_port = 443
    to_port = 443
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

   egress {
    from_port = 587
    to_port = 587
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }


  tags =  {
    Name = "efs-SG"
  }


}
```

Lets create EFS, add the following code to `efs.tf`

```
resource "aws_efs_file_system" "efs" {
  tags =  {
    Name = "efs"
  }
  encrypted = true
  kms_key_id = "${var.kms_arn}${aws_kms_key.kms.key_id}"
}
```

Then we need to create a mount volume. We will add two mount volumes 1 per availability zone

```
resource "aws_efs_mount_target" "mounta" {
  file_system_id  = aws_efs_file_system.efs.id
  subnet_id       = aws_subnet.private_2[0].id
  security_groups = [aws_security_group.SG.id]
}

resource "aws_efs_mount_target" "mountb" {
  file_system_id  = aws_efs_file_system.efs.id
  subnet_id       = aws_subnet.private_2[1].id
  security_groups = [aws_security_group.SG.id]
}

```
## Create RDS 

lets add RDS security Groups to `security_groups.tf`

```
resource "aws_security_group" "myapp_mysql_rds" {
  name        = "secuirty_group_web_mysqlserver"
  description = "Allow access to MySQL RDS"
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "rds_secuirty_group"
  }

}

resource "aws_security_group_rule" "security_rule" {
  security_group_id = aws_security_group.myapp_mysql_rds.id
  type              = "ingress"
  from_port         = 3306
  to_port           = 3306
  protocol          = "tcp"

  cidr_blocks = [
    var.vpc_cidr,
  ]
}
```
lets add RDS subnet groups to configure the high availabilty in out to AZ to `rds.tf` 

```
resource "aws_security_group" "myapp_mysql_rds" {
  name        = "secuirty_group_web_mysqlserver"
  description = "Allow access to MySQL RDS"
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "rds_secuirty_group"
  }

}
```
Lets Create the RDS, Add the following code to `rds.tf`

```
resource "aws_db_instance" "default" {
  allocated_storage    = 20
  storage_type         = "gp2"
  engine               = "mysql"
  engine_version       = "5.7"
  instance_class       = "db.t2.micro"
  name                 = "mydb"
  username             = "admin"
  password             = "admin1234"
  parameter_group_name = "default.mysql5.7"
  db_subnet_group_name = aws_db_subnet_group.db_subnet.name
  skip_final_snapshot  = true
  multi_az             = "true"
}
```

5.  Create an AMI from the Console 
    1. Select the EC2 Instance to create an AMI 
    2. Navigate to the AMi menu
    3. Name the AMI 
    4. Copy the AMI ID 
11. Create launch template for bastion host - https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/launch_template
    1.  Introduce userdata 
12. NLB and public facing ALB
13. Target groups 
14. Create bastion-asg.tf and nginx-tg.tf to configure the ASG respectively
15. Repeat the process for private subnets (Internal ALB, autoscaling for webservers)
16. Create EFS and RDS in the 3rd private subnet
17. Ensure security groups are configured to allow respective connectivities.
                --------Required updates to the diagram 
                1. Introduce Tooling applications - More servers within the private subnet
                1. Jenkins 
                2. Sonarqube
                3. Artifactory
18. Add more resources (HA is not considered here)
    1.  Jenkins Ec2
    2.  Sonarqube
    3.  Artifactory


# Automate Infrastructure With IAC using Terraform Part 2
<p>In the previous part, we were able to to provision a VPC with public subnet </p>

<p>In this part we are going to finalize all of the infrastructure based on the below architectural digram</p>

![tooling_project_15](https://user-images.githubusercontent.com/76074379/127383520-f081e1da-44d3-464d-bf4b-8b0e28273147.png)


---
## Create Private Subnets

* According to the architecture we need to create 4 private subnets. let's create the first two .  

In the `main.tf` file which we created before, please add the the snippet of code below

```
resource "aws_subnet" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4 , count.index + 1)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```
As we did before with public subnet we are doing the same with private subnet 
- Define a variable in `Variables.tf` file called  `preferred_number_of_private_subnets_A`
```
  variable "preferred_number_of_private_subnets_A" {
      default = null
}
```
- Update `terraform.tfvars` with the value below: 
```
preferred_number_of_private_subnets_A = 2
```

---

**NOTE:**

We are going to create lots of resources so we need to <b>tag </b> all resources we will create. Tags enable you to categorize your AWS resources in different ways, for example, by purpose, owner, or environment. This is useful when you have many resources of the same type
```
tags = merge(
    var.additional_tags,
    {
      Name = format("PrivateSubnetA-%s", count.index)
    } 
  )
}
```
The `format` function produces a string by formatting a number of other values according to a specification string. It is similar to the **printf** function in C, and other similar functions in other programming languages.
`format(spec, values...)`.

We create everything with `multi AZ` so we need to differentiate between them. This will help us to create good tagging sting by adding for example the count of the resource.

---
Update your private subnet resource to be as below

```
resource "aws_subnet" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4 , count.index + 1)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = merge(
    var.additional_tags,
    {
      Name = format("PrivateSubnetA-%s", count.index)
    } 
  )
}
```
All we did was just to add tag block to be able to create the tag we want.

---
Update ***variables.tf*** file to define a variable called **additional_tags** since it will be used more than once as we will be tagging our other resources we need to tag.

```
variable "additional_tags" {
  default     = {}
  description = "Additional resource tags"
  type        = map(string)
}
```

Our full `main.tf` file after adding tags will be as below
```
# Get list of availability zones
data "aws_availability_zones" "available" {
state = "available"
}

provider "aws" {
  region = var.region
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support 
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink
  tags = {
    Name = "PublicSubnet"
    Environment = var.environment
  }
}

# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index +3)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  
}
resource "aws_subnet" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4 , count.index + 1)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = merge(
    var.additional_tags,
    {
      Name = format("PrivateSubnetA-%s", count.index)
    } 
  )
}

```
Now lets run command 
```
terraform plan
```
If we are okay with the output, then run
```
terraform apply -auto-aprrove
```


Check the AWS Console, we will find that aws has been updated with new subnets


---

## Replicate what did we do for the other private subnet 
- Make sure to update `variables.tf` file and `terraform.tfvars` file and then add the code below to `main.tf`
```
resource "aws_subnet" "private_B" {
  count = var.preferred_number_of_private_subnets_B == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_B
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4 , count.index + 7)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = merge(
    var.additional_tags,
    {
      Name = format("PrivateSubnetB-%s", count.index)
    } 
  )
}
```
---
**NOTE:** The main aim of using terraform is to convert our infrastructure to code and this gives us the opportunity to organize our resources and one of the best practices that terraform gives. <b>It gives us the power to organize our resources in seprate files.</b>

---

## Create internet gateway
- create a file called `internet_gateway.tf` and add the code below
```
resource "aws_internet_gateway" "ig" {
  vpc_id = aws_vpc.main.id
  tags = merge(
    var.additional_tags,
    {
      Name = format("%s-%s!", aws_vpc.main.id,"IG")
    } 
  )
}
```
## Create an Elastic IP(EIP) and AWS NATgateway and allocate the EIP to the NATgateway
create a new file called `natgateway.tf` and the following code to it 
```
resource "aws_eip" "nat_eip" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc        = true
  depends_on = [aws_internet_gateway.ig]
  tags = {
    Name        =  format("EIP-%s",var.environment)
    Environment = var.environment
  }
}

resource "aws_nat_gateway" "nat" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  allocation_id = aws_eip.nat_eip[count.index].id
  subnet_id     = element(aws_subnet.public.*.id, 0)
  depends_on    = [aws_internet_gateway.ig]
  tags = {
    Name        =  format("Nat-%s",var.environment)
    Environment = var.environment
  }
}
```

**NOTE**: As we can see in the code snippet above, we included a new variable called *Environment* in the tag block. So we will have to define that variable in the ***variable.tf*** file. Add the code below
```
variable "environment" {
  default = "for-private-subnets"
}
```

## Create AWS Routes 
### Create aws routes for public subnet
Create a file called `route_tables.tf` add the following code to it 

```
resource "aws_route_table" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets
  vpc_id = aws_vpc.main.id
  tags = {
    Name        = format("Public-RT-%s", var.environment)
    Environment = var.environment
  }
}
resource "aws_route" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets
  route_table_id         = aws_route_table.public[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.ig.id
}

resource "aws_route_table_association" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public[count.index].id
}

```
### Create aws routes for first private subnet
Add to the same file `route_tables.tf` . Add the following code to it 
```
resource "aws_route_table" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  vpc_id = aws_vpc.main.id
    tags = {
    Name        =  format("PrivateA-RT, %s!",var.environment)
    Environment = var.environment
  }
}

resource "aws_route" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  route_table_id         = aws_route_table.private[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.nat[count.index].id
}

resource "aws_route_table_association" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```  
### Create aws routes for second private subnet
Add to the same file `route_tables.tf` . Add the following code to it 
```
resource "aws_route_table" "private_B" {
  count = var.preferred_number_of_private_subnets_B == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_B
  vpc_id = aws_vpc.main.id
    tags = {
    Name        =  format("privateB-RT, %s!",var.environment)
    Environment = var.environment
  }
}

resource "aws_route" "private_B" {
  count = var.preferred_number_of_private_subnets_B == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_B
  route_table_id         = aws_route_table.private_2[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.nat[count.index].id
}

resource "aws_route_table_association" "private_B" {
  count = var.preferred_number_of_private_subnets_B == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_B
  subnet_id      = aws_subnet.private_2[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```
Now if you did `terraform plan` and `terraform apply` it will add the following in `multi az`:
- Our main vpc 
- 2 public subnets 
- 4 private subnets 
- 2 internet gateway 
- 2 natgateway
- 2 EIPS
- 6 routetables

All of them are tagged resources as we set in `format` function.

You may check AWS Console to confirm

---

##  Identity and Access Management 
We want to pass an IAM role to our EC2s to give them access on some specic resources so will need to do the following: 

1. Create Assume Role 
    
Assume Role uses Security Token Service (STS) API that returns a set of temporary security credentials that you can use to access AWS resources that you might not normally have access to. These temporary credentials consist of an access key ID, a secret access key, and a security token. Typically, you use AssumeRole within your account or for cross-account access.

Add the following code to a new file named ***security.tf***   
    ```
    resource "aws_iam_role" "ec2_instance_role" {
    name = "ec2_instance_role"
    assume_role_policy = jsonencode({
        "Version": "2012-10-17",
        "Statement": [
            {
            "Action": "sts:AssumeRole",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Effect": "Allow",
            "Sid": ""
            }
        ]
    })
    tags = {
        Name = "aws assume role"
        Environment = var.environment
        }
    }
    ```
 In the code above, we are creating AssumeRole with AssumeRole policy. It grants to an entity, in our case it is an EC2, permissions to assume the role. 

2. Create IAM policy to this role 
    This is where we need to define the required policy (i.e. permissions) according to the necessities. For example, allowing the IAM role to describe the ec2
    ```
    resource "aws_iam_policy" "policy" {
        name        = "test-policy"
        description = "A test policy"
        policy = <<EOF
        {
        "Version": "2012-10-17",
        "Statement": [
            {
            "Action": [
                "ec2:Describe*"
            ],
            "Effect": "Allow",
            "Resource": "*"
            }
        ]
        }
        EOF
        tags = {
            Name = "aws assume policy"
            Environment = var.environment
        }
    }
    ```
3. Attach thie Policy to this role 
    
    This is where, we’ll be attaching the policy which we wrote above, to the role we created in the first step.
    ```    
    resource "aws_iam_role_policy_attachment" "test-attach" {
        role       = aws_iam_role.test_role.name
        policy_arn = aws_iam_policy.policy.arn
    }
    ```

4. Create Instance Profile and interpolate the IAM role
    ```
    resource "aws_iam_instance_profile" "ip" {
        name = "aws_instance_profile_test"
        role =  aws_iam_role.test_role.name
    }
    ```

## Create resources inside public subnet
As per our architecture we need to create the following resources inside the public subnet:

1. Upload the ssh key to keypairs 
2. 2 Bastion Ec2s with security groups
2. ALB 
3. Auto Scaling Group    

### Upload SSH key-pair to aws keypairs 
In order to be able to ssh to your machines you need to upload your ssh-key to aws keypair 
to generate yours just run the following command: `ssh-keygen` and just follow the instructions and save your public and private key.

To Upload your public key to aws keypair add the following code to `keypair.tf` file
```
resource "aws_key_pair" "Dare" {
  key_name   = "Dare"
  public_key = file("C:/Users/Dare/.ssh/id_rsa.pub")
}
``` 
### Create EC2 instances
In order to create EC2 we need first to Create security group to allow the 80, 443 and 22 ports 

To do it create a `security_group.tf` file  and add the following code to it:
```
resource "aws_security_group" "bastion_sg" {
    name = "vpc_web_sg"
    description = "Allow incoming HTTP connections."

    ingress {
        description = "SSH"
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    ingress {
        description = "HTTPS"
        from_port   = 443
        to_port     = 443
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    ingress {
        description = "HTTP"
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

    vpc_id = aws_vpc.main.id

    tags = {
        Name = "Bastion-SG"
        Environment = var.environment
    }
}
```
Add the EC2 configuration to `ec2.tf` file as follows:
```
resource "aws_instance" "bastion" {
    count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
    key_name      = aws_key_pair.Dare.key_name
    ami           = var.ami
    instance_type = "t2.micro"
    vpc_security_group_ids = [
        aws_security_group.bastion_sg.id
    ]
    iam_instance_profile = aws_iam_instance_profile.ip.name
    subnet_id = element(aws_subnet.public.*.id,count.index)
    associate_public_ip_address = true
    source_dest_check = false
    user_data = << EOF
		  #! /bin/bash
      echo "Hello from the other side"
	EOF
    tags = {
        Name = "Bastion-Test${count.index}"
    }
}
```
This creates 2 EC2s one in each subnet using the instance profile which we created before
 
---

**NOTE:**


In the above code inside `ec2.tf` you can find something called `user_data`. 

In many cases we might want to pass commands to the machine once it has been created, `user_data` helps us to do that. We can pass to file path or inline bash script  

---

**NOTE:** 

We are using the ami id multiple times in ASG and ec2 so we need to create it as variable so you need to add it to your variables.tf file 

```
variable "ami" {
  default = "ami-06255644cfdfdb869"
}
```
---
### Create ALB 

Create a file called `alb.tf`

First of all we need to create a security group for this ALB in `security_groups.tf`

```
resource "aws_security_group" "my-alb-sg" {
  name   = "my-alb-sg"
  vpc_id = aws_vpc.main.id
}

resource "aws_security_group_rule" "inbound_http" {
  from_port         = 80
  protocol          = "tcp"
  security_group_id = aws_security_group.my-alb-sg.id
  to_port           = 80
  type              = "ingress"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "outbound_all" {
  from_port         = 0
  protocol          = "-1"
  security_group_id = aws_security_group.my-alb-sg.id
  to_port           = 0
  type              = "egress"
  cidr_blocks       = ["0.0.0.0/0"]
}
```

We need to Create ALB to work as load balancer between EC2s 
```
resource "aws_lb" "my-aws-alb" {
  name     = "my-test-alb"
  internal = false
  security_groups = [
    aws_security_group.my-alb-sg.id,
  ]
  subnets = [
    aws_subnet.public[0].id,
    aws_subnet.public[1].id
  ]
  tags = {
    Name = "my-test-alb"
  }
  ip_address_type    = "ipv4"
  load_balancer_type = "application"
}
```

To be able to let the ALB routes to  the EC2s we need to create <b>Target Group</b> to know its target 

```
resource "aws_lb_target_group" "my-target-group" {
  health_check {
    interval            = 10
    path                = "/"
    protocol            = "HTTP"
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }

  name        = "my-test-tg"
  port        = 80
  protocol    = "HTTP"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id
}
```
Then we will need to Create Listner to this target Group
```
resource "aws_lb_listener" "my-test-alb-listner" {
  load_balancer_arn = aws_lb.my-aws-alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.my-target-group.arn
  }
}
```

Add the following outputs to `output.tf` to print them on screen  
```
output "alb_dns_name" {
  value = aws_lb.my-aws-alb.dns_name
}

output "alb_target_group_arn" {
  value = aws_lb_target_group.my-target-group.arn
}
```
### Create ASG (Auto Scaling Group)

here we need to configure our ASG to be able to scale the EC2s Up and down depends on the application trrafic

1. As every new resource lets create resources group in `security_groups.tf`
```

resource "aws_security_group" "my-asg-sg" {
  name   = "my-asg-sg"
  vpc_id = aws_vpc.main.id
}

resource "aws_security_group_rule" "inbound_ssh-asg" {
  from_port         = 22
  protocol          = "tcp"
  security_group_id = aws_security_group.my-asg-sg.id
  to_port           = 22
  type              = "ingress"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "inbound_http-asg" {
  from_port         = 80
  protocol          = "tcp"
  security_group_id = aws_security_group.my-asg-sg.id
  to_port           = 80
  type              = "ingress"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "outbound_all-asg" {
  from_port         = 0
  protocol          = "-1"
  security_group_id = aws_security_group.my-asg-sg.id
  to_port           = 0
  type              = "egress"
  cidr_blocks       = ["0.0.0.0/0"]
}
```

2. we need to create the lunch configuration of this asg in `asg.tf` .  # Ask the students to refactor this code to use Launch templates.

```
resource "aws_launch_configuration" "my-test-launch-config" {
  image_id        = var.ami
  instance_type   = "t2.micro"
  security_groups = [aws_security_group.my-asg-sg.id]

  user_data = <<-EOF
              #!/bin/bash
              yum -y install httpd
              echo "Hello, from Terraform" > /var/www/html/index.html
              service httpd start
              chkconfig httpd on
              EOF

  lifecycle {
    create_before_destroy = true
  }
}
```
3. Lets create the ASG itself in `asg.tf`
```
resource "aws_autoscaling_group" "public_asg" {
  launch_configuration = aws_launch_configuration.my-test-launch-config.name
  vpc_zone_identifier  = [
    aws_subnet.public[0].id,
    aws_subnet.public[1].id
  ]
  target_group_arns    = [aws_lb_target_group.my-target-group.arn]
  health_check_type    = "EC2"

  min_size = 2
  max_size = 10

  tag {
    key                 = "Name"
    value               = "my-test-asg"
    propagate_at_launch = true
  }
}
```

## Repeat all of the creation of EC2, ALB, ASG for private subnet

As per our architecture we need to create the following resources inside the private subnet:

1. 2 Ec2s with security groups
2. ALB 
3. Auto Scaling Group

---
**Note:** 

We are goinf to use some created resources we created while creating resources in public subnet as there resources are genreic and doesn't request specifec subnet 

---
### Create 2 EC2s

as we did before inside the public subnet we will create 2 ec2s with security groups

lets add the following code to `security_groups.tf`

```
resource "aws_security_group" "private_sg" {
    name = "vpc_private"
    description = "Allow incoming HTTP connections."

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]

  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP"
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

    vpc_id = aws_vpc.main.id

    tags = {
        Name = "private-sg"
    }
}

```

Then we will be able to create the 2 EC2s by adding the following code to `ec2.tf`

```
resource "aws_instance" "private" {
    count  = var.preferred_number_of_private_subnets_1 == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_1   
    key_name      = aws_key_pair.Dare.key_name
    ami           = var.ami
    instance_type = "t2.micro"
    vpc_security_group_ids = [
        aws_security_group.private_sg.id
    ]
    subnet_id = element(aws_subnet.private.*.id,count.index)
    associate_public_ip_address = false
    source_dest_check = false
    tags = {
        Name = "private-Test${count.index}"
    }
}
```

### Create ALB 

As we did beofre in public subnet we will create new ALB but it will be private one add the following code to `alb.tf` file 

```
resource "aws_lb" "my-aws-alb-private" {
  name     = "my-test-alb-privare"
  internal = true
  security_groups = [
    aws_security_group.my-alb-sg.id,
  ]
  subnets = [
    aws_subnet.private[0].id,
    aws_subnet.private[1].id
  ]
  tags = {
    Name = "my-test-alb-private"
  }
  ip_address_type    = "ipv4"
  load_balancer_type = "application"
}
```
We used here the same security froup that we used for the public subnet one and for sure we can create new one 

We need to Create listner to this ALB and we will use same target group we used in public subnet, Add the following code to `alb.tf`

```
resource "aws_lb_listener" "my-test-alb-listner-private" {
  load_balancer_arn = aws_lb.my-aws-alb-private.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.my-target-group.arn
  }
}
```

### Create ASG 
As we did for the public one we will do the same for private one add the following code to `asg.tf`
```
resource "aws_autoscaling_group" "private_asg" {
  launch_configuration = aws_launch_configuration.my-test-launch-config.name
  vpc_zone_identifier  = [
    aws_subnet.private[0].id,
    aws_subnet.private[1].id
  ]
  target_group_arns    = [aws_lb_target_group.my-target-group.arn]
  health_check_type    = "EC2"
  min_size = 2
  max_size = 10

  tag {
    key                 = "Name"
    value               = "private_asg"
    propagate_at_launch = true
  }
}

```



## Create EFS

Inorder to create EFS you need to create KMS key.
AWS Key Management Service (KMS) makes it easy for you to create and manage cryptographic keys and control their use across a wide range of AWS services and in your applications.
Add the following code to `kms.tf`
```
resource "aws_kms_key" "kms" {
  description = "KMS key "
  policy = <<EOF
  {
  "Version": "2012-10-17",
  "Id": "kms-key-policy",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::${var.account_no}:root"},
      "Action": "kms:*",
      "Resource": "*"
    }
  ]
}
EOF
}
resource "aws_kms_alias" "alias" {
  name          = "alias/kms"
  target_key_id = aws_kms_key.kms.key_id
}
```
lets create EFS security group as every resource inside `security_groups.tf`

```
resource "aws_security_group" "SG" {
  vpc_id      = aws_vpc.main.id
  name        = "SG"
  description = "Allow Inbound Traffic"

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  # ingress {
  #   from_port   = 0
  #   to_port     = 65535
  #   protocol    = "tcp"
  #   cidr_blocks = ["${var.vpc_cidr}"]
  # }
    ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = "2049"
    to_port     = "2049"
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }
  egress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    cidr_blocks = [var.vpc_cidr]
  }
  
  egress {
    from_port = 443
    to_port = 443
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

   egress {
    from_port = 587
    to_port = 587
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }


  tags =  {
    Name = "efs-SG"
  }


}
```

Lets create EFS, add the following code to `efs.tf`

```
resource "aws_efs_file_system" "efs" {
  tags =  {
    Name = "efs"
  }
  encrypted = true
  kms_key_id = "${var.kms_arn}${aws_kms_key.kms.key_id}"
}
```

Then we need to create a mount volume. We will add two mount volumes 1 per availability zone

```
resource "aws_efs_mount_target" "mounta" {
  file_system_id  = aws_efs_file_system.efs.id
  subnet_id       = aws_subnet.private_2[0].id
  security_groups = [aws_security_group.SG.id]
}

resource "aws_efs_mount_target" "mountb" {
  file_system_id  = aws_efs_file_system.efs.id
  subnet_id       = aws_subnet.private_2[1].id
  security_groups = [aws_security_group.SG.id]
}

```
## Create RDS 

lets add RDS security Groups to `security_groups.tf`

```
resource "aws_security_group" "myapp_mysql_rds" {
  name        = "secuirty_group_web_mysqlserver"
  description = "Allow access to MySQL RDS"
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "rds_secuirty_group"
  }

}

resource "aws_security_group_rule" "security_rule" {
  security_group_id = aws_security_group.myapp_mysql_rds.id
  type              = "ingress"
  from_port         = 3306
  to_port           = 3306
  protocol          = "tcp"

  cidr_blocks = [
    var.vpc_cidr,
  ]
}
```
lets add RDS subnet groups to configure the high availabilty in out to AZ to `rds.tf` 

```
resource "aws_security_group" "myapp_mysql_rds" {
  name        = "secuirty_group_web_mysqlserver"
  description = "Allow access to MySQL RDS"
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "rds_secuirty_group"
  }

}
```
Lets Create the RDS, Add the following code to `rds.tf`

```
resource "aws_db_instance" "default" {
  allocated_storage    = 20
  storage_type         = "gp2"
  engine               = "mysql"
  engine_version       = "5.7"
  instance_class       = "db.t2.micro"
  name                 = "mydb"
  username             = "admin"
  password             = "admin1234"
  parameter_group_name = "default.mysql5.7"
  db_subnet_group_name = aws_db_subnet_group.db_subnet.name
  skip_final_snapshot  = true
  multi_az             = "true"
}
```

5.  Create an AMI from the Console 
    1. Select the EC2 Instance to create an AMI 
    2. Navigate to the AMi menu
    3. Name the AMI 
    4. Copy the AMI ID 
11. Create launch template for bastion host - https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/launch_template
    1.  Introduce userdata 
12. NLB and public facing ALB
13. Target groups 
14. Create bastion-asg.tf and nginx-tg.tf to configure the ASG respectively
15. Repeat the process for private subnets (Internal ALB, autoscaling for webservers)
16. Create EFS and RDS in the 3rd private subnet
17. Ensure security groups are configured to allow respective connectivities.
                --------Required updates to the diagram 
                1. Introduce Tooling applications - More servers within the private subnet
                1. Jenkins 
                2. Sonarqube
                3. Artifactory
18. Add more resources (HA is not considered here)
    1.  Jenkins Ec2
    2.  Sonarqube
    3.  Artifactory


# Automate Infrastructure With IAC using Terraform Part 2
<p>In the previous part, we were able to to provision a VPC with public subnet </p>

<p>In this part we are going to finalize all of the infrastructure based on the below architectural digram</p>

![tooling_project_15](https://user-images.githubusercontent.com/76074379/127383520-f081e1da-44d3-464d-bf4b-8b0e28273147.png)


---
## Create Private Subnets

* According to the architecture we need to create 4 private subnets. let's create the first two .  

In the `main.tf` file which we created before, please add the the snippet of code below

```
resource "aws_subnet" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4 , count.index + 1)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```
As we did before with public subnet we are doing the same with private subnet 
- Define a variable in `Variables.tf` file called  `preferred_number_of_private_subnets_A`
```
  variable "preferred_number_of_private_subnets_A" {
      default = null
}
```
- Update `terraform.tfvars` with the value below: 
```
preferred_number_of_private_subnets_A = 2
```

---

**NOTE:**

We are going to create lots of resources so we need to <b>tag </b> all resources we will create. Tags enable you to categorize your AWS resources in different ways, for example, by purpose, owner, or environment. This is useful when you have many resources of the same type
```
tags = merge(
    var.additional_tags,
    {
      Name = format("PrivateSubnetA-%s", count.index)
    } 
  )
}
```
The `format` function produces a string by formatting a number of other values according to a specification string. It is similar to the **printf** function in C, and other similar functions in other programming languages.
`format(spec, values...)`.

We create everything with `multi AZ` so we need to differentiate between them. This will help us to create good tagging sting by adding for example the count of the resource.

---
Update your private subnet resource to be as below

```
resource "aws_subnet" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4 , count.index + 1)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = merge(
    var.additional_tags,
    {
      Name = format("PrivateSubnetA-%s", count.index)
    } 
  )
}
```
All we did was just to add tag block to be able to create the tag we want.

---
Update ***variables.tf*** file to define a variable called **additional_tags** since it will be used more than once as we will be tagging our other resources we need to tag.

```
variable "additional_tags" {
  default     = {}
  description = "Additional resource tags"
  type        = map(string)
}
```

Our full `main.tf` file after adding tags will be as below
```
# Get list of availability zones
data "aws_availability_zones" "available" {
state = "available"
}

provider "aws" {
  region = var.region
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support 
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink
  tags = {
    Name = "PublicSubnet"
    Environment = var.environment
  }
}

# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index +3)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  
}
resource "aws_subnet" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4 , count.index + 1)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = merge(
    var.additional_tags,
    {
      Name = format("PrivateSubnetA-%s", count.index)
    } 
  )
}

```
Now lets run command 
```
terraform plan
```
If we are okay with the output, then run
```
terraform apply -auto-aprrove
```


Check the AWS Console, we will find that aws has been updated with new subnets


---

## Replicate what did we do for the other private subnet 
- Make sure to update `variables.tf` file and `terraform.tfvars` file and then add the code below to `main.tf`
```
resource "aws_subnet" "private_B" {
  count = var.preferred_number_of_private_subnets_B == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_B
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4 , count.index + 7)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = merge(
    var.additional_tags,
    {
      Name = format("PrivateSubnetB-%s", count.index)
    } 
  )
}
```
---
**NOTE:** The main aim of using terraform is to convert our infrastructure to code and this gives us the opportunity to organize our resources and one of the best practices that terraform gives. <b>It gives us the power to organize our resources in seprate files.</b>

---

## Create internet gateway
- create a file called `internet_gateway.tf` and add the code below
```
resource "aws_internet_gateway" "ig" {
  vpc_id = aws_vpc.main.id
  tags = merge(
    var.additional_tags,
    {
      Name = format("%s-%s!", aws_vpc.main.id,"IG")
    } 
  )
}
```
## Create an Elastic IP(EIP) and AWS NATgateway and allocate the EIP to the NATgateway
create a new file called `natgateway.tf` and the following code to it 
```
resource "aws_eip" "nat_eip" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc        = true
  depends_on = [aws_internet_gateway.ig]
  tags = {
    Name        =  format("EIP-%s",var.environment)
    Environment = var.environment
  }
}

resource "aws_nat_gateway" "nat" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  allocation_id = aws_eip.nat_eip[count.index].id
  subnet_id     = element(aws_subnet.public.*.id, 0)
  depends_on    = [aws_internet_gateway.ig]
  tags = {
    Name        =  format("Nat-%s",var.environment)
    Environment = var.environment
  }
}
```

**NOTE**: As we can see in the code snippet above, we included a new variable called *Environment* in the tag block. So we will have to define that variable in the ***variable.tf*** file. Add the code below
```
variable "environment" {
  default = "for-private-subnets"
}
```

## Create AWS Routes 
### Create aws routes for public subnet
Create a file called `route_tables.tf` add the following code to it 

```
resource "aws_route_table" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets
  vpc_id = aws_vpc.main.id
  tags = {
    Name        = format("Public-RT-%s", var.environment)
    Environment = var.environment
  }
}
resource "aws_route" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets
  route_table_id         = aws_route_table.public[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.ig.id
}

resource "aws_route_table_association" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public[count.index].id
}

```
### Create aws routes for first private subnet
Add to the same file `route_tables.tf` . Add the following code to it 
```
resource "aws_route_table" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  vpc_id = aws_vpc.main.id
    tags = {
    Name        =  format("PrivateA-RT, %s!",var.environment)
    Environment = var.environment
  }
}

resource "aws_route" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  route_table_id         = aws_route_table.private[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.nat[count.index].id
}

resource "aws_route_table_association" "private_A" {
  count = var.preferred_number_of_private_subnets_A == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_A
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```  
### Create aws routes for second private subnet
Add to the same file `route_tables.tf` . Add the following code to it 
```
resource "aws_route_table" "private_B" {
  count = var.preferred_number_of_private_subnets_B == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_B
  vpc_id = aws_vpc.main.id
    tags = {
    Name        =  format("privateB-RT, %s!",var.environment)
    Environment = var.environment
  }
}

resource "aws_route" "private_B" {
  count = var.preferred_number_of_private_subnets_B == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_B
  route_table_id         = aws_route_table.private_2[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.nat[count.index].id
}

resource "aws_route_table_association" "private_B" {
  count = var.preferred_number_of_private_subnets_B == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_B
  subnet_id      = aws_subnet.private_2[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```
Now if you did `terraform plan` and `terraform apply` it will add the following in `multi az`:
- Our main vpc 
- 2 public subnets 
- 4 private subnets 
- 2 internet gateway 
- 2 natgateway
- 2 EIPS
- 6 routetables

All of them are tagged resources as we set in `format` function.

You may check AWS Console to confirm

---

##  Identity and Access Management 
We want to pass an IAM role to our EC2s to give them access on some specic resources so will need to do the following: 

1. Create Assume Role 
    
Assume Role uses Security Token Service (STS) API that returns a set of temporary security credentials that you can use to access AWS resources that you might not normally have access to. These temporary credentials consist of an access key ID, a secret access key, and a security token. Typically, you use AssumeRole within your account or for cross-account access.

Add the following code to a new file named ***security.tf***   
    ```
    resource "aws_iam_role" "ec2_instance_role" {
    name = "ec2_instance_role"
    assume_role_policy = jsonencode({
        "Version": "2012-10-17",
        "Statement": [
            {
            "Action": "sts:AssumeRole",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Effect": "Allow",
            "Sid": ""
            }
        ]
    })
    tags = {
        Name = "aws assume role"
        Environment = var.environment
        }
    }
    ```
 In the code above, we are creating AssumeRole with AssumeRole policy. It grants to an entity, in our case it is an EC2, permissions to assume the role. 

2. Create IAM policy to this role 
    This is where we need to define the required policy (i.e. permissions) according to the necessities. For example, allowing the IAM role to describe the ec2
    ```
    resource "aws_iam_policy" "policy" {
        name        = "test-policy"
        description = "A test policy"
        policy = <<EOF
        {
        "Version": "2012-10-17",
        "Statement": [
            {
            "Action": [
                "ec2:Describe*"
            ],
            "Effect": "Allow",
            "Resource": "*"
            }
        ]
        }
        EOF
        tags = {
            Name = "aws assume policy"
            Environment = var.environment
        }
    }
    ```
3. Attach thie Policy to this role 
    
    This is where, we’ll be attaching the policy which we wrote above, to the role we created in the first step.
    ```    
    resource "aws_iam_role_policy_attachment" "test-attach" {
        role       = aws_iam_role.test_role.name
        policy_arn = aws_iam_policy.policy.arn
    }
    ```

4. Create Instance Profile and interpolate the IAM role
    ```
    resource "aws_iam_instance_profile" "ip" {
        name = "aws_instance_profile_test"
        role =  aws_iam_role.test_role.name
    }
    ```

## Create resources inside public subnet
As per our architecture we need to create the following resources inside the public subnet:

1. Upload the ssh key to keypairs 
2. 2 Bastion Ec2s with security groups
2. ALB 
3. Auto Scaling Group    

### Upload SSH key-pair to aws keypairs 
In order to be able to ssh to your machines you need to upload your ssh-key to aws keypair 
to generate yours just run the following command: `ssh-keygen` and just follow the instructions and save your public and private key.

To Upload your public key to aws keypair add the following code to `keypair.tf` file
```
resource "aws_key_pair" "Dare" {
  key_name   = "Dare"
  public_key = file("C:/Users/Dare/.ssh/id_rsa.pub")
}
``` 
### Create EC2 instances
In order to create EC2 we need first to Create security group to allow the 80, 443 and 22 ports 

To do it create a `security_group.tf` file  and add the following code to it:
```
resource "aws_security_group" "bastion_sg" {
    name = "vpc_web_sg"
    description = "Allow incoming HTTP connections."

    ingress {
        description = "SSH"
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    ingress {
        description = "HTTPS"
        from_port   = 443
        to_port     = 443
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    ingress {
        description = "HTTP"
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

    vpc_id = aws_vpc.main.id

    tags = {
        Name = "Bastion-SG"
        Environment = var.environment
    }
}
```
Add the EC2 configuration to `ec2.tf` file as follows:
```
resource "aws_instance" "bastion" {
    count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
    key_name      = aws_key_pair.Dare.key_name
    ami           = var.ami
    instance_type = "t2.micro"
    vpc_security_group_ids = [
        aws_security_group.bastion_sg.id
    ]
    iam_instance_profile = aws_iam_instance_profile.ip.name
    subnet_id = element(aws_subnet.public.*.id,count.index)
    associate_public_ip_address = true
    source_dest_check = false
    user_data = << EOF
		  #! /bin/bash
      echo "Hello from the other side"
	EOF
    tags = {
        Name = "Bastion-Test${count.index}"
    }
}
```
This creates 2 EC2s one in each subnet using the instance profile which we created before
 
---

**NOTE:**


In the above code inside `ec2.tf` you can find something called `user_data`. 

In many cases we might want to pass commands to the machine once it has been created, `user_data` helps us to do that. We can pass to file path or inline bash script  

---

**NOTE:** 

We are using the ami id multiple times in ASG and ec2 so we need to create it as variable so you need to add it to your variables.tf file 

```
variable "ami" {
  default = "ami-06255644cfdfdb869"
}
```
---
### Create ALB 

Create a file called `alb.tf`

First of all we need to create a security group for this ALB in `security_groups.tf`

```
resource "aws_security_group" "my-alb-sg" {
  name   = "my-alb-sg"
  vpc_id = aws_vpc.main.id
}

resource "aws_security_group_rule" "inbound_http" {
  from_port         = 80
  protocol          = "tcp"
  security_group_id = aws_security_group.my-alb-sg.id
  to_port           = 80
  type              = "ingress"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "outbound_all" {
  from_port         = 0
  protocol          = "-1"
  security_group_id = aws_security_group.my-alb-sg.id
  to_port           = 0
  type              = "egress"
  cidr_blocks       = ["0.0.0.0/0"]
}
```

We need to Create ALB to work as load balancer between EC2s 
```
resource "aws_lb" "my-aws-alb" {
  name     = "my-test-alb"
  internal = false
  security_groups = [
    aws_security_group.my-alb-sg.id,
  ]
  subnets = [
    aws_subnet.public[0].id,
    aws_subnet.public[1].id
  ]
  tags = {
    Name = "my-test-alb"
  }
  ip_address_type    = "ipv4"
  load_balancer_type = "application"
}
```

To be able to let the ALB routes to  the EC2s we need to create <b>Target Group</b> to know its target 

```
resource "aws_lb_target_group" "my-target-group" {
  health_check {
    interval            = 10
    path                = "/"
    protocol            = "HTTP"
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }

  name        = "my-test-tg"
  port        = 80
  protocol    = "HTTP"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id
}
```
Then we will need to Create Listner to this target Group
```
resource "aws_lb_listener" "my-test-alb-listner" {
  load_balancer_arn = aws_lb.my-aws-alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.my-target-group.arn
  }
}
```

Add the following outputs to `output.tf` to print them on screen  
```
output "alb_dns_name" {
  value = aws_lb.my-aws-alb.dns_name
}

output "alb_target_group_arn" {
  value = aws_lb_target_group.my-target-group.arn
}
```
### Create ASG (Auto Scaling Group)

here we need to configure our ASG to be able to scale the EC2s Up and down depends on the application trrafic

1. As every new resource lets create resources group in `security_groups.tf`
```

resource "aws_security_group" "my-asg-sg" {
  name   = "my-asg-sg"
  vpc_id = aws_vpc.main.id
}

resource "aws_security_group_rule" "inbound_ssh-asg" {
  from_port         = 22
  protocol          = "tcp"
  security_group_id = aws_security_group.my-asg-sg.id
  to_port           = 22
  type              = "ingress"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "inbound_http-asg" {
  from_port         = 80
  protocol          = "tcp"
  security_group_id = aws_security_group.my-asg-sg.id
  to_port           = 80
  type              = "ingress"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "outbound_all-asg" {
  from_port         = 0
  protocol          = "-1"
  security_group_id = aws_security_group.my-asg-sg.id
  to_port           = 0
  type              = "egress"
  cidr_blocks       = ["0.0.0.0/0"]
}
```

2. we need to create the lunch configuration of this asg in `asg.tf` .  # Ask the students to refactor this code to use Launch templates.

```
resource "aws_launch_configuration" "my-test-launch-config" {
  image_id        = var.ami
  instance_type   = "t2.micro"
  security_groups = [aws_security_group.my-asg-sg.id]

  user_data = <<-EOF
              #!/bin/bash
              yum -y install httpd
              echo "Hello, from Terraform" > /var/www/html/index.html
              service httpd start
              chkconfig httpd on
              EOF

  lifecycle {
    create_before_destroy = true
  }
}
```
3. Lets create the ASG itself in `asg.tf`
```
resource "aws_autoscaling_group" "public_asg" {
  launch_configuration = aws_launch_configuration.my-test-launch-config.name
  vpc_zone_identifier  = [
    aws_subnet.public[0].id,
    aws_subnet.public[1].id
  ]
  target_group_arns    = [aws_lb_target_group.my-target-group.arn]
  health_check_type    = "EC2"

  min_size = 2
  max_size = 10

  tag {
    key                 = "Name"
    value               = "my-test-asg"
    propagate_at_launch = true
  }
}
```

## Repeat all of the creation of EC2, ALB, ASG for private subnet

As per our architecture we need to create the following resources inside the private subnet:

1. 2 Ec2s with security groups
2. ALB 
3. Auto Scaling Group

---
**Note:** 

We are goinf to use some created resources we created while creating resources in public subnet as there resources are genreic and doesn't request specifec subnet 

---
### Create 2 EC2s

as we did before inside the public subnet we will create 2 ec2s with security groups

lets add the following code to `security_groups.tf`

```
resource "aws_security_group" "private_sg" {
    name = "vpc_private"
    description = "Allow incoming HTTP connections."

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]

  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP"
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

    vpc_id = aws_vpc.main.id

    tags = {
        Name = "private-sg"
    }
}

```

Then we will be able to create the 2 EC2s by adding the following code to `ec2.tf`

```
resource "aws_instance" "private" {
    count  = var.preferred_number_of_private_subnets_1 == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets_1   
    key_name      = aws_key_pair.Dare.key_name
    ami           = var.ami
    instance_type = "t2.micro"
    vpc_security_group_ids = [
        aws_security_group.private_sg.id
    ]
    subnet_id = element(aws_subnet.private.*.id,count.index)
    associate_public_ip_address = false
    source_dest_check = false
    tags = {
        Name = "private-Test${count.index}"
    }
}
```

### Create ALB 

As we did beofre in public subnet we will create new ALB but it will be private one add the following code to `alb.tf` file 

```
resource "aws_lb" "my-aws-alb-private" {
  name     = "my-test-alb-privare"
  internal = true
  security_groups = [
    aws_security_group.my-alb-sg.id,
  ]
  subnets = [
    aws_subnet.private[0].id,
    aws_subnet.private[1].id
  ]
  tags = {
    Name = "my-test-alb-private"
  }
  ip_address_type    = "ipv4"
  load_balancer_type = "application"
}
```
We used here the same security froup that we used for the public subnet one and for sure we can create new one 

We need to Create listner to this ALB and we will use same target group we used in public subnet, Add the following code to `alb.tf`

```
resource "aws_lb_listener" "my-test-alb-listner-private" {
  load_balancer_arn = aws_lb.my-aws-alb-private.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.my-target-group.arn
  }
}
```

### Create ASG 
As we did for the public one we will do the same for private one add the following code to `asg.tf`
```
resource "aws_autoscaling_group" "private_asg" {
  launch_configuration = aws_launch_configuration.my-test-launch-config.name
  vpc_zone_identifier  = [
    aws_subnet.private[0].id,
    aws_subnet.private[1].id
  ]
  target_group_arns    = [aws_lb_target_group.my-target-group.arn]
  health_check_type    = "EC2"
  min_size = 2
  max_size = 10

  tag {
    key                 = "Name"
    value               = "private_asg"
    propagate_at_launch = true
  }
}

```



## Create EFS

Inorder to create EFS you need to create KMS key.
AWS Key Management Service (KMS) makes it easy for you to create and manage cryptographic keys and control their use across a wide range of AWS services and in your applications.
Add the following code to `kms.tf`
```
resource "aws_kms_key" "kms" {
  description = "KMS key "
  policy = <<EOF
  {
  "Version": "2012-10-17",
  "Id": "kms-key-policy",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::${var.account_no}:root"},
      "Action": "kms:*",
      "Resource": "*"
    }
  ]
}
EOF
}
resource "aws_kms_alias" "alias" {
  name          = "alias/kms"
  target_key_id = aws_kms_key.kms.key_id
}
```
lets create EFS security group as every resource inside `security_groups.tf`

```
resource "aws_security_group" "SG" {
  vpc_id      = aws_vpc.main.id
  name        = "SG"
  description = "Allow Inbound Traffic"

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  # ingress {
  #   from_port   = 0
  #   to_port     = 65535
  #   protocol    = "tcp"
  #   cidr_blocks = ["${var.vpc_cidr}"]
  # }
    ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = "2049"
    to_port     = "2049"
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }
  egress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    cidr_blocks = [var.vpc_cidr]
  }
  
  egress {
    from_port = 443
    to_port = 443
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

   egress {
    from_port = 587
    to_port = 587
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }


  tags =  {
    Name = "efs-SG"
  }


}
```

Lets create EFS, add the following code to `efs.tf`

```
resource "aws_efs_file_system" "efs" {
  tags =  {
    Name = "efs"
  }
  encrypted = true
  kms_key_id = "${var.kms_arn}${aws_kms_key.kms.key_id}"
}
```

Then we need to create a mount volume. We will add two mount volumes 1 per availability zone

```
resource "aws_efs_mount_target" "mounta" {
  file_system_id  = aws_efs_file_system.efs.id
  subnet_id       = aws_subnet.private_2[0].id
  security_groups = [aws_security_group.SG.id]
}

resource "aws_efs_mount_target" "mountb" {
  file_system_id  = aws_efs_file_system.efs.id
  subnet_id       = aws_subnet.private_2[1].id
  security_groups = [aws_security_group.SG.id]
}

```
## Create RDS 

lets add RDS security Groups to `security_groups.tf`

```
resource "aws_security_group" "myapp_mysql_rds" {
  name        = "secuirty_group_web_mysqlserver"
  description = "Allow access to MySQL RDS"
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "rds_secuirty_group"
  }

}

resource "aws_security_group_rule" "security_rule" {
  security_group_id = aws_security_group.myapp_mysql_rds.id
  type              = "ingress"
  from_port         = 3306
  to_port           = 3306
  protocol          = "tcp"

  cidr_blocks = [
    var.vpc_cidr,
  ]
}
```
lets add RDS subnet groups to configure the high availabilty in out to AZ to `rds.tf` 

```
resource "aws_security_group" "myapp_mysql_rds" {
  name        = "secuirty_group_web_mysqlserver"
  description = "Allow access to MySQL RDS"
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "rds_secuirty_group"
  }

}
```
Lets Create the RDS, Add the following code to `rds.tf`

```
resource "aws_db_instance" "default" {
  allocated_storage    = 20
  storage_type         = "gp2"
  engine               = "mysql"
  engine_version       = "5.7"
  instance_class       = "db.t2.micro"
  name                 = "mydb"
  username             = "admin"
  password             = "admin1234"
  parameter_group_name = "default.mysql5.7"
  db_subnet_group_name = aws_db_subnet_group.db_subnet.name
  skip_final_snapshot  = true
  multi_az             = "true"
}
```

5.  Create an AMI from the Console 
    1. Select the EC2 Instance to create an AMI 
    2. Navigate to the AMi menu
    3. Name the AMI 
    4. Copy the AMI ID 
11. Create launch template for bastion host - https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/launch_template
    1.  Introduce userdata 
12. NLB and public facing ALB
13. Target groups 
14. Create bastion-asg.tf and nginx-tg.tf to configure the ASG respectively
15. Repeat the process for private subnets (Internal ALB, autoscaling for webservers)
16. Create EFS and RDS in the 3rd private subnet
17. Ensure security groups are configured to allow respective connectivities.
                --------Required updates to the diagram 
                1. Introduce Tooling applications - More servers within the private subnet
                1. Jenkins 
                2. Sonarqube
                3. Artifactory
18. Add more resources (HA is not considered here)
    1.  Jenkins Ec2
    2.  Sonarqube
    3.  Artifactory


