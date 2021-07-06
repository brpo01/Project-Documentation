# AWS Cloud Solution For 2 Company Websites Using Nginx as a Reverse Proxy
## Pretasks:
- Configure AWS account and Organization Unit
  - Create an AWS account (ignore if you already have one)
  - Create an Organization Unit 
  - Click 'Add an AWS account' and create a new AWS account from there with name as Dev
  - Invite another AWS account( You can open a totally new AWS account for this)
  - Login and accept the invitation in the new AWS account
    ![Inked{285E6809-6CE8-44C2-A935-1BC4DD4D2ADE} png_LI](https://user-images.githubusercontent.com/76074379/123252997-c7181700-d4a1-11eb-9534-a58fc329b71d.jpg)
    
- Create a free domain name from http://www.freenom.com
- Create a hosted zone in AWS Route 53
  - Go to the Route 53 Console
  - Click 'Create Hosted Zone'
  - For Domain name, enter the domain name you got from freenom
  - Enter a description (if you want)
  - For type, select Public Hosted Zone
  - Click Create Hosted Zone
  - Click on the created hosted zone and copy the contents of the NS record
  - Click 'Manage Domain' next to your domain name, and click Management Tools and select Nameservers
  - Click 'Use custom nameservers' radio button
  - Replace the content there with the items you got from Route 53 (one per line)
   ![{06426222-3F1B-43F4-9327-AAB4D3C8D49D} png](https://user-images.githubusercontent.com/76074379/123254120-1f9be400-d4a3-11eb-8d7a-f30383ab3865.jpg)
   
   ![{34F81419-3828-43E0-83FC-09DCE1246566} png](https://user-images.githubusercontent.com/76074379/123254328-625dbc00-d4a3-11eb-97fe-1e4d9868c831.jpg)
   
- Ensure to tag all resources you create (Project, Environment, Name etc)

## Step 1: TLS Certificates from Amazon Certificate Manager (ACM)

- Navigate to AWS ACM
- Under 'Provision certificates' click Get started
- Click Request a certificate
- Enter the domain name you registered (<*domain-name>.com*).Also enter additional domain names:
   - tooling.<*domain-name>.com*
   - www.<*domain-name>.com*
- Click Next
- Select DNS validation and click Next
- Tag the certificate, click Review then confirm and request
- Click Continue
- Click 'Export DNS Configuration file'
- Go to Route 53
- Create a new CNAME record with items from the DNS configuration.csv file downloaded.

## Step 2: Setup a Virtual Private Cloud

![tooling_project_15](https://user-images.githubusercontent.com/76074379/123254593-b4064680-d4a3-11eb-8099-329e9fb7c060.png)

- Create a VPC from the VPC Management Console use a large enough CIDR block (/16)
![{AD89A4F9-51A4-4D26-BCF4-AC862B5378A7} png](https://user-images.githubusercontent.com/76074379/123254815-f92a7880-d4a3-11eb-9625-330625b47592.jpg)

- Create subnets as shown in the diagram above
  
  ![{6901C4E0-248E-4316-A52A-7288B2FEFA86} png](https://user-images.githubusercontent.com/76074379/123254959-26772680-d4a4-11eb-95bc-79320dd90009.jpg)
  
  - For the public subnet, enable auto-assign IP by selecting the subnet (after you've created it) and clicking Actions button on the top right, then select Modify auto-assign IP settings and enable it
  ![{374AB1F3-5FA4-4B51-95F4-A848FB2B530E} png](https://user-images.githubusercontent.com/76074379/123255086-49093f80-d4a4-11eb-83b9-dbfa4f9b33fa.jpg)
  
- Create a route table and associate it with the public subnets
  - Select the route table you created, click Actions on the top and click 'Edit Subnet associations'
  - Select the public subnets and click save
![{0B0EE674-F32D-4295-815E-845FA90B282A} png](https://user-images.githubusercontent.com/76074379/123257860-8de2a580-d4a7-11eb-942b-d1dc29079b81.jpg)

- Create a route table for the private subnets
  - Repeate the steps above
  ![{B4B3AD01-F825-44D0-81C4-0B3F94F37803} png](https://user-images.githubusercontent.com/76074379/123258128-df8b3000-d4a7-11eb-9f32-fa2f012866c2.jpg)
  
- Create an Internet Gateway, select it and click Actions the click Attach to VPC and attach it to the VPC you created
![{8CC4661F-BB81-4268-8AB0-1D11B8298D63} png](https://user-images.githubusercontent.com/76074379/123256553-f2046a00-d4a5-11eb-8d90-70d13f9a3c9e.jpg)

- Add a new route to your public subnet route table
  - Select the route table, click Actions and 'Edit routes'
  - For destination, enter 0.0.0.0/0
  - For target, select Internet Gateway and click the Internet Gateway you created
  - Click Save
  ![{F1C4C675-ABA8-4CB8-A892-E42F79E51C44} png](https://user-images.githubusercontent.com/76074379/123258374-25e08f00-d4a8-11eb-8259-01cbc9c060e1.jpg)
  
- Create a NAT Gateway for your private subnets
- Allocate three Elastic IPs and associate one of them to the NAT Gateway(the other two are for the Bastion Servers)
- Add a new route to your private route table with destination as 0.0.0.0/0 and target as the NAT Gateway you created
![{CEA496E8-669A-4F44-8EB0-5D38E67156F6} png](https://user-images.githubusercontent.com/76074379/123258549-59bbb480-d4a8-11eb-9279-32786ead4b46.jpg)

- Create a security group for:
  - Nginx servers: Access to nginx servers should only be from the Application Load Balancer
  - Bastion servers: Access to bastion servers should only be from the IPs of your workstation
  - Application Load Balancer: ALB should be open to the internet
  - Webservers: Webservers should only be accessible from the Nginx servers
  - Data Layer: This comprises the RDS and EFS servers. Access to RDS should only be from Webservers, while Nginx and Webservers can have access to EFS
 
 ![{B4BB3DEE-B49F-4D1B-A1D6-C197EBA1E403} png](https://user-images.githubusercontent.com/76074379/124334344-a7d95380-db4b-11eb-8b62-fa22e30ca035.jpg)
![{30A6CA71-1FE8-49FC-B53E-41A5F08C9CE4} png](https://user-images.githubusercontent.com/76074379/124334346-a90a8080-db4b-11eb-9b05-3c5a2e75668f.jpg)
![{DC07A949-E80A-4A0F-99E3-62844788312B} png](https://user-images.githubusercontent.com/76074379/124334350-a9a31700-db4b-11eb-87c4-3b27d1a8c2fb.jpg)
![{3F0F6E04-9341-41B6-8689-5A0C9B12E2F5} png](https://user-images.githubusercontent.com/76074379/124334352-a9a31700-db4b-11eb-81af-6b1568130fa0.jpg)
![{844AC8F2-4655-4DA1-A255-D659BCE1F23C} png](https://user-images.githubusercontent.com/76074379/124334377-bd4e7d80-db4b-11eb-920f-c4f71bfa1a8e.jpg)

## Step 3: Proceed with Compute Resources

### Step 3.1: Setup Compute Resources for Nginx

- Provision EC2 Instances for Nginx
  - Create a t2.micro RHEL 8 instance in any of your two public AZs
  - Install the following packages
    ```
    epel-release
    python
    htop
    ntp
    net-tools
    vim
    wget
    telnet
    ```
  - Create an AMI from the instance
    - Right click on the instance
    - Select Image and click Create Image
    - Give the AMI a name
- Prepare Launch Template for Nginx
  - From EC2 Console, click Launch Templates from the left pane
  - Choose the Nginx AMI
  - Select the instance type (t2.micro)
  - Select the key pair
  - Select the security group
  - Add resource tags
  - Click Advanced details, scroll down to the end and configure the user data script to update the yum repo and install nginx
    ```
    #!/bin/bash
    yum update -y
    mkdir -p /var/www/html
    echo "hello world from $(hostname)" > /var/www/html/healthstatus
    yum install nginx -y
    sed -i 's/\/usr\/share\/nginx\/html/\/var\/www\/html/' /etc/nginx/nginx.conf
    systemctl enable nginx
    setenforce 0
    systemctl restart nginx 
    yum install -y git
    ```
- Configure Target Group (for both Port 80)
  
  - Select instances as target type 
  - Enter the target group name
  - Select the VPC you created
  - For health checks, select HTTP(port 80) and health check path as /healthstatus
  - Add Tags
  - Register Nginx instances as targets
 Note: In order to avoid confusion to a newbie. I will opt to ceate an Application Load Balancer and configure rather than try to create an HTTPS Target Group straight away. Reason for this will be clear as one develop familiarity with the project.
 
- Configure Application Load Balancer (ALB) for Nginx
  
Nginx instances should only accept connections coming from the ALB and deny any connections directly to it.

- Create an internet facing ALB
  - From the EC2 Console, click Load Balancers. 
  - On the block for Application Load Balancers, click create
  - Enter the name for the load balancer
  - Since it's for the Nginx servers, add a HTTPS Listener
  - Select the VPC you created, check the two AZs and add the public subnets you have. Click next.
  - Select the certificate you created on ACM
  - On the next page, select the ALB security group
  - Configure routing, select the Nginx target group for port 443
  - Register Nginx instances as targets (if you were not able to configure target group for HTTPS(port 443) in the above step)
  - for health checks, select HTTPS(port 443) and health check path as /healthstatus
  - Click Review and complete the process
  
  You may add HTTP Listener and the corresponding Target Group

- Configure Autoscaling for Nginx
  
  - Enter the name
  - Select the Nginx launch template, click Next
  - For Load Balancer, Click Existing Load Balancer and add the right Target Groups( for both port 80 and 443)
  - Select the VPC and select the two public subnets you created, click Next
  - For health checks, select ELB too. Click Next.
  - For Group size, enter 2 for minimum and desired capacity, 4 as maximum capacity
  - For Scaling policies, select Target Tracking scaling policy and set the target value as 90
  - Click Next and add Notifications, create a new SNS topic and enter your email under 'With these recipients'
  - Add Tags
  
### Step 3.2: Setup Compute Resources for Bastion

- Provision EC2 Instances for Bastion server
  - Create a t2.micro RHEL 8 instance in any of your two public AZs where you created Nginx instances
  - Install the following packages
    ```
    epel-release
    python
    htop
    ntp
    net-tools
    vim
    wget
    telnet
    ```
  - Attach an Elastic IP to each of the servers
  - Create an AMI from the instance
    - Right click on the instance
    - Select Image and click Create Image
    - Give the AMI a name

- Prepare Launch Template for Nginx
  - From EC2 Console, click Launch Templates from the left pane
  - Choose the Bastion AMI
  - Select the instance type (t2.micro)
  - Select the key pair
  - Select the security group
  - Add resource tags
  - Click Advanced details, scroll down to the end and configure the user data script to update the yum repo and install nginx
    ```
    #!/bin/bash
    yum update -y
    mkdir -p /var/www/html
    echo "hello world from $(hostname)" > /var/www/html/healthstatus
    yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
    yum install -y python3
    python3 -m pip install ansible
    yum install -y git
    ```
- Configure Target Groups
  - Select instances as target type 
  - Enter the target group name
  - Select the VPC you created
  - For health checks, select HTTPS and health check path as /healthstatus
  - Add Tags
  - Register Bastion instances as targets

- Configure Autoscaling for Nginx
  
  - Enter the name
  - Select the Bastion launch template, click Next
  - Select the VPC and select the two public subnets you created, click Next
  - For health checks, select ELB too. Click Next.
  - For Group size, enter 2 for minimum and desired capacity, 4 as maximum capacity
  - For Scaling policies, select Target Tracking scaling policy and set the target value as 90
  - Click Next and add Notifications, create a new SNS topic and enter your email under 'With these recipients'
  - Add Tags



### Step 3.3: Setup Compute Resources for Webservers

We have to create two launch templates for Wordpress and Tooling respectively.

- Provision two EC2 Instances one for Tooling and another for Wordpress
  - Create a t2.micro RHEL 8 instance in any of your two public AZs where you created Nginx instances
  - Install the following packages
    ```
    epel-release
    python
    htop
    ntp
    net-tools
    vim
    wget
    telnet
    ```
  - Attach an Elastic IP to each of the servers
  - Create an AMI from the instance
    - Right click on the instance
    - Select Image and click Create Image
    - Give the AMI a name

- Prepare Launch Template for Webservers
  - From EC2 Console, click Launch Templates from the left pane
  - Choose the Wordpress AMI
  - Select the instance type (t2.micro)
  - Select the key pair
  - Select the security group
  - Add resource tags
  - Click Advanced details, scroll down to the end and configure the user data script to update yum package, install apache,create health status file,install wordpress, install     efs(which will be specified later) an other necessary programs, modules and dependencies
    ```
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    touch /var/www/html/healthstatus
    echo "wp from $(hostname)" >> /var/www/html/healthstatus
    yum install -y git
    yum install -y mysql
    yum install -y php php-{mysqlnd,cli,gd,common,mbstring,fpm,json}
    wget http://wordpress.org/latest.tar.gz
    tar xzvf latest.tar.gz
    yum install -y rpm-build
    git clone https://github.com/aws/efs-utils
    cd /efs-utils
    yum -y install make
    make rpm
    yum -y install ./build/amazon-efs-utils*rpm
    setenforce 0
    systemctl restart httpd
    ```
    Repeat the above steps for Tooling Webserver. Check user data for Tooling Webserver below:
    ```
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    touch /var/www/html/healthstatus
    echo "tooling from $(hostname)" >> /var/www/html/healthstatus
    yum install -y git
    yum install -y mysql
    yum install -y php php-{mysqlnd,cli,gd,common,mbstring,fpm,json}
    yum install -y rpm-build
    git clone https://github.com/aws/efs-utils
    cd /efs-utils
    yum -y install make
    make rpm
    yum install -y ./build/amazon-efs-utils*rpm
    setenforce 0
    systemctl restart httpd
    ```
    
- Configure Target Groups
  
  - Select instances as target type 
  - Enter the target group name
  - Select the VPC you created
  - For health checks, select HTTP and health check path as /healthstatus
  - Add Tags
  - Register Tooling instance as target

Repeat above steps for Wordpress
  
- Configure ALB for Webservers

   The ALB for the webservers should not be internet facing. And we'll need two ALBs, one for Tooling and Wordpress

- Create an internal ALB
  
  - From the EC2 Console, click Load Balancers. 
  - On the block for Application Load Balancers, click create
  - Enter the name for the load balancer
  - Select the VPC you created, check the two AZs and add the private subnets you have. Click next.
  - On the next page, select the webserver security group
  - Configure routing, select the Tooling target group
  - Register targets (unnecessary if you configured your target group correctly)
  - Click Review and complete the process
  
    Repeat the above steps for the Wordpress ALB

- Configure Autoscaling for Webservers(Tooling and Wordpress)

  For Tooling
  
  - Enter the name
  - Select the appropriate launch template, click Next
  - Select the VPC and select the two public subnets you created, click Next
  - For health checks, select ELB too. Click Next.
  - For Group size, enter 2 for minimum and desired capacity, 4 as maximum capacity
  - For Scaling policies, select Target Tracking scaling policy and set the target value as 90
  - Click Next and add Notifications, create a new SNS topic and enter your email under 'With these recipients'
  - Add Tags
    Repeat the above steps for Wordpress

## Step 4: Setup EFS
- Navigate to EFS from your Management Console
- Click create file system from the right
- Click Customize
- Enter the name for the EFS
- Tag the resource
- Leave everything else and click next
- Select the VPC you created, select the two AZs and choose the private subnets
- Select the EFS security group for each AZ
- Click next, next then create
![{2BB96602-1248-44A3-89EE-D9B1FCB9A133} png](https://user-images.githubusercontent.com/76074379/124367062-03b5e200-dc09-11eb-916f-9978210ffe11.jpg)

- Create an EFS access point. (Give it a name and leave all other settings as default)
![{523EE8BA-BB52-44BE-9025-9998569A9095} png](https://user-images.githubusercontent.com/76074379/124367141-8d65af80-dc09-11eb-9dd4-b1a59efe8eec.jpg)

## Step 5: Setup RDS
### Step 5.1: Create a KMS key
- Navigate to AWS KMS
- Click create key
- Make sure it's symmetric
- Give the key an alias
- For 'Define Key admininstrative privileges', select AWSServiceRoleForRDS and rdsMonitoringRole

![{AB88012C-7F94-4616-89DE-BDB4C40B2F74} png](https://user-images.githubusercontent.com/76074379/124367186-e6cdde80-dc09-11eb-9ed9-c6e4619c4899.jpg)

- Select the same thing for Key usage
- Click Finish

### Step 5.2: Create a DB Subnet Group
- Navigate to RDS Management Console
- Click the three horizontal lines on the top left
- Select Subnet groups
- Click Create DB subnet group
- Enter the name, description and select your VPC
- Under Add subnets, select the two AZs your data layer subnets are in and select the two private data layer subnets.
- Click Create

![{6EC64FA0-C2DE-4151-B20A-85C17AD0D0EC} png](https://user-images.githubusercontent.com/76074379/124367227-2ac0e380-dc0a-11eb-9369-db42e2452b4d.jpg)

### Step 5.3: Create RDS Instance
- Navigate to RDS Management Console
- Click Create database
- For Engine options, select MySQL
- For Template, choose Dev/Test
- Enter a name for your DB under DB instance identifier
- Enter Master username and passsword
- Choose the smallest possible instance class (to reduce costs)
- Under Availability, select do not create a standby instance
- Select your VPC, select the subnet group you created and also the data layer security group
- Leave everything else and scroll down to Additional configuration
- Enter initial database name (but i'll personally recommend you connect to it from your webservers and create required databases)
- Leave everything else, scroll down to Encryption and select the KMS key you created
- Scroll down and click Create database

![{1D2C2675-1F4A-4E2C-8FF9-ED89510F0C23} png](https://user-images.githubusercontent.com/76074379/124367257-72476f80-dc0a-11eb-9ce7-c0625f28019a.jpg)

## Step 6: Configure DNS with Route 53
- Create a CNAME record that points www.domain.com to the DNS name of your NGINX load balancer
- Create a CNAME record that points tooling.domain.com to the DNS name of your NGINX load balancer

### Step 6.2
- Create two configuration files (one for tooling, one for wordpress) for the nginx load balancer and add to a github repo so you can pull the config during a scale out
- The tooling config file should contain the following settings: 
  ```
  server {
    server_name tooling.domain.com;
    location ~ { # case-sensitive regular expression match
		include /etc/nginx/mime.types;
	    proxy_redirect      off;
	    proxy_set_header    X-Real-IP $remote_addr;
	    proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header    Host $http_host;
		proxy_pass http://tooling-ALB-DNS;
	  }
  }
  ```
  The wordpress config file:
  ```
  server {
    server_name domain.com www.domain.com; 
    location ~ { # case-sensitive regular expression match
		include /etc/nginx/mime.types;
	    proxy_redirect      off;
	    proxy_set_header    X-Real-IP $remote_addr;
	    proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header    Host $http_host;
		proxy_pass http://wordpress-ALB-DNS;
	  }
  }
  ```
![{9DA7708A-4274-402A-A491-E96AFEDE7E74} png](https://user-images.githubusercontent.com/76074379/124367321-09142c00-dc0b-11eb-8c41-9d7122a459a2.jpg)

![{DC558EFC-2080-480A-AD2D-F60E0DDAA084} png](https://user-images.githubusercontent.com/76074379/124367332-1d582900-dc0b-11eb-8d9a-0cdad5d4491a.jpg)

![{E0B9ED33-9FC3-4DED-8AEB-6C04FB4E3C92} png](https://user-images.githubusercontent.com/76074379/124367362-52647b80-dc0b-11eb-818f-b52dce6bfa1a.jpg)

![{BE97DCE5-9775-4C05-9721-E313A6CB7433} png](https://user-images.githubusercontent.com/76074379/124367368-57292f80-dc0b-11eb-8748-d962b0a489b5.jpg)

# Blockers

- I had to install mod_ssl module manually on the apache webservers after because the error logs indicated it was missing. After installing the module on my webservers, my target webserver instances on passed the healthcheck on port 443(HTTPS). For more on mod_ssl, https://en.wikipedia.org/wiki/Mod_ssl. I ran the following commands:
```
sudo dnf install -y mod_ssl

apachectl -M | grep ssl

sudo systemctl restart httpd
```

- I was getting a 502 Bad Gateway error, to solve this, I checked the logs for my nginx instance (I had to SSH into it) and noticed it was returning a permission denied error, so I disabled SELinux in by opening /etc/sysconfig/selinux and setting the SELinux to disabled. I also disabled SElinux on the web servers.

# Credits

darey.io
