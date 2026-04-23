# OpenID Connect (OIDC) in GitHub Actions for AWS

A comprehensive guide to securely deploy a static website to AWS S3 using GitHub Actions with OpenID Connect (OIDC) for authentication, eliminating the need for long-lived AWS credentials.

---

## Table of Contents

1. [Overview](#overview)
2. [What is OIDC?](#what-is-oidc)
3. [Prerequisites](#prerequisites)
4. [Step-by-Step Setup](#step-by-step-setup)
5. [How It Works](#how-it-works)
6. [File Structure](#file-structure)
7. [Troubleshooting](#troubleshooting)

---

## Overview

This repository demonstrates how to:
- Set up OpenID Connect (OIDC) between GitHub and AWS
- Authenticate GitHub Actions workflows securely without storing AWS credentials
- Deploy a static website (`index.html`) to an AWS S3 bucket automatically on every push to the `main` branch

**Benefits of using OIDC:**
- ✅ No long-lived AWS access keys needed
- ✅ Short-lived, temporary credentials for each workflow run
- ✅ Improved security and reduced risk of credential leaks
- ✅ Audit trail in AWS CloudTrail
- ✅ Easier credential rotation

---

## What is OIDC?

**OpenID Connect (OIDC)** is an open authentication protocol that allows secure verification of identity without exchanging passwords. In this context:

- **GitHub** acts as the identity provider (IdP)
- **AWS** is the service provider that trusts GitHub's identity tokens
- When a GitHub Actions workflow runs, GitHub issues a cryptographically signed token
- AWS verifies this token and grants temporary credentials to the workflow
- The workflow then uses these credentials to access AWS resources (S3 in this case)

---

## Prerequisites

Before you begin, you'll need:

1. **GitHub Repository**: This repository (with access to GitHub Actions)
2. **AWS Account**: An active AWS account with appropriate permissions
3. **AWS IAM Role**: Created specifically for GitHub Actions
4. **S3 Bucket**: Created to store your static website files
5. **GitHub Variables**: Configured in your repository settings

---

## Step-by-Step Setup

### Step 1: Create an S3 Bucket

1. Log in to the [AWS Management Console](https://console.aws.amazon.com/)
2. Navigate to **S3** → **Create Bucket**
3. Enter bucket name: `deploy-static-site-to-s3`
4. Choose your preferred region (default: `us-east-2`)
5. Click **Create Bucket**

### Step 2: Enable S3 Static Website Hosting (Optional)

1. Go to your S3 bucket
2. Click **Properties** → **Static website hosting**
3. Enable static website hosting
4. Set index document to `index.html`
5. Save changes

### Step 3: Create an IAM Role for GitHub

1. Navigate to **IAM** → **Roles** → **Create Role**
2. Choose **Custom trust policy** (not "AWS service")
3. Paste the trust policy from `trust-policy.json` (see below)
4. Update the policy with:
   - Replace `YOUR_ACCOUNT_ID` with your actual AWS Account ID
   - Replace the GitHub repo path with your actual repository path
5. Click **Next**

### Step 4: Add Permissions to the Role

1. In the permissions step, click **Create Policy**
2. Use the following policy to allow S3 uploads:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::deploy-static-site-to-s3",
        "arn:aws:s3:::deploy-static-site-to-s3/*"
      ]
    }
  ]
}
```

3. Name the policy: `GitHubActionsS3Policy`
4. Attach this policy to your GitHub role

### Step 5: Configure GitHub Repository Variables

1. Go to your GitHub repository
2. Navigate to **Settings** → **Secrets and Variables** → **Variables**
3. Create a new variable:
   - **Name**: `AWS_ROLE_ARN`
   - **Value**: The ARN of the role you created (format: `arn:aws:iam::YOUR_ACCOUNT_ID:role/GitHubActionsRole`)

### Step 6: Prepare Your Static Files

1. Create or update `index.html` in the root of your repository
2. Example:

```html
<!DOCTYPE html>
<html>
<head>
    <title>My Static Site</title>
</head>
<body>
    <h1>Welcome to my static website!</h1>
    <p>Deployed via GitHub Actions and OIDC</p>
</body>
</html>
```

### Step 7: Deploy!

Simply push changes to the `main` branch:

```bash
git add index.html
git commit -m "Update website"
git push origin main
```

GitHub Actions will automatically:
1. Check out your code
2. Assume the AWS role using OIDC
3. Sync your files to S3
4. Your website is now live!

---

## How It Works

### The Workflow Process

```
┌──────────────────────────────────────────────────────────────┐
│ 1. Push to main branch                                       │
└─────────────────────┬──────────────────────────────────────────┘
                      │
┌─────────────────────▼──────────────────────────────────────────┐
│ 2. GitHub Actions workflow triggers                            │
└─────────────────────┬──────────────────────────────────────────┘
                      │
┌─────────────────────▼──────────────────────────────────────────┐
│ 3. Checkout code from repository                              │
└─────────────────────┬──────────────────────────────────────────┘
                      │
┌─────────────────────▼──────────────────────────────────────────┐
│ 4. Generate OIDC token from GitHub                            │
└─────────────────────┬──────────────────────────────────────────┘
                      │
┌─────────────────────▼──────────────────────────────────────────┐
│ 5. Exchange token for temporary AWS credentials via OIDC       │
└─────────────────────┬──────────────────────────────────────────┘
                      │
┌─────────────────────▼──────────────────────────────────────────┐
│ 6. Verify AWS identity with STS                              │
└─────────────────────┬──────────────────────────────────────────┘
                      │
┌─────────────────────▼──────────────────────────────────────────┐
│ 7. Sync files to S3 bucket                                   │
└─────────────────────┬──────────────────────────────────────────┘
                      │
┌─────────────────────▼──────────────────────────────────────────┐
│ 8. Website live and accessible!                              │
└──────────────────────────────────────────────────────────────────┘
```

### Workflow File Breakdown

The workflow file (`.github/workflows/deploy-static-site.yaml`) contains:

- **Trigger**: Runs on every push to the `main` branch
- **Permissions**: Grants `id-token: write` permission for OIDC token generation
- **Checkout**: Pulls your latest code
- **AWS Credentials**: Uses OIDC to get temporary AWS credentials
- **Identity Verification**: Confirms the AWS role assumption was successful
- **S3 Sync**: Uploads files to your S3 bucket with caching headers

---

## File Structure

```
OpenID-Connect-in-GitHub-Actions-for-AWS/
├── .github/
│   └── workflows/
│       └── deploy-static-site.yaml    # GitHub Actions workflow
├── index.html                          # Your static website
├── trust-policy.json                   # AWS trust relationship policy
└── README.md                           # This file
```

### Key Files Explained

**`deploy-static-site.yaml`**
- Defines the GitHub Actions workflow
- Specifies when to run (on push to main)
- Contains all deployment steps

**`trust-policy.json`**
- AWS IAM trust policy
- Tells AWS to trust tokens from GitHub
- Specifies which GitHub repository and branch can assume the role

**`index.html`**
- Your static website content
- Deployed to S3 on every push

---

## Troubleshooting

### "Role assumption failed"
- **Cause**: The trust policy doesn't match your GitHub repository path
- **Solution**: Update `trust-policy.json` with your correct repo owner and name

### "S3 sync failed"
- **Cause**: The IAM role doesn't have S3 permissions
- **Solution**: Check the IAM policy attached to your role includes S3 actions

### "Variable AWS_ROLE_ARN not found"
- **Cause**: The GitHub repository variable isn't set
- **Solution**: Add `AWS_ROLE_ARN` variable in repository Settings → Secrets and Variables

### Workflow doesn't trigger
- **Cause**: Workflow might be disabled or push is to wrong branch
- **Solution**: Ensure workflow is enabled and you're pushing to `main` branch

---

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS OIDC Provider Setup](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
- [AWS CLI S3 Sync Command](https://docs.aws.amazon.com/cli/latest/reference/s3/sync.html)
