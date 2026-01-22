# AWS 101 for Trainium Development Workshop

## Introduction

This hands-on workshop helps you get started with AWS Trainium instances for machine learning development. AWS Trainium is a custom ML accelerator chip designed by AWS specifically for training deep learning models with high performance and cost efficiency. By the end of this one-hour session, you'll be able to set up a secure AWS environment, launch Trainium instances, and run basic ML workloads.

### What You'll Accomplish

By completing this workshop, you will:
- Create and secure an AWS environment for ML development
- Launch and connect to a Trainium instance
- Run your first ML workload on AWS Trainium
- Learn how to manage instances to control costs
- Set up a remote development environment

> **Note**: This guide is updated for April 2025 with the latest AWS Neuron SDK (version 2.22.0) and supports both Trn1 and the newer Trn2 instances.

## Prerequisites

- A laptop with internet connection
- An AWS account (new or existing) with appropriate permissions to create IAM users, EC2 instances, and security groups
- Basic familiarity with command line interfaces (Bash for Linux/macOS or PowerShell for Windows)
- Basic knowledge of Python and PyTorch (for the ML examples)
- Understanding of SSH for connecting to remote instances

If you're new to AWS, review the [AWS Getting Started Guide](https://aws.amazon.com/getting-started/) before beginning this workshop.

---

## Workshop Outline

