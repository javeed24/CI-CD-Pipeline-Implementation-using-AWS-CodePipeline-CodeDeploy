# AWS CI/CD Pipeline with CodePipeline, CodeDeploy & Elastic Beanstalk

## Problem Statement
Design and implement a complete Software Development Life Cycle (SDLC)
for a web application using AWS infrastructure and AWS Developer Tools
to automate the build and deployment pipeline.

---

## Architecture Overview

```
GitHub
  ↓
AWS CodeCommit
  ↓
AWS CodePipeline
  ↓
CodeDeploy – QA Stage (EC2)
  ↓ (only if QA passes)
CodeDeploy – Production Stage (EC2)
  ↓
Elastic Beanstalk Environment
```

---

## Tools & Technologies Used

- **GitHub** – Original source code repository
- **AWS CodeCommit** – Managed Git repository on AWS
- **AWS CodeDeploy** – Automated deployment to EC2 instances
- **AWS CodePipeline** – CI/CD pipeline orchestration
- **AWS Elastic Beanstalk** – Managed platform for Stage 3 deployment
- **Amazon EC2** – Deployment target for QA and Production stages
- **AWS IAM** – Roles and permissions for pipeline services

---

## Step 1: Create Website & Push to GitHub

- Created a static website using HTML/CSS
- Pushed source code to a GitHub repository

```bash
git init
git add .
git commit -m "Initial website commit"
git remote add origin https://github.com/<your-username>/<your-repo>.git
git push -u origin master
```

---

## Step 2: Migrate GitHub Repository to AWS CodeCommit

### Create CodeCommit Repository

```bash
aws codecommit create-repository \
  --repository-name companyabc-website \
  --repository-description "Company ABC Website Repository"
```

### Mirror GitHub to CodeCommit

```bash
# Clone from GitHub
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>

# Add CodeCommit as a new remote
git remote add codecommit https://git-codecommit.us-east-1.amazonaws.com/v1/repos/companyabc-website

# Push all branches to CodeCommit
git push codecommit master
```

---

## Step 3: AWS CodeDeploy Setup

### appspec.yml

```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/html
hooks:
  BeforeInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: scripts/start_server.sh
      timeout: 300
      runas: root
  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 300
      runas: root
```

### scripts/install_dependencies.sh

```bash
#!/bin/bash
yum update -y
yum install -y httpd
```

### scripts/start_server.sh

```bash
#!/bin/bash
systemctl start httpd
systemctl enable httpd
```

### scripts/stop_server.sh

```bash
#!/bin/bash
systemctl stop httpd || true
```

### Two Deployment Groups Created

| Deployment Group | Environment | EC2 Tag Key | EC2 Tag Value |
|-----------------|-------------|-------------|---------------|
| QA-DeploymentGroup | QA/Testing | Environment | QA |
| Prod-DeploymentGroup | Production | Environment | Production |

---

## Step 4: AWS CodePipeline – SDLC Automation

### Pipeline Stages

| Stage | Action | Details |
|-------|--------|---------|
| Source | CodeCommit | Monitors master branch for changes |
| QA Deploy | CodeDeploy | Deploys to QA EC2 instances |
| Approval | Manual (optional) | Gate between QA and Production |
| Prod Deploy | CodeDeploy | Deploys to Production EC2 instances |
| Beanstalk | Elastic Beanstalk | Deploys same website to managed env |

### Key Pipeline Rules

- Pipeline is **automatically triggered** on every commit to `master` branch
- **Production stage executes only when QA stage is successful**
- If QA fails, the pipeline stops and Production is never touched

---

## Step 5: Elastic Beanstalk Deployment (Stage 3)

```bash
# Create application
aws elasticbeanstalk create-application \
  --application-name companyabc-app \
  --description "Company ABC Website"

# Create environment
aws elasticbeanstalk create-environment \
  --application-name companyabc-app \
  --environment-name companyabc-env \
  --solution-stack-name "64bit Amazon Linux 2 v4.0.0 running PHP 8.0" \
  --option-settings \
    Namespace=aws:autoscaling:launchconfiguration,\
    OptionName=InstanceType,Value=t2.micro
```

- Same website code from CodeCommit is deployed to Elastic Beanstalk
- Elastic Beanstalk handles provisioning, load balancing, and scaling automatically
- Added as **Stage 3** in the CodePipeline

---

## Pipeline Flow Diagram

```
Commit to master branch
        ↓
  AWS CodeCommit (Source)
        ↓
  Stage 1 – QA Deployment
  CodeDeploy → EC2 (QA)
  Run tests & validation
        ↓
  ✓ Only if QA stage passes
        ↓
  Stage 2 – Production Deployment
  CodeDeploy → EC2 (Production)
        ↓
  Stage 3 – Elastic Beanstalk
  Same website → Managed environment
```

---

## IAM Roles Required

| Role | Used By | Permissions |
|------|---------|-------------|
| CodePipeline-Role | CodePipeline | S3, CodeCommit, CodeDeploy, Beanstalk |
| CodeDeploy-Role | CodeDeploy | EC2, Auto Scaling, S3 |
| EC2-CodeDeploy-Role | EC2 Instances | S3 read, CodeDeploy agent |
| Beanstalk-Role | Elastic Beanstalk | EC2, S3, CloudWatch |

---

## Screenshots

### CodeCommit Repository
![CodeCommit](screenshots/codecommit-repo.png)

### CodeDeploy – QA Deployment Group
![CodeDeploy QA](screenshots/codedeploy-qa.png)

### CodeDeploy – Production Deployment Group
![CodeDeploy Production](screenshots/codedeploy-prod.png)

### CodePipeline – All Stages Overview
![CodePipeline](screenshots/codepipeline-stages.png)

### CodePipeline – QA Stage Success
![QA Success](screenshots/codepipeline-qa-success.png)

### CodePipeline – Production Stage Success
![Prod Success](screenshots/codepipeline-prod-success.png)

### Elastic Beanstalk Environment
![Elastic Beanstalk](screenshots/elasticbeanstalk-env.png)

### Full Pipeline Execution – Success
![Pipeline Success](screenshots/pipeline-success.png)

### Website Running on EC2 Production
![Website EC2](screenshots/website-ec2.png)

### Website Running on Elastic Beanstalk
![Website Beanstalk](screenshots/website-beanstalk.png)

---

## Key Learnings

- Migrated GitHub repository to AWS CodeCommit for fully managed source control
- Created two CodeDeploy deployment groups for QA and Production EC2 environments
- Built a multi-stage CodePipeline with gate-controlled promotion from QA to Production
- Production deployment only triggers when QA stage passes successfully
- Added Elastic Beanstalk as Stage 3 to deploy the same website on a managed platform
- Configured IAM roles with least privilege for each AWS service in the pipeline
- Used appspec.yml to define deployment lifecycle hooks for Apache installation
