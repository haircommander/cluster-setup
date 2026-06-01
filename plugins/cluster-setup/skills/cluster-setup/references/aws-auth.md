# AWS Authentication for Red Hat Users

## Overview

Red Hat uses SAML-based SSO for AWS access. This guide covers setting up temporary credentials for CLI and programmatic access.

## Prerequisites

- Active Red Hat Kerberos credentials
- VPN connection to Red Hat network
- Successfully logged into the AWS web console via SSO

## Installation

### Install AWS CLI (if not already installed)

```bash
# Option 1: Using system package manager
sudo dnf install -y awscli

# Option 2: Using pip
pip install awscli

# Option 3: Using Homebrew
brew install awscli
```

### Install Red Hat AWS Automation Tools

Install the Red Hat AWS automation tools:

```bash
# Install system dependencies
sudo dnf install -y python3-devel krb5-devel openldap-devel

# Create and activate a Python virtual environment (recommended)
python -m venv ~/.aws-saml-venv
source ~/.aws-saml-venv/bin/activate

# Install aws-automation tools (VPN required)
pip install --upgrade pip
pip install --upgrade git+https://gitlab.cee.redhat.com/compute/aws-automation.git
```

## Authentication Workflow

### Step 1: Obtain Kerberos Ticket

```bash
# Get a fresh Kerberos ticket
kinit your_kerberos_id@REDHAT.COM
# or
kinit your_kerberos_id@IPA.REDHAT.COM

# Enter your Kerberos password when prompted
```

**Note**: If you need to set/reset your Kerberos password, see: https://redhat.service-now.com/help?id=kb_article_view&sysparm_article=KB0000072

### Step 2: Run AWS SAML Authentication

```bash
# Activate the virtualenv if not already active
source ~/.aws-saml-venv/bin/activate

# Run the SAML authentication tool
aws-saml.py
```

You'll be prompted to select an AWS account role:

```
# You'll see a list of AWS accounts you have access to
# Select the appropriate account number for your role (admin or poweruser)

Please choose the role you would like to assume:
[0]: <account-role-1>
[1]: <account-role-2>
...

Selection: 0

-------------------------------------------------------------
 Your new access key pair has been stored in the AWS credentials
 file /home/your_kerberos_id/.aws/credentials under the "saml" profile.

 Note that it will expire at <timestamp>.

 To use this credential, call the AWS CLI with the --profile option
 (e.g. aws --profile "saml" ec2 describe-instances)
-------------------------------------------------------------
```

### Step 3: Use the Credentials

The credentials are stored under the `saml` profile. Use them in one of two ways:

**Option 1: Environment Variable (recommended for scripts)**
```bash
export AWS_PROFILE=saml
aws ec2 describe-instances
openshift-install create cluster
```

**Option 2: CLI Flag**
```bash
aws --profile saml ec2 describe-instances
```

## Using with OpenShift Installer

For cluster creation, set the `AWS_PROFILE` environment variable before running the installer:

```bash
# Set the profile
export AWS_PROFILE=saml

# Verify credentials are working
aws sts get-caller-identity

# Run cluster setup
${CLAUDE_PLUGIN_ROOT}/bin/setup.sh \
  --cluster-name my-cluster \
  --cloud aws \
  --pull-secret ~/openshift/pull-secret \
  --instance-type m6i.xlarge
```

**Note**: For installations with AWS SSO and restricted IAM roles, see the [AWS SSO Installation Guide](aws-sso-installation.md) for additional requirements.

## Credential Expiration

SAML credentials are temporary and expire after ~12 hours. When they expire:

1. Run `kinit` to refresh your Kerberos ticket (if needed)
2. Run `aws-saml.py` again to get fresh credentials
3. Continue working with `AWS_PROFILE=saml`

### Problem: Credentials Expiring During Long Operations

AWS SSO session credentials can expire during long-running operations like OpenShift cluster installation (30-45 minutes), causing installations to fail mid-process with authentication errors:

```
level=fatal msg=failed to destroy cluster: failed to describe subnets by tags: 
operation error EC2: DescribeSubnets, https response error StatusCode: 401, 
api error AuthFailure: AWS was not able to validate the provided access credentials
```

**Solution**: For long-running operations, use an IAM user with long-lived credentials instead of temporary SSO credentials. See the "IAM User for Long-Running Operations" section below.

## Troubleshooting

### Command not found: aws-saml.py

**Solution**: Activate the virtualenv where you installed the tools:
```bash
source ~/.aws-saml-venv/bin/activate
```

### VPN Required Error

**Solution**: Connect to the Red Hat VPN before running `pip install` or `aws-saml.py`

### Invalid Kerberos Ticket

