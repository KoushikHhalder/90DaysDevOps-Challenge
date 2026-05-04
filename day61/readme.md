📅 Day 61 – Introduction to Terraform and Your First AWS Infrastructure

1. SIMPLE LEARNING PLAN
You've been working inside infrastructure — containers, pods, clusters. Today you learn how to create the infrastructure itself.
Think about everything Kubernetes runs on: servers, networks, storage buckets, security groups. In most companies, someone used to click through the AWS console to create all of that by hand. That's slow, error-prone, and impossible to reproduce exactly. Infrastructure as Code (IaC) means you describe your infrastructure in text files and let a tool build it — the same way every time, reviewable in Git, deletable with one command.
Terraform is the most widely used IaC tool in the industry. You write .tf files describing what you want — an EC2 server, an S3 bucket, a VPC — and Terraform figures out how to create, update, or delete them.
The core workflow you'll use every single day with Terraform:
Write .tf file → terraform init → terraform plan → terraform apply → terraform destroy
CommandWhat it doesterraform initDownloads the cloud provider plugin (AWS, GCP, Azure)terraform planShows exactly what will change — nothing is created yetterraform applyCreates/updates real infrastructureterraform destroyTears down everything Terraform manages
By end of today you'll have created a real S3 bucket and EC2 instance on AWS using nothing but a terminal — and destroyed them just as cleanly.

⚠️ Cost note: A t2.micro EC2 instance costs ~$0.012/hour. As long as you run terraform destroy at the end of today, your total bill will be a few cents or zero (covered by Free Tier if your account is under 12 months old).


2. STEP-BY-STEP PRACTICAL TASKS

🔧 PRE-CHECK
bash# Verify WSL and internet access
curl -s https://www.google.com > /dev/null && echo "Internet OK"
lsb_release -a
📁 Set up your working directory
bashmkdir -p ~/github-actions-practice/2026/day-61
cd ~/github-actions-practice/2026/day-61

✅ TASK 1 — Understand Infrastructure as Code (Write in Your Own Words)
Before touching any commands, spend 10 minutes writing short answers to these in your README. Here are the concepts explained simply — rewrite these in your own words, don't copy:
What is IaC and why does it matter?
Instead of clicking through the AWS console to create servers and networks, you write code files that describe the infrastructure. The tool reads your files and builds everything automatically. This means your infrastructure is version-controlled, repeatable, and shareable — just like application code.
What problems does IaC solve vs manual console clicks?
Manual console work can't be reviewed, can't be reproduced exactly, and can't be rolled back. If a colleague clicks the wrong settings, you won't know. With IaC, every change is a code diff — reviewable, testable, and reversible.
How is Terraform different from CloudFormation / Ansible / Pulumi?
ToolWhat it doesCloud supportTerraformDeclares infrastructure stateAny cloud (AWS, GCP, Azure, k8s)CloudFormationSame idea but AWS-onlyAWS onlyAnsibleConfigures servers that already existAny, but mainly serversPulumiIaC but using real programming languages (Python, TypeScript)Any cloud
What does "declarative" and "cloud-agnostic" mean?
Declarative = you describe the end state ("I want a t2.micro EC2 instance"), not the steps to get there. Terraform figures out the steps.
Cloud-agnostic = the same Terraform workflow works on AWS, GCP, Azure, and even Kubernetes — just swap the provider.
✅ Action: Write 3-4 sentences of your own on each point above in a notes file. You'll use these in your README.

✅ TASK 2 — Install Terraform and Configure AWS
Step 1: Install Terraform on WSL (Ubuntu)
bash# Install HashiCorp GPG key
wget -O - https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Add HashiCorp apt repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

# Install
sudo apt update && sudo apt install terraform -y
Step 2: Verify Terraform
bashterraform -version
Expected:
Terraform v1.10.x
on linux_amd64
Step 3: Install AWS CLI (if not already installed)
bash# Check if already installed
aws --version

# If not installed:
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
Step 4: Configure AWS credentials
bashaws configure
You'll be prompted for:
AWS Access Key ID [None]: <your-access-key-id>
AWS Secret Access Key [None]: <your-secret-access-key>
Default region name [None]: ap-south-1
Default output format [None]: json

💡 Where to get AWS credentials:

Log into AWS Console → top-right menu → Security credentials
Create Access Key → Application running outside AWS
Copy the Access Key ID and Secret Access Key
⚠️ Never commit these to Git. They give full account access.


Step 5: Verify AWS access
bashaws sts get-caller-identity
Expected output:
json{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
✅ Verify: Your AWS Account ID appears. This confirms Terraform will be able to create resources.

✅ TASK 3 — Your First Terraform Config: Create an S3 Bucket
Step 1: Create the project directory
bashmkdir -p ~/github-actions-practice/2026/day-61/terraform-basics
cd ~/github-actions-practice/2026/day-61/terraform-basics
Step 2: Create the .gitignore FIRST (before any state files are created)
bashcat > .gitignore <<EOF
# Terraform state files - NEVER commit these
*.tfstate
*.tfstate.backup
*.tfstate.lock.info

# Terraform working directory
.terraform/
.terraform.lock.hcl

# Variable files that may contain secrets
*.tfvars
*.tfvars.json

# Crash log files
crash.log
crash.*.log
EOF
Step 3: Write your first main.tf
bashcat > main.tf <<EOF
# Tell Terraform which provider to use and what version
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.5"
}

# Configure the AWS provider - use your region
provider "aws" {
  region = "ap-south-1"
}

# Create an S3 bucket - name must be globally unique
resource "aws_s3_bucket" "my_bucket" {
  bucket = "terraweek-kousik-2026"

  tags = {
    Name        = "TerraWeek Bucket"
    Environment = "Dev"
    Project     = "90DaysDevOps"
  }
}
EOF

💡 Change terraweek-kousik-2026 to something unique. S3 bucket names are global across ALL AWS accounts worldwide — if the name is taken, terraform apply will fail. Use your name + a number.

Step 4: Run terraform init
bashterraform init
Expected output:
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...
- Installed hashicorp/aws v5.x.x (signed by HashiCorp)

Terraform has been successfully initialized!

💡 What did terraform init do?

Created a .terraform/ directory
Downloaded the AWS provider plugin (~50MB) into .terraform/providers/
Created .terraform.lock.hcl — a lockfile pinning the exact provider version
This is like npm install or pip install — it pulls dependencies before you can run anything.