1. [Setting Up Your AWS Environment](#1-setting-up-your-aws-environment)
2. [Configuring Budget Alerts](#2-configuring-budget-alerts)
3. [Installing and Configuring AWS Command Line Interface (CLI)](#3-installing-and-configuring-aws-cli)
4. [Launching a Trainium Instance](#4-launching-a-trainium-instance)
5. [Connecting to Your Instance](#5-connecting-to-your-instance)
6. [Setting Up Remote Development](#6-setting-up-remote-development)
7. [Hello World on Trainium](#7-hello-world-on-trainium)
8. [Instance Management](#8-instance-management)
9. [Cleanup and Best Practices](#9-cleanup-and-best-practices)
10. [Additional Resources](#10-additional-resources)
11. [Trainium Management Script](#11-trainium-management-script)
12. [Common Troubleshooting and Next Steps](#12-common-troubleshooting-and-next-steps)

> **Important**: This workshop uses the **US West (Oregon)** region (`us-west-2`), which supports Trainium instances. If you need to use a different region, first verify that it supports Trainium instances using the commands provided in section 4.

> **Note on Placeholders**: Throughout this guide, you'll see placeholder values like `<your-instance-ip>` or AMI IDs like `ami-0123456789abcdef`. These are just examples and must be replaced with your actual values. **Never use placeholder AMI IDs in actual commands**.

> **Quick Reference Card**: A single-page summary of key commands used in this workshop is available in the [Appendices](#appendices) section.

---

## 1. Setting Up Your AWS Environment

### Creating an IAM User

For security best practices, you should use an IAM user instead of your root account. This creates a user with limited permissions specific to the workshop needs:

1. Sign in to the [AWS Management Console](https://console.aws.amazon.com/) using your root credentials
2. Navigate to the IAM service (search for "IAM" in the top search bar)
3. In the left navigation panel, click "Users" and then "Create user"
4. Name your user (e.g., "trainium-workshop-user") and click "Next"
5. Select "Attach policies directly" and create a custom policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:*",
                "s3:*",
                "cloudwatch:*",
                "budgets:*",
                "pricing:*"
            ],
            "Resource": "*"
        }
    ]
}
```

> **Note**: In a production environment, you would use more restrictive permissions following the principle of least privilege. This policy is simplified for workshop purposes.

6. Click "Next" and then "Create user"

### Creating Access Keys for CLI

1. From the IAM dashboard, select your newly created user
2. Go to the "Security credentials" tab
3. Under "Access keys", click "Create access key"
4. Select "Command Line Interface (CLI)" as your use case
5. (Optional) Add a tag for better tracking (e.g., Key: Purpose, Value: TrainiumWorkshop)
6. Click "Create access key"
7. **IMPORTANT**: Download the .csv file or copy your Access Key ID and Secret Access Key - you won't be able to access the secret again!

### Creating a Key Pair for SSH

1. Navigate to EC2 in the AWS Console
2. In the left navigation panel, under "Network & Security", select "Key Pairs"
3. Click "Create key pair"
4. Name your key (e.g., "trainium-workshop-key")
5. For key pair type, choose "RSA"
6. For private key format: 
   - For macOS/Linux users: Choose .pem
   - For Windows users: Choose .ppk if using PuTTY, or .pem if using OpenSSH
7. Click "Create key pair" and save the file securely (usually in `~/.ssh/` directory)
8. Change permissions (for .pem on macOS/Linux):
   ```bash
   chmod 400 ~/.ssh/trainium-workshop-key.pem
   ```

> **Tips**:
> - Save this key in a secure location, as you'll need it to connect to your instances
> - If you lose this key, you won't be able to connect to your instances and will need to create new ones
> - Never share your private key with others

---

<!-- ## 2. Configuring Budget Alerts

### Using the Console

1. Go to the [AWS Billing Dashboard](https://console.aws.amazon.com/billing/)
2. In the left navigation pane, click "Budgets"
3. Click "Create budget"
4. Select "Cost budget" and click "Next"
5. Configure your budget:
   - Name: "TrainiumWorkshopBudget"
   - Period: Monthly
   - Budget amount: Set to $1,500 (or your desired amount)
6. Set up alerts at 25%, 50%, 75%, and 90% thresholds
7. Add your email address as a notification recipient
8. Review and click "Create budget"

--- -->

## 2. Installing and Configuring AWS Command Line Interface (CLI)

The AWS Command Line Interface (CLI) allows you to interact with AWS services from the terminal. This section covers installation and setup for different operating systems.

### For macOS

#### Using Homebrew (recommended)
```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install AWS CLI
brew install awscli

# Verify installation
aws --version
```

#### Using the official installer
```bash
# Download the installer
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"

# Install the package
sudo installer -pkg AWSCLIV2.pkg -target /

# Verify installation
aws --version
```

### For Windows using WSL (Recommended)

Windows Subsystem for Linux (WSL) provides a Linux environment that works better with developer tools:

1. Install WSL by opening PowerShell as Administrator and running:
```powershell
wsl --install
```

2. Once installed and rebooted, open the Ubuntu terminal from your Start menu
3. Update your Linux distribution:
```bash
sudo apt update && sudo apt upgrade -y
```

4. Install AWS CLI in WSL:
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt-get install unzip
unzip awscliv2.zip
sudo ./aws/install
```

5. Verify installation:
```bash
aws --version
```

6. Your WSL environment will now behave like Linux for all AWS CLI commands in this guide

### For Windows (Native - Alternative)

If you prefer not to use WSL:

1. Download the MSI installer: [AWS CLI MSI Installer for Windows](https://awscli.amazonaws.com/AWSCLIV2.msi)
2. Run the downloaded MSI file and follow installation prompts
3. Open Command Prompt or PowerShell to verify installation:
```
aws --version
```

### Configuring the AWS CLI

After installing the AWS CLI, you need to configure it with your AWS credentials:

1. Open a terminal (or Command Prompt/PowerShell on Windows)
2. Run the following command:

```bash
aws configure
```

3. You'll be prompted to enter the following information:
   - AWS Access Key ID: [Enter your access key]
   - AWS Secret Access Key: [Enter your secret key]
   - Default region name: us-west-2
   - Default output format: json

4. If you don't have access keys yet, create them in the IAM service after creating your IAM user (see Section 1)

5. Verify your configuration:
```bash
aws sts get-caller-identity
```

6. You should see output similar to:
```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/trainium-workshop-user"
}
```

7. Create a credentials file backup (optional but recommended):
```bash
# For macOS/Linux/WSL
cp ~/.aws/credentials ~/.aws/credentials.backup

# For Windows (PowerShell)
Copy-Item -Path "$env:USERPROFILE\.aws\credentials" -Destination "$env:USERPROFILE\.aws\credentials.backup"
```

<!-- ### Using the AWS CLI for Budget Alerts

Now that you have the AWS CLI installed, you can also create budget alerts and check your credit balance using the command line:

To create a budget alert, first create the required JSON files:

Create `budget.json`:
```json
{
    "BudgetName": "TrainiumWorkshopBudget",
    "BudgetLimit": {
        "Amount": "1500",
        "Unit": "USD"
    },
    "BudgetType": "COST",
    "TimeUnit": "MONTHLY"
}
```

Create `notifications.json`:
```json
[
    {
        "Notification": {
            "NotificationType": "ACTUAL",
            "ComparisonOperator": "GREATER_THAN",
            "Threshold": 50.0,
            "ThresholdType": "PERCENTAGE",
            "NotificationState": "ALARM"
        },
        "Subscribers": [
            {
                "SubscriptionType": "EMAIL",
                "Address": "your-email@example.com"
            }
        ]
    }
]
```

Then run the create-budget command:

```bash
aws budgets create-budget \
    --account-id $(aws sts get-caller-identity --query 'Account' --output text) \
    --budget file://budget.json \
    --notifications-with-subscribers file://notifications.json
``` -->

### Checking AWS Cost and Usage

To check your AWS current month spending using the CLI:

```bash
# For Linux:
aws ce get-cost-and-usage \
    --time-period Start=$(date -d "first day of this month" +%Y-%m-%d),End=$(date -d "tomorrow" +%Y-%m-%d) \
    --granularity MONTHLY \
    --metrics UnblendedCost \
    --output table

# For macOS:
aws ce get-cost-and-usage \
    --time-period Start=$(date -v1d +%Y-%m-%d),End=$(date -v+1d +%Y-%m-%d) \
    --granularity MONTHLY \
    --metrics UnblendedCost \
    --output table
```

To check how many AWS credits you've used this month:

```bash
# For Linux:
aws ce get-cost-and-usage \
    --time-period Start=$(date -d "first day of this month" +%Y-%m-%d),End=$(date -d "tomorrow" +%Y-%m-%d) \
    --granularity MONTHLY \
    --metrics UnblendedCost \
    --filter '{"Dimensions": {"Key": "RECORD_TYPE", "Values": ["Credit"]}}' \
    --output table

# For macOS:
aws ce get-cost-and-usage \
    --time-period Start=$(date -v1d +%Y-%m-%d),End=$(date -v+1d +%Y-%m-%d) \
    --granularity MONTHLY \
    --metrics UnblendedCost \
    --filter '{"Dimensions": {"Key": "RECORD_TYPE", "Values": ["Credit"]}}' \
    --output table
```

### Checking Your Remaining AWS Credit Balance

To check your remaining AWS credit balance (how many credits you have left):

1. Log into the [AWS Management Console](https://console.aws.amazon.com/)
2. Navigate to "Billing and Cost Management" dashboard
3. Select "Credits" from the left navigation pane
4. Here you can view:
   - Available credits
   - Remaining balances
   - Expiration dates
   - Which services each credit applies to

> **Note**: There is currently no AWS CLI command to directly check remaining credit balances.

---

## 3. Launching a Trainium Instance