**Solution**: Get a fresh ticket:
```bash
kdestroy  # Clear old tickets
kinit your_kerberos_id@REDHAT.COM
```

### Expired Credentials

**Symptom**: AWS CLI returns "ExpiredToken" errors

**Solution**: Re-run `aws-saml.py` to refresh credentials

## IAM User for Long-Running Operations

For operations that take longer than the SSO credential expiration window (cluster installation: 30-45 min, teardown: 10-20 min), create an IAM user with long-lived access keys.

### Step 1: Create IAM User

**Via AWS CLI (after SSO authentication):**

```bash
# First, authenticate with AWS SSO
export AWS_PROFILE=saml
aws-saml.py

# Create the IAM user
aws iam create-user --user-name $(whoami)-installer

# Create access key and save the output
aws iam create-access-key --user-name $(whoami)-installer
```

**Via AWS Console:**

1. Log into AWS Console via SSO
2. Navigate to **IAM** → **Users** → **Create user**
3. User name: `<your-kerberos-id>-installer` (e.g., `pehunt-installer`)
4. AWS credential type: **Access key - Programmatic access** only
5. Click **Next**

Save the `AccessKeyId` and `SecretAccessKey` from the output.

### Step 2: Grant Permissions

```bash
# Attach AdministratorAccess policy
aws iam attach-user-policy \
  --user-name $(whoami)-installer \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

**Note**: For production, use a more restrictive policy. See [OpenShift AWS IAM requirements](https://docs.openshift.com/container-platform/4.21/installing/installing_aws/installing-aws-account.html#installation-aws-permissions_installing-aws-account).

### Step 3: Configure Local Credentials

```bash
# Add credentials to ~/.aws/credentials
cat >> ~/.aws/credentials <<EOF

[$(whoami)-installer]
aws_access_key_id = AKIA4366G3NT63UPCJVC
aws_secret_access_key = YOUR_SECRET_ACCESS_KEY_HERE
EOF
```

### Step 4: Use for Cluster Operations

```bash
# Set profile to IAM user
export AWS_PROFILE=$(whoami)-installer

# Verify credentials
aws sts get-caller-identity
# Should show: "Arn": "arn:aws:iam::884692409191:user/pehunt-installer"

# Run cluster setup (30-45 minutes - won't expire)
setup.sh --cluster-name my-cluster --cloud aws --instance-type m6i.xlarge

# Run cluster teardown (10-20 minutes - won't expire)
teardown.sh --cluster-name my-cluster
```

### When to Use Each Method

**Use AWS SSO (`aws-saml.py`):**
- Quick AWS CLI operations (<15 minutes)
- Interactive development work
- Exploring AWS resources
- Security-conscious workflows (credentials auto-expire)

**Use IAM User:**
- ✅ **OpenShift cluster installation** (30-45 minutes)
- ✅ **Cluster teardown** (10-20 minutes)
- ✅ CI/CD pipelines
- ✅ When SSO credentials keep expiring mid-operation

### Security: Rotate and Cleanup

**Rotate credentials regularly:**

```bash
# Create new key
aws iam create-access-key --user-name $(whoami)-installer

# Update ~/.aws/credentials with new key

# Delete old key (after testing new one)
aws iam delete-access-key \
  --user-name $(whoami)-installer \
  --access-key-id AKIA_OLD_KEY_HERE
```

**Delete IAM user when done:**

```bash
# Delete all access keys
for key in $(aws iam list-access-keys --user-name $(whoami)-installer \
  --query 'AccessKeyMetadata[].AccessKeyId' --output text); do
  aws iam delete-access-key --user-name $(whoami)-installer --access-key-id $key
done

# Detach policies
aws iam detach-user-policy \
  --user-name $(whoami)-installer \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Delete user
aws iam delete-user --user-name $(whoami)-installer
```

## References

- Full AWS SSO Documentation: https://source.redhat.com/departments/it/devit/it-infrastructure/itcloudservices/itpubliccloudpage/cloud/docs/consumer/using_ansible_and_the_cli_to_access_aws_in_a_saml_world
- AWS SSO Admin Guide: https://docs.google.com/document/d/1KoJtwzzcSDuMBhpKmdSk0a2YT0c9zuGMK2zGpTRcrAk/view
- AWS SSO User Guide: https://docs.google.com/document/d/1yziT4KU2BhreGP7r1c9LySoW_MseQ5dDqpp6Cm3-18M/view
- AWS IAM User Guide: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html
- OpenShift AWS Permissions: https://docs.openshift.com/container-platform/4.21/installing/installing_aws/installing-aws-account.html#installation-aws-permissions_installing-aws-account
