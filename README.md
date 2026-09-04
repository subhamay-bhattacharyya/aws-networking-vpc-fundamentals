# AWS Networking VPC Fundamentals

A CloudFormation-based project for deploying AWS VPC infrastructure using GitHub Actions with OIDC authentication.

## Project Structure

```
.
├── .github/workflows/
│   ├── deploy.yaml          # Main deployment workflow
│   └── destroy.yaml         # Stack destruction workflow
├── infra/
│   ├── template.yaml        # CloudFormation template
│   └── parameters.json      # Stack parameters
└── README.md
```

## CloudFormation Template

### File: `infra/template.yaml`

The CloudFormation template defines an S3 bucket with the following features:

**Parameters:**
- `BucketBaseName` (default: `subhamay-test-bucket`) - Base name for the S3 bucket

**Resources:**
- S3 Bucket with public access blocking enabled

**Outputs:**
- S3 bucket name
- S3 bucket ARN

### Bucket Naming Convention

The bucket name is constructed as:
```
{BucketBaseName}-{AWS::AccountId}-{AWS::Region}
```

Example: `subhamay-test-bucket-123456789012-us-east-1`

## Parameters

### File: `infra/parameters.json`

CloudFormation parameters are defined in CloudFormation parameter format:

```json
[
  {
    "ParameterKey": "BucketBaseName",
    "ParameterValue": "subhamay-test-bucket"
  }
]
```

## GitHub Actions Workflow

### Deploy Workflow (`.github/workflows/deploy.yaml`)

The deployment workflow consists of three jobs that run in sequence:

#### 1. **Validate Job**
- Validates the CloudFormation template syntax
- Runs CloudFormation linting checks
- Prerequisites for uploading template to S3

#### 2. **Upload to S3 Job**
- Uploads the validated template to S3
- Constructs the template URL for CloudFormation deployment
- **Output:** `template_url` - Used by the deploy job

#### 3. **Deploy Job**
- Creates or updates the CloudFormation stack
- Polls for stack creation/update completion
- Displays stack information and outputs
- Handles parameter files conditionally

**Workflow Trigger:**
- On push to `main` branch with changes in:
  - `infra/**` (template files)
  - `.github/workflows/deploy.yaml` (workflow itself)
- On pull request with changes in `infra/**`

**Environment:**
- Requires `devl` environment with OIDC authentication

### Destroy Workflow (`.github/workflows/destroy.yaml`)

Manually triggered workflow to delete the CloudFormation stack:

- Checks if stack exists before deletion
- Waits for stack deletion to complete
- Triggered via `workflow_dispatch` (GitHub Actions UI)

## Environment Variables

**Workflow Environment Variables:**
- `AWS_REGION` - AWS region (from `vars.AWS_REGION`)
- `CFN_TEMPLATES_S3_BUCKET` - S3 bucket for CloudFormation templates (from `vars.CFN_TEMPLATES_S3_BUCKET`)
- `STACK_NAME` - CloudFormation stack name (defaults to repository name)
- `AWS_ACCOUNT_ID` - AWS account ID (from `vars.AWS_ACCOUNT_ID`)
- `OIDC_ROLE_NAME` - OIDC role name for authentication (from `vars.OIDC_ROLE_NAME`)

**Required GitHub Secrets:**
- `AWS_ACCOUNT_ID` - AWS account ID for OIDC role assumption
- `AWS_ROLE_ARN` (optional) - Role ARN for validation step

**Required GitHub Variables:**
- `AWS_REGION` - AWS region
- `CFN_TEMPLATES_S3_BUCKET` - S3 bucket for templates
- `OIDC_ROLE_NAME` - OIDC role name

## Authentication

The workflows use **GitHub OIDC (OpenID Connect)** for AWS authentication:

- No long-lived AWS credentials stored in GitHub Secrets
- AWS role must be configured for GitHub OIDC provider
- Role must have permissions for:
  - CloudFormation operations (create, update, describe stacks)
  - S3 operations (upload templates)
  - IAM and related services

## Stack Operations

### Create Stack

When deploying for the first time:
1. Validates the template
2. Uploads template to S3
3. Creates new CloudFormation stack
4. Waits for `CREATE_COMPLETE` status

### Update Stack

When template or parameters change:
1. Validates the template
2. Uploads template to S3
3. Updates existing CloudFormation stack
4. Waits for `UPDATE_COMPLETE` status
5. Handles `CREATE_COMPLETE` (no changes needed)

### Delete Stack

To destroy the stack:
1. Navigate to **Actions** → **Destroy Stack**
2. Click **Run workflow**
3. Stack is deleted and waits for `DELETE_COMPLETE`

## GitHub Actions Versions

**Node.js 24 Compatible Versions:**
- `actions/checkout@v7.0.1` (or higher v7.x)
- `aws-actions/configure-aws-credentials@v6.2.4` (or higher v6.x)

## CloudFormation Capabilities

The workflow applies the following capabilities:
- `CAPABILITY_NAMED_IAM` - Allows creation/update of IAM resources with custom names
- `CAPABILITY_AUTO_EXPAND` - Allows auto-expansion of macros

## Troubleshooting

### Stack Deployment Failed

1. Check GitHub Actions workflow logs
2. Review CloudFormation console for error details
3. Verify template syntax in `infra/template.yaml`
4. Ensure parameters in `infra/parameters.json` are valid

### Template Upload Issues

- Verify S3 bucket exists and is accessible
- Check AWS credentials and OIDC role permissions
- Ensure `CFN_TEMPLATES_S3_BUCKET` variable is correct

### Stack Status Issues

- **DELETE_IN_PROGRESS**: Stack is being deleted, wait for completion
- **ROLLBACK_COMPLETE**: Previous operation failed, review logs
- **UPDATE_ROLLBACK_COMPLETE**: Update failed and was rolled back

## Deployment Workflow Diagram

```
Push to main branch
        ↓
    [Validate]
        ↓
  (Validate template & lint)
        ↓
  [Upload to S3]
        ↓
  (Upload template, generate URL)
        ↓
    [Deploy]
        ↓
  (Create/Update stack)
        ↓
  (Wait for completion)
        ↓
  (Display results)
```

## Useful Commands

### Validate Template Locally

```bash
aws cloudformation validate-template \
  --template-body file://./infra/template.yaml \
  --region us-east-1
```

### Check Stack Status

```bash
aws cloudformation describe-stacks \
  --stack-name aws-networking-vpc-fundamentals \
  --region us-east-1 \
  --query 'Stacks[0].StackStatus'
```

### Delete Stack Manually

```bash
aws cloudformation delete-stack \
  --stack-name aws-networking-vpc-fundamentals \
  --region us-east-1
```

## License

MIT
