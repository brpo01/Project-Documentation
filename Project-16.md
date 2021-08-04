# Automate Infrastructure With IaC using Terraform

![tooling_project_15](https://user-images.githubusercontent.com/76074379/126079856-ac2b5dea-45d0-4f1f-85fa-54284a91a5de.png)

## Prerequisites before writing Terraform code

- On the console, create an IAM user, name it **terraform** (*ensure that the user has only programatic access to your AWS account*) and grant this user AdministratorAccess permissions.

![Inked{87B177FE-6AC9-41F2-85E5-9817FE59C4A6} png_LI](https://user-images.githubusercontent.com/76074379/126080985-a8b289ed-0539-4d6e-a486-26a76ebf69f8.jpg)

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
  
  ## Introducing Backend on S3 
Each Terraform configuration can specify a backend, which defines where and how operations are performed, where state snapshots are stored, etc.

- If you are still learning how to use Terraform, we recommend using the default `local backend`, which requires no configuration.

- If you and your team are using Terraform to manage meaningful infrastructure, you will need to create terraform backend to save your state file at the cloud 

Add the following configs to a file called `backend.tf`
1. We need to provision S3 to save our statefile there 
```
resource "aws_s3_bucket" "terraform_state" {
  bucket = "name-of-bucket"
  # Enable versioning so we can see the full revision history of our state files
  versioning {
    enabled = true
  }
  # Enable server-side encryption by default
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}
```
2. We need to provision Dynamo DB to check state locking and consistency checking
```
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "name-of-dynamodb_table"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```
3. Do `terraform apply` to provision resources 
4. Create the Backend block
```
terraform {
  backend "s3" {
    bucket         = "name-of-bucket"
    key            = "global/s3/terraform.tfstate"
    region         = "us-west-1"
    dynamodb_table = "name-of-dynamodb_table"
    encrypt        = true
  }
}
```
5. Do a `terraform init` to initialize the new backend 

    This will initialize our new backend after confirming by typing `yes` 

6. Test it now 

    Before doing anything if you opened aws now to see what heppened you shoud be able to see the following:
    - tfstatefile is now inside the S3 bucket 
    - dynamodb table which we create has an entry includes state file status 
    - run now `terraform plan` and quickly go and watch what happened to dynamodb table 
      and how it is locked 
    - wait until `terraform plan` is finished and refresh dynamo db table, you be able to see that is back to their state 


### conclusion

Terraform will automatically pull the latest state from this S3 bucket before running a command, and automatically push the latest state to the S3 bucket after running a command. To see this in action, add the following output variables:
```
output "s3_bucket_arn" {
  value       = aws_s3_bucket.terraform_state.arn
  description = "The ARN of the S3 bucket"
}
output "dynamodb_table_name" {
  value       = aws_dynamodb_table.terraform_locks.name
  description = "The name of the DynamoDB table"
}
```
and then do `terraform apply`

Now, head over to the S3 console again, refresh the page, and click the gray “Show” button next to “Versions.” You should now see several versions of your terraform.tfstate file in the S3 bucket:

With a remote backend and locking, collaboration is no longer a problem. However, there is still one more problem remaining: isolation. When you first start using Terraform, you may be tempted to define all of your infrastructure in a single Terraform file or a single set of Terraform files in one folder. The problem with this approach is that all of your Terraform state is now stored in a single file, too, and a mistake anywhere could break everything.

So the best solution for this should be state Isolation.

There are two ways you could isolate state files:
1. Isolation via workspaces: useful for quick, isolated tests on the same configuration.
2. Isolation via file layout: useful for production use-cases where you need strong separation between environments.
---

  ![{820CE5EC-E630-471E-B68D-794522E29B6D} png](https://user-images.githubusercontent.com/76074379/126080150-189345d7-8d75-45b4-95e4-6b584a1f093f.jpg)

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
Run *terraform plan* and *terraform apply -auto-approve*

![{47AA2431-3056-4DDD-BF7A-7206BDD9970C} png](https://user-images.githubusercontent.com/76074379/126080568-e111c5bd-d707-4f93-85e6-c2279d7da61b.jpg)

![{A588F2CD-3219-4C04-90AA-32AC12B61E31} png](https://user-images.githubusercontent.com/76074379/126080294-99469a77-d97d-468a-b892-d128f22e5c03.jpg)



#### Observations:

Hard coded values: Remember our best practice hint from the beginning? Both the availability_zone and cidr_block arguments are hard coded. We should always endeavour to make our work dynamic.

Multiple Resource Blocks: Notice in our code that, we have declared multiple resource blocks for each subnet. This is bad coding practice. We need to create a single resource that can dynamically create the resources without specifying multiple blocks. Imagine if we wanted to create 10 subnets. Our code will look very clumsy. So we need to optimize this by introducing the count argument.
Now let's improve this by refactoring the code.

First, destroy the current infrastructure. Since we are still in development, this is totally fine. Otherwise, DO NOT DESTROY an infrastructure that has been deployed to production.

To destroy, run ***terraform destroy*** command, and type ***yes*** after evaluating the plan. You can also run this command so you do not have to type ***yes***
```
terraform destroy -auto-approve
```


## Fixing The Problems By Code Refactoring

- Fixing Hard Coded Values: We will introduce variables, and remove hard coding.

    - Starting with the provider block, declare a variable named region, give it a default value, and update the provider section by referring to the declared variable.
    ```
    variable "region" {
        default = "us-west-1"
    }

    provider "aws" {
        region = var.region
    }
    ```
    - Do the same to cidr value in the vpc block, and all the other arguments.
    
    ```
    variable "region" {
        default = "us-west-1"
    }

    variable "vpc_cidr" {
        default = "172.16.0.0/16"
    }

    variable "enable_dns_support" {
        default = "true"
    }

    variable "enable_dns_hostnames" {
        default ="true" 
    }

    variable "enable_classiclink" {
        default = "false"
    }

    variable "enable_classiclink_dns_support" {
        default = "false"
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

    }
    ```
- Fixing multiple resource blocks: This is where things become a little tricky. It’s not complex, we are just going to introduce some interesting concepts. Loops & Data sources    

Terraform has a functionality that allows us to pull data which exposes information to us. For example, every region has Availability Zones (AZ). Different regions have from 2 to 4 Availability Zones. With over 20 geographic regions and over 70 AZs served by AWS, it is impossible to keep up with the latest information by hard coding the names of AZs. Hence, we will explore the use of Terraform’s Data Sources to fetch information outside of Terraform. In this case, from AWS.

Let us fetch Availability zones from AWS, and replace the hard coded value in the subnet’s availability_zone section.

```
# Get list of availability zones
        data "aws_availability_zones" "available" {
        state = "available"
        }
```
To make use of this new data resource, we will need to introduce a count argument in the subnet block: Something like this.

```
# Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = "172.16.1.0/24"
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }
```
Let us quickly understand what is going on here.

- The *count* tells us that we need 2 subnets. Therefore, Terraform will invoke a loop to create 2 subnets.
- The *data* resource will return a list object that contains a list of AZs. Internally, Terraform will receive the data like this

```
["us-west-1b", "us-west-1c"]
```

Each of them is an index, the first one is ***index 0***, while the other is ***index 1***. If the data returned had more than 2 records, then the index numbers would continue to increment.

Therefore, each time Terraform goes into a loop to create a subnet, it must be created in the retrieved AZ from the list. Each loop will need the index number to determine what AZ the subnet will be created. That is why we have ***data.aws_availability_zones.available.names[count.index]*** as the value for ***availability_zone***. When the first loop runs, the first index will be 0, therefore the AZ will be us-west-1b. The pattern will repeat for the second loop.

But we still have a problem. If we run Terraform with this configuration, it may succeed for the first time, but by the time it goes into the second loop, it will fail because we still have cidr_block hard coded. The same cidr_block cannot be created twice within the same VPC. So, we have a little more work to do.

### Let’s make cidr_block dynamic.
We will introduce a function ***cidrsubnet()*** to make this happen. It accepts 3 parameters. Let us use it first by updating the configuration, then we will explore its internals.

```
# Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }
```
A closer look at *cidrsubnet* – this function works like an algorithm to dynamically create a subnet CIDR per AZ. Regardless of the number of subnets created, it takes care of the cidr value per subnet.

Its parameters are *cidrsubnet(prefix, newbits, netnum)*

- The *prefix* parameter must be given in CIDR notation, same as for VPC.
- The *newbits* parameter is the number of additional bits with which to extend the prefix. For example, if given a prefix ending with /16 and a newbits value of 4, the resulting subnet address will have length /20
- The *netnum* parameter is a whole number that can be represented as a binary integer with no more than newbits binary digits, which will be used to populate the additional bits added to the prefix

You can experiment how this works by entering the **terraform console** and keep changing the figures to see the output.

On the terminal, run ***terraform console***

type *cidrsubnet("172.16.0.0/16", 4, 0)*

Hit *enter*

See the output

Keep changing the numbers and see what happens.

To get out of the console, type *exit*

### The final problem to solve is removing hard coded *count* value.
If we cannot hard code a value we want, then we will need a way to dynamically provide the value based on some input. Since the *data* resource returns all the AZs within a region, it makes sense to count the number of AZs returned and pass that number to the *count* argument.

To do this, we can introduce *length()* function, which basically determines the length of a given list, map, or string.

Since *data.aws_availability_zones.available.names* returns a list like **["us-west-1b", "us-west-1c"]** we can pass it into a *length* function and get number of the AZs.

***length(["us-west-1b", "us-west-1c"])***

Open up *terraform console* and try it

Now we can simply update the public subnet block like this
```
# Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = length(data.aws_availability_zones.available.names)
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }
```

### Observations:

- What we have now, is sufficient to create the subnet resource required. But if you observe, it is not satisfying our business requirement of just 2 subnets. The length function will return number 3 to the *count* argument, but what we actually need is 2.

Now, let us fix this.

- Declare a variable to store the desired number of public subnets, and set the default value

```
variable "preferred_number_of_public_subnets" {
  default = 2
}
```

- Next, update the *count* argument with a condition. Terraform needs to check first if there is a desired number of subnets. Otherwise, use the data returned by the *length* function. See how that is presented below.

```
# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}
```

Now lets break it down:

- The first part *var.preferred_number_of_public_subnets == null* checks if the value of the variable is set to null or has some value defined.
- The second part *?* and *length(data.aws_availability_zones.available.names)* means, if the first part is true, then use this. In other words, if preferred number of public subnets is null (Or not known) then set the value to the data returned by *length* function.
- The third part *:* and *var.preferred_number_of_public_subnets* means, if the first condition is false, i.e preferred number of public subnets is not null then set the value to whatever is defined in *var.preferred_number_of_public_subnets*

Now the entire configuration should now look like this

```
# Get list of availability zones
data "aws_availability_zones" "available" {
state = "available"
}

variable "region" {
      default = "us-west-1"
}

variable "vpc_cidr" {
    default = "172.16.0.0/16"
}

variable "enable_dns_support" {
    default = "true"
}

variable "enable_dns_hostnames" {
    default ="true" 
}

variable "enable_classiclink" {
    default = "false"
}

variable "enable_classiclink_dns_support" {
    default = "false"
}

  variable "preferred_number_of_public_subnets" {
      default = 2
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

}

# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}
```

***Note:*** You should try changing the value of *preferred_number_of_public_subnets* variable to null and notice how many subnets get created.

## Introducing variables.tf & terraform.tfvars
Instead of havng a long list of variables in *main.tf* file, we can actually make our code a lot more readable and better structured by moving out some parts of the configuration content to other files.

- We will put all variable declarations in a separate file
- And provide non-default values to each of them

    - Create a new file and name it *variables.tf*
    - Copy all the variable declarations into the new file.
    - Create another file, name it *terraform.tfvars*
    - Set values for each of the variables.
 
 **main.tf**
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

}

# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
}
```

**variables.tf**
```
variable "region" {
      default = "us-west-1"
}

variable "vpc_cidr" {
    default = "172.16.0.0/16"
}

variable "enable_dns_support" {
    default = "true"
}

variable "enable_dns_hostnames" {
    default ="true" 
}

variable "enable_classiclink" {
    default = "false"
}

variable "enable_classiclink_dns_support" {
    default = "false"
}

  variable "preferred_number_of_public_subnets" {
      default = null
}
```

**terraform.tfvars**
```
region = "us-west-1"

vpc_cidr = "172.16.0.0/16" 

enable_dns_support = "true" 

enable_dns_hostnames = "true"  

enable_classiclink = "false" 

enable_classiclink_dns_support = "false" 

preferred_number_of_public_subnets = 2
```

You should also have this file structure in the PBL folder.

```
└── PBL
    ├── main.tf
    ├── terraform.tfstate
    ├── terraform.tfstate.backup
    ├── terraform.tfvars
    └── variables.tf
```

Run ***terraform plan*** and ***terraform apply -auto-approve*** and ensure everything works.

![{60B91E88-B7FA-4933-BF89-89F584DF9B2D} png](https://user-images.githubusercontent.com/76074379/126080909-9e3d6395-f4e1-41e1-8117-d3c49a36b29f.jpg)

**Note**: Create a `.gitignore` file and add files such as `variables.tf`, `terraform.tfvars`, `.terraform.tfstate` etc., that contain sensitive information so  that it will not be tracked and exposed to the public on your version control software. e.g Github

## Credits

https://darey.io/docs/project-16-introduction/

https://docs.chocolatey.org/en-us/choco/setup

https://github.com/aws/aws-cli/issues/3567



