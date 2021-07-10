
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
