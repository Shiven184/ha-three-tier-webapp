# Project 3 - Setup Guide and How It Works

---

## What This Project Does in Simple Words

Imagine a popular website like Flipkart's internal tool.

You need it to:
- Handle traffic from thousands of users at the same time
- Never go down even if one server crashes
- Never go down even if one entire AWS data centre has a problem
- Scale up automatically when traffic spikes (like during a sale)
- Keep the database completely hidden from the internet

That is exactly what this project builds. One CloudFormation template
creates the entire setup from scratch — network, servers, load balancer,
and database — all wired together correctly with the right security rules.

---

## What Each Piece Does

VPC (Virtual Private Cloud)
  Your own private section of AWS. Like a fenced office building.
  Nobody from the internet can enter unless you explicitly allow it.

Public Subnets (2 of them, in different AZs)
  The reception area. Only the ALB and NAT Gateway sit here.
  Reachable from the internet.

Private Subnets (2 of them, in different AZs)
  The back office. Your EC2 app servers live here.
  Not reachable directly from the internet. Only ALB can talk to them.

DB Subnets (2 of them, in different AZs)
  The vault. RDS MySQL lives here.
  Only EC2 servers in private subnets can talk to it. Nothing else.

Application Load Balancer (ALB)
  The receptionist. Receives every user request.
  Distributes traffic across your EC2 servers evenly.
  Keeps checking if each EC2 is healthy. Stops sending traffic to sick ones.

Auto Scaling Group (ASG)
  The manager. Watches CPU usage across all EC2 servers.
  If CPU goes above 70%, it automatically adds more EC2 instances.
  If CPU drops, it removes extra instances to save cost.
  If an instance crashes, it replaces it automatically.

NAT Gateway
  Lets your private EC2 servers reach the internet outbound (for OS updates,
  downloading packages) without being reachable inbound from the internet.

RDS MySQL
  Your database. Encrypted storage, automated backups.
  Sits in the most locked-down subnets. EC2 connects to it on port 3306.

Security Groups (3 of them)
  Firewall rules between each layer:
  Internet can only hit ALB on port 80
  ALB can only hit EC2 on port 80
  EC2 can only hit RDS on port 3306
  Nothing else is allowed in any direction.

---

## Key Difference from Project 2 CI/CD

Project 2 (Serverless) auto-deploys on every git push.
This is fine because Lambda and DynamoDB cost nothing when idle.

Project 3 costs money every hour it runs:
  ALB      ~$18/month
  NAT GW   ~$5/month  (plus $0.045 per GB of data)
  2x EC2   ~$15/month (Free Tier eligible first year)
  RDS      ~$15/month (Free Tier eligible first year)
  Total    ~$38/month

For this reason the pipeline uses manual triggers (workflow_dispatch).
You click Deploy when you want it up. You click Destroy when you are done.

This is also real-world practice. Production infrastructure pipelines
in most companies require a human to approve before deploying.

---

## Step by Step: First Time Setup

### 1. Create GitHub repository

Go to github.com > New repository
Name: ha-three-tier-webapp
Visibility: Public
Do not add README or gitignore (we have our own)
Click Create repository

### 2. Create an EC2 Key Pair in AWS

This is required by the template. It lets you SSH into EC2 if needed.

AWS Console > EC2 > Key Pairs > Create key pair
Name: portfolio-keypair
Type: RSA
Format: .pem
Click Create

Download and save the .pem file somewhere safe on your laptop.
Never commit this file to GitHub.

### 3. Add these secrets to GitHub

Go to your repo > Settings > Secrets and variables > Actions

Add these four secrets:

  Name: AWS_ACCESS_KEY_ID
  Value: your IAM user access key

  Name: AWS_SECRET_ACCESS_KEY
  Value: your IAM user secret key

  Name: AWS_REGION
  Value: ap-south-1

  Name: EC2_KEY_PAIR_NAME
  Value: portfolio-keypair   (just the name, not the file)

  Name: DB_PASSWORD
  Value: choose a strong password e.g. Portfolio2025Secure!

Note: DB_PASSWORD must be at least 8 characters. No @ or / characters.

### 4. Push your code to GitHub

Open PowerShell in your project3 folder:

  git init
  git add .
  git commit -m "Initial commit: HA 3-Tier Web App"
  git branch -M main
  git remote add origin https://github.com/YOURUSERNAME/ha-three-tier-webapp.git
  git push -u origin main

  git checkout -b dev
  git push origin dev

### 5. Trigger the validation pipeline

Push any small change to the dev branch.
Go to GitHub > Actions and watch the Lint + Validate job run.
This does NOT create any AWS resources yet. It just checks your template is correct.

### 6. Deploy to dev (creates actual AWS resources)

Go to GitHub > Actions
Click "Deploy to Dev (3-Tier App)" in the left panel
Click "Run workflow" button (top right)
Select branch: dev
Action: deploy
Click "Run workflow"

Watch the pipeline. It takes 15-20 minutes because:
  RDS takes ~10 minutes to create
  NAT Gateway takes ~2 minutes
  EC2 instances need to boot and pass health checks

### 7. Get your ALB URL

When pipeline finishes, click the deploy job and scroll to the
"Print ALB URL" step. Copy the URL. Open it in your browser.

You should see the "Project 3 - HA 3-Tier App" page showing
the Instance ID and Availability Zone of whichever EC2 served your request.
Refresh a few times - you may see it switch between AZ-a and AZ-b.

### 8. Take screenshots for your portfolio

Take screenshots of:
  - The running website in your browser
  - GitHub Actions showing a successful deployment
  - AWS Console > CloudFormation showing the ha-webapp-dev stack
  - AWS Console > EC2 showing 2 running instances in different AZs
  - AWS Console > RDS showing your database

### 9. Destroy when done (IMPORTANT - stops charges)

Go to GitHub > Actions
Click "Deploy to Dev (3-Tier App)"
Click "Run workflow"
Action: destroy
Run it

This deletes all resources. Charges stop immediately after.

---

## Your Daily Workflow After Setup

Make a change to the template:

  git checkout dev
  (edit cloudformation/three-tier.yaml)
  git add .
  git commit -m "what you changed"
  git push origin dev

Validation runs automatically.

When ready to test in AWS:
  GitHub > Actions > Deploy Dev > Run workflow > deploy

When done testing:
  GitHub > Actions > Deploy Dev > Run workflow > destroy

When ready for prod:
  Create a Pull Request from dev to main on GitHub
  Merge it
  GitHub > Actions > Deploy Prod > Run workflow > deploy > type CONFIRM

---

## What to Say in Interviews

"Project 3 is a highly available 3-tier architecture on AWS. I designed a custom
VPC with 6 subnets across 2 Availability Zones - public for the load balancer,
private for EC2 app servers, and isolated DB subnets for RDS MySQL. The Auto
Scaling Group maintains minimum 2 instances and scales out when CPU exceeds 70%.
All security group rules follow least-privilege - each tier only accepts traffic
from the tier directly above it."

"For the CI/CD pipeline, I made a deliberate choice to use manual workflow_dispatch
triggers rather than auto-deploy on push. This reflects real-world practice for
infrastructure that incurs hourly cost - you want a human in the loop before
resources are created or modified."

---

## Difference Between This and Project 2 Pipeline

Project 2 (Serverless)              Project 3 (Infrastructure)
sam build then sam deploy           aws cloudformation deploy directly
Auto-deploy on every push           Manual trigger to control cost
pytest unit tests                   cfn-lint + aws validate as the test step
Free tier - costs nothing running   ~$38/month while running
Deploy takes 2-3 minutes            Deploy takes 15-20 minutes
