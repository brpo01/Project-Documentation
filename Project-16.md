
# Automate Infrastructure With IaC using Terraform
## Prerequisites before writing Terraform code

- On the console, create an IAM user, name it **terraform** (*ensure that the user has only programatic access to your AWS account*) and grant this user AdministratorAccess permissions.
- Copy the secret access key and access key ID. Save them in a notepad temporarily.
- Configure programmatic access from your workstation to connect to AWS using the access keys copied above and a Python SDK (boto3). You must have Python 3.6 or higher on your       workstation.

    - **For Windows** (you may use Powershell or cmd. But I'll preferably use Gitbash)
       - Run Powershell, cmd or Gitbash as Administrator
       - Install aws CLI. Click [here](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html)
       - Install and configure Python SDK. Click [here](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/quickstart.html#installation)
       - For easier authentication configuration - use AWS CLI with *aws configure* command. For guidance or help,run command
         ```
         aws configure help
         ```
       - Install chocolatey. Click [here](https://docs.chocolatey.org/en-us/choco/setup)
       - Install Terraform.
         ```
         choco install terraform
         ```
- Create an s3 bucket using the AWS CLI. Follow Bucket naming rules. For more info, click [here](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html)
  ```
  aws s3 mb s3://<bucket-name>
  ```

## The VPC | Subnets | Security Groups
Let us create a directory structure.
- Create a folder PBL
- Create a file in the folder, name it *main.tf*

### Provider and VPC resource section
- Add the AWS provider, and a resource to create the VPC in the *main.tf* file.
- The provider block informs terraform that we intend to build infrastructure against AWS.
- The resource block will create a VPC.

```
provider "aws" {
  region = "us-west-1"
}

# Create VPC
  resource "aws_vpc" "main" {
  cidr_block                     = "172.16.0.0/16"
  enable_dns_support             = "true"
  enable_dns_hostnames           = "true"
  enable_classiclink             = "false"
  enable_classiclink_dns_support = "false"
}
```

The next thing we need to do, is to download the necessary plugins for terraform to work. These plugins are used by providers and provisioners. At this stage, we only have provider in our *main.tf* file. So terraform will just download plugin for the AWS provider.
Lets accomplish this with ***terraform init*** command

#### Observations:

Notice that a new directory is being created: .terraform. This is where terraform keeps data about plugins. Generally, it is safe to delete this folder. It just means that you must execute ***terraform init*** again, to download them.
Moving on, let us create the only resource we just defined. aws_vpc. But before we do that, we should check to see what terraform intends to create before we tell it to go ahead and create it.

Run command
```
terraform plan
```
Then, If you are happy with the output, then create VPC by executing the below command:
```
terraform apply -auto-approve
```

#### Observations:

A new file is created ***terraform.tfstate***. This is how terraform keeps itself up to date with the exact state of the infrastructure. It reads this file to know what already exists, what should be added, or destroyed based on the entire terraform code that is being developed.
If you also observed closely, you will realise that another file gets created during planning and apply. But this file gets deleted immediately. terraform.tfstate.lock.info This is what terraform uses to track, who is running terraform code against the infrastructure at any point in time. Its content is usually like this.
It is a json format that stores the ID of the user, what operation he/she is doing, timestamp, and location of the state file.

## Subnets resource section
We need 6 subnets
- 2 public(internet-facing) subnet for nginx servers
- 2 private for web servers
- 2 private for the data layer

Let us create the first 2 public facing subnets. Add this to the ***main.tf*** file.

```
# Create public subnets1
    resource "aws_subnet" "public1" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.0.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "us-west-1b"

}

# Create public subnet2
    resource "aws_subnet" "public2" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.1.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "us-west-1c"

}
```
We are creating 2 subnets, therefore declaring 2 resource blocks for each of the subnets.
We are using the vpc_id argument to interpolate the value of the VPC id by setting it to aws_vpc.main.id. That way, terraform knows which VPC to create the subnet.
Run terraform plan and terraform apply

#### Observations:

Hard coded values: Remember our best practice hint from the beginning? Both the availability_zone and cidr_block arguments are hard coded. We should always endeavour to make our work dynamic.

Multiple Resource Blocks: Notice in our code that, we have declared multiple resource blocks for each subnet. This is bad coding practice. We need to create a single resource that can dynamically create the resources without specifying multiple blocks. Imagine if we wanted to create 10 subnets. Our code will look very clumsy. So we need to optimize this by introducing the count argument.
Now let's improve this by refactoring the code.

First, destroy the current infrastructure. Since we are still in development, this is totally fine. Otherwise, DO NOT DESTROY an infrastructure that has been deployed to production.

To destroy, run ***terraform destroy*** command, and type ***yes*** after evaluating the plan. You can also run this command so you do not have to type ***yes***
```
terraform destroy -auto-approve
```