Step 5: Run terraform plan
bashterraform plan
Expected output:
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_s3_bucket.my_bucket will be created
  + resource "aws_s3_bucket" "my_bucket" {
      + arn           = (known after apply)
      + bucket        = "terraweek-kousik-2026"
      + id            = (known after apply)
      + tags          = {
          + "Environment" = "Dev"
          + "Name"        = "TerraWeek Bucket"
          + "Project"     = "90DaysDevOps"
        }
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.

💡 + means CREATE. Nothing has been created yet — plan only shows what will happen. This is your review step before spending money.

Step 6: Run terraform apply
bashterraform apply
Type yes when prompted.
Expected output:
aws_s3_bucket.my_bucket: Creating...
aws_s3_bucket.my_bucket: Creation complete after 3s [id=terraweek-kousik-2026]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
Step 7: Verify in AWS console
bash# Verify via CLI
aws s3 ls | grep terraweek
Expected:
2026-04-05 10:00:00 terraweek-kousik-2026
✅ Verify: S3 bucket exists in the AWS console and via aws s3 ls.

✅ TASK 4 — Add an EC2 Instance
Step 1: Find the correct AMI for your region
bash# Get Amazon Linux 2 AMI ID for ap-south-1
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
             "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text \
  --region ap-south-1
Expected output:
ami-0f5ee92e2d63afc18
Note this AMI ID — you'll use it in the next step.
Step 2: Add EC2 instance to main.tf
bashcat >> main.tf <<EOF

# Create an EC2 instance
resource "aws_instance" "my_ec2" {
  ami           = "ami-0f5ee92e2d63afc18"   # Amazon Linux 2 - ap-south-1
  instance_type = "t2.micro"

  tags = {
    Name        = "TerraWeek-Day1"
    Environment = "Dev"
    Project     = "90DaysDevOps"
  }
}
EOF
Step 3: Format the file
bashterraform fmt

💡 terraform fmt auto-fixes indentation and spacing in your .tf files. Run it before every commit — it's like prettier for Terraform.

Step 4: Validate syntax
bashterraform validate
Expected:
Success! The configuration is valid.
Step 5: Plan — notice Terraform only shows the EC2 instance as new
bashterraform plan
Expected output:
aws_s3_bucket.my_bucket: Refreshing state... [id=terraweek-kousik-2026]

Terraform used the selected providers to generate the following execution plan.

  # aws_instance.my_ec2 will be created
  + resource "aws_instance" "my_ec2" {
      + ami           = "ami-0f5ee92e2d63afc18"
      + instance_type = "t2.micro"
      + tags          = {
          + "Name" = "TerraWeek-Day1"
        }
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.

💡 How does Terraform know S3 already exists? It checks terraform.tfstate — the state file records every resource Terraform has created. When you run plan, Terraform compares your .tf files against the state file and only shows the difference. The S3 bucket is already in state → no change needed.

Step 6: Apply
bashterraform apply
Type yes.
Expected:
aws_s3_bucket.my_bucket: Refreshing state...
aws_instance.my_ec2: Creating...
aws_instance.my_ec2: Still creating... [10s elapsed]
aws_instance.my_ec2: Creation complete after 32s [id=i-0xxxxxxxxxxxxxxxxx]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
Step 7: Verify the EC2 instance
bashaws ec2 describe-instances \
  --filters "Name=tag:Name,Values=TerraWeek-Day1" \
  --query "Reservations[*].Instances[*].[InstanceId,State.Name,Tags[?Key=='Name'].Value|[0]]" \
  --output table \
  --region ap-south-1
Expected:
----------------------------------------------
|         DescribeInstances                  |
+----------------------+---------+-----------+
|  i-0xxxxxxxxxxxxxxxxx|  running| TerraWeek-Day1|
+----------------------+---------+-----------+
✅ Verify: EC2 instance is running with tag TerraWeek-Day1 in AWS console and CLI.

✅ TASK 5 — Understand the State File
Step 1: Look at the state file
bashcat terraform.tfstate
You'll see a large JSON file. Key sections:
json{
  "version": 4,
  "terraform_version": "1.10.x",
  "resources": [
    {
      "type": "aws_s3_bucket",
      "name": "my_bucket",
      "instances": [
        {
          "attributes": {
            "id": "terraweek-kousik-2026",
            "arn": "arn:aws:s3:::terraweek-kousik-2026",
            "bucket": "terraweek-kousik-2026",
            "region": "ap-south-1"
            ...
          }
        }
      ]
    },
    {
      "type": "aws_instance",
      "name": "my_ec2",
      ...
    }
  ]
}
Step 2: Explore state with Terraform commands
bash# Human-readable view of all resources
terraform show

# List just the resource addresses
terraform state list
Expected state list output:
aws_instance.my_ec2
aws_s3_bucket.my_bucket
bash# Detailed view of a specific resource
terraform state show aws_s3_bucket.my_bucket
terraform state show aws_instance.my_ec2
Expected state show for EC2:
# aws_instance.my_ec2:
resource "aws_instance" "my_ec2" {
    ami                          = "ami-0f5ee92e2d63afc18"
    arn                          = "arn:aws:ec2:ap-south-1:..."
    id                           = "i-0xxxxxxxxxxxxxxxxx"
    instance_state               = "running"
    instance_type                = "t2.micro"
    private_ip                   = "172.31.x.x"
    public_ip                    = "13.x.x.x"
    tags                         = {
        "Name" = "TerraWeek-Day1"
    }
    ...
}

💡 Three critical rules about the state file:
1. Never manually edit it. The state file is a database. Manual edits corrupt it and cause Terraform to lose track of real resources — leading to duplicate resource creation or failed destroys.
2. Never commit it to Git. The state file contains sensitive data — private IPs, resource ARNs, sometimes even passwords. It also causes merge conflicts in team workflows.
3. In production, store it remotely. Use an S3 backend + DynamoDB lock table so your whole team shares one state file without conflicts. You'll learn this in upcoming days.

✅ Verify: terraform state list shows both aws_instance.my_ec2 and aws_s3_bucket.my_bucket.

✅ TASK 6 — Modify, Plan, and Destroy
Step 1: Modify the EC2 tag in main.tf
bash# Change TerraWeek-Day1 to TerraWeek-Modified
sed -i 's/TerraWeek-Day1/TerraWeek-Modified/' main.tf

# Verify the change
grep "TerraWeek" main.tf
Step 2: Run terraform plan and read the symbols
bashterraform plan
Expected output:
aws_instance.my_ec2: Refreshing state... [id=i-0xxxxxxxxxxxxxxxxx]
aws_s3_bucket.my_bucket: Refreshing state... [id=terraweek-kousik-2026]

Terraform will perform the following actions:

  # aws_instance.my_ec2 will be updated in-place
  ~ resource "aws_instance" "my_ec2" {
        id    = "i-0xxxxxxxxxxxxxxxxx"
      ~ tags  = {
          ~ "Name" = "TerraWeek-Day1" -> "TerraWeek-Modified"
        }
    }

Plan: 0 to add, 1 to change, 0 to destroy.

💡 What the symbols mean:

+ CREATE — new resource will be added
- DESTROY — resource will be deleted
~ UPDATE IN-PLACE — resource exists, only this attribute changes (no rebuild)
-/+ DESTROY AND RECREATE — change requires deleting and rebuilding (e.g. changing an AMI ID forces instance replacement)

A tag change is just metadata — AWS can update it without touching the instance. That's why this is ~ (in-place update).

Step 3: Apply the change
bashterraform apply
Type yes.
Expected:
aws_instance.my_ec2: Modifying... [id=i-0xxxxxxxxxxxxxxxxx]
aws_instance.my_ec2: Modifications complete after 5s

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
Step 4: Verify the tag changed
bashaws ec2 describe-instances \
  --filters "Name=tag:Name,Values=TerraWeek-Modified" \
  --query "Reservations[*].Instances[*].[InstanceId,Tags[?Key=='Name'].Value|[0]]" \
  --output table \
  --region ap-south-1
Expected: Shows your instance with TerraWeek-Modified.
Step 5: Destroy everything
bashterraform destroy
Expected output:
Terraform will perform the following actions:

  # aws_instance.my_ec2 will be destroyed
  - resource "aws_instance" "my_ec2" { ... }

  # aws_s3_bucket.my_bucket will be destroyed
  - resource "aws_s3_bucket" "my_bucket" { ... }

Plan: 0 to add, 0 to change, 2 to destroy.

Do you really want to destroy all resources? yes

aws_instance.my_ec2: Destroying... [id=i-0xxxxxxxxxxxxxxxxx]
aws_s3_bucket.my_bucket: Destroying... [id=terraweek-kousik-2026]
aws_instance.my_ec2: Still destroying... [10s elapsed]
aws_instance.my_ec2: Destruction complete after 32s
aws_s3_bucket.my_bucket: Destruction complete after 2s

Destroy complete! Resources: 2 destroyed.
Step 6: Verify everything is gone
bash# EC2 - should show terminated or nothing
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=90DaysDevOps" \
  --query "Reservations[*].Instances[*].[InstanceId,State.Name]" \
  --output table \
  --region ap-south-1

# S3 - should return nothing
aws s3 ls | grep terraweek

# State file - should be empty
terraform state list
Expected: No resources. State file is empty.
✅ Verify: AWS console shows no running instances and no S3 bucket. terraform state list returns nothing.

📸 Take Your Screenshots
bash# Screenshot 1 — terraform apply creating S3 bucket
# Screenshot 2 — terraform apply creating EC2 instance
# Screenshot 3 — S3 bucket in AWS console
# Screenshot 4 — EC2 instance in AWS console with TerraWeek-Day1 tag
# Screenshot 5 — terraform plan showing ~ for tag update
# Screenshot 6 — terraform destroy complete

3. COMMON MISTAKES & FIXES
MistakeWhy it happensFixBucketAlreadyExists errorS3 bucket names are global — someone else took that nameChange bucket name to something more unique: add your full name + random numberInvalidAMIID.NotFoundAMI IDs are region-specificRun the aws ec2 describe-images command above to get the correct AMI for your regionAuthFailure or UnauthorizedOperationIAM user doesn't have EC2/S3 permissionsAttach AmazonEC2FullAccess and AmazonS3FullAccess policies to your IAM user in AWS consoleterraform init fails with provider download errorNetwork issue or HashiCorp registry timeoutTry again. If persistent, check VPN or proxy settingsState file shows resources but they don't exist in AWSResources were deleted manually in AWS console (outside Terraform)Run terraform refresh to sync state, then terraform destroy to clean upterraform destroy fails on S3 bucketBucket is not empty — Terraform can't delete non-empty buckets by defaultAdd force_destroy = true to aws_s3_bucket resource, then destroyError: No valid credential sourcesAWS CLI not configured or credentials expiredRun aws configure again and verify with aws sts get-caller-identity.terraform/ committed to GitForgot to add .gitignore firstAdd to .gitignore, then git rm -r --cached .terraform/

4. README.md (HUMANIZED — LINKEDIN READY)
markdown# Day 61 – Introduction to Terraform and My First AWS Infrastructure

## What I Learned Today

For the past ten days I've been deploying workloads onto
Kubernetes. But there's a layer below that I've been taking
for granted: the actual servers, networks, and storage that
Kubernetes runs on. Today I started learning how to create
that infrastructure from code.

**Infrastructure as Code** means treating your servers and
cloud resources the same way you treat application code —
written in files, reviewed in pull requests, version-
controlled in Git, and reproducible on any machine. Before
IaC, teams clicked through AWS console UIs to create
resources. That works once, for one person. It doesn't scale,
can't be reviewed, and can't be rolled back.

Terraform is the tool that makes IaC practical. You describe
what you want — an EC2 instance, an S3 bucket, a VPC — in
a `.tf` file, and Terraform figures out how to create it.
It's declarative: you say *what* you want, not *how* to get
there. And it works on any cloud — AWS, GCP, Azure, even
Kubernetes — with the same workflow.

Today's workflow in four commands:

````bash
terraform init     # Pull the AWS provider plugin
terraform plan     # Preview changes before spending money
terraform apply    # Create real infrastructure
terraform destroy  # Tear it all down cleanly




I created an S3 bucket and an EC2 instance, modified a tag
(and watched `terraform plan` show `~` for an in-place
update instead of `-/+` for destroy-and-recreate — that
distinction matters for production), then destroyed
everything in one command.

The state file was the most important concept of the day.
Terraform tracks every resource it creates in `terraform.tfstate`.
That's how it knows "the S3 bucket already exists, only the
EC2 instance needs to be created" when you add a new resource.
The rules: never edit it manually, never commit it to Git,
and in production — store it in S3 with DynamoDB locking so
your whole team shares one source of truth.





## Commands Used

````bash
# Installation
terraform -version
aws sts get-caller-identity

# Workflow
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply

# State inspection
terraform show
terraform state list
terraform state show aws_instance.my_ec2
terraform state show aws_s3_bucket.my_bucket

# Cleanup
terraform destroy
````

## What the State File Contains

The `terraform.tfstate` file is a JSON database storing:
- Every resource Terraform created (type, name, ID)
- All attributes of each resource (ARN, IP, region, tags)
- Provider metadata and Terraform version

Why it matters: without the state file, Terraform can't
tell what already exists and would try to recreate everything
on the next apply. Why not commit it to Git: it may contain
sensitive values, causes merge conflicts in team workflows,
and grows large over time. In production, store it remotely
in an S3 backend.

## Challenges Faced

S3 bucket names are global across all AWS accounts worldwide —
not just yours. My first name attempt failed with
`BucketAlreadyExists`. Adding my name and the year fixed it.

Also had to double-check the AMI ID — they're region-specific.
`ami-0f5ee92e2d63afc18` works for `ap-south-1` (Mumbai),
but would fail in any other region. The `aws ec2
describe-images` command to fetch the latest valid AMI is
the right habit for any production Terraform.

## Final Outcome

- ✅ Terraform v1.10+ installed and verified
- ✅ AWS CLI configured and `sts get-caller-identity` working
- ✅ S3 bucket created via `terraform apply`
- ✅ EC2 t2.micro instance created with correct name tag
- ✅ State file inspected — understood what it stores
- ✅ Tag modified — observed `~` in-place update (no rebuild)
- ✅ `terraform destroy` cleanly removed both resources
- ✅ AWS console verified: no running instances, no buckets
- ✅ `.gitignore` set up correctly — state files excluded
