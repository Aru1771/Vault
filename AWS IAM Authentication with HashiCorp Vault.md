Theory – What is AWS IAM authentication? (5 minutes):
----------------------------------------------------

First, understand the problem.

Imagine you have:

    - An EC2 instance running an application.
    - A Vault server storing database credentials.
    - An IAM role attached to the EC2 instance.
Your application needs to retrieve database credentials from Vault.

Question: How will Vault identify whether this EC2 instance is authorized?

This is where AWS IAM authentication comes in.

AWS IAM authentication allows an AWS workload to prove its identity to Vault using AWS-signed requests.

It avoids storing a permanent Vault token inside your application.

Understand the architecture:
----------------------------

<img width="1050" height="594" alt="image" src="https://github.com/user-attachments/assets/4d283922-58ee-4b7c-8739-9a1865219837" />

Let's understand the flow:

  The flow:

    1. EC2 obtains temporary AWS credentials from its IAM role.
    2. Application creates an AWS-signed GetCallerIdentity request.
    3. Application sends that signed request to Vault.
    4. Vault validates the request with AWS STS.
    5. Vault identifies the AWS IAM principal.
    6. Vault checks whether that principal is allowed by its AWS auth role.
    7. Vault issues a Vault token if authentication succeeds.
 Remember this distinction:
    
    -  AWS IAM identifies WHO you are.
    - Vault policies determine WHAT you can access.
             
Important: AWS IAM authentication does not directly give the application access to every secret. Vault policies control what the issued token can access.

Important concepts (5 minutes):
-------------------------------

There are three components you must understand.

A. AWS IAM Role

An IAM role gives temporary AWS credentials to an EC2 instance or another AWS workload.

Example:
EC2 Instance
    |
    | IAM Instance Profile
    |
    v
DevOps-EC2-Role
    |
    v
Temporary AWS credentials


B. AWS STS – GetCallerIdentity

AWS Security Token Service (STS) provides temporary security credentials and identity-related operations.

The GetCallerIdentity API tells AWS which IAM identity is making the request.

Example:
aws sts get-caller-identity


Expected output:
{
  "UserId": "AROAXXXXX:i-123456",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/DevOps-EC2-Role/i-123456"
}


This helps identify the AWS role being used.

C. Vault AWS Auth Role

Inside Vault, we configure an AWS auth role.

For example:
Vault AWS Auth Role: ec2-devops

Allowed IAM Role:
DevOps-EC2-Role

Vault Policy:
application-read-policy


This means only an AWS identity matching the configured restrictions can authenticate through this Vault role.

<img width="752" height="392" alt="image" src="https://github.com/user-attachments/assets/e9c1ee43-4d83-4470-bec0-5511324f4949" />

Step 1: Enable AWS authentication:

* Run this on a machine with an authorized Vault administrative token:

      vault auth enable aws

Verify:

    vault auth list

Expected mount:

    aws/    aws

Step 2: Configure Vault AWS client credentials

Vault needs AWS credentials to validate signed requests with AWS STS.

For a production Vault server running on EC2, prefer an appropriate IAM instance profile rather than static AWS access keys.

For example, configure the AWS auth client to use its available AWS credentials:

    vault write auth/aws/config/client

The Vault server's AWS identity must have the permissions required by the configured AWS authentication validation method.

Step 3: Create an AWS authentication role

Assume the application EC2 instance uses:

    DevOps-EC2-Role


Configure the Vault role:

    vault write auth/aws/role/ec2-devops \
        auth_type=iam \
        bound_iam_principal_arn="arn:aws:iam::123456789012:role/DevOps-EC2-Role" \
        policies="application-read-policy" \
        max_ttl=1h

<img width="820" height="353" alt="image" src="https://github.com/user-attachments/assets/ec214344-0348-4088-947c-31d90d8d3f77" />

Step 4: Authenticate from the application EC2 instance

First, check the EC2 IAM identity:

    aws sts get-caller-identity


Then authenticate with Vault:

    vault login -method=aws role=ec2-devops


Vault uses the available AWS credentials to create a signed authentication request.

If authentication succeeds, Vault returns a Vault token.

Step 5: Verify the Vault token

vault token lookup

You can inspect:
    
    - Token policies.
    - Token TTL.
    - Token creation information.
The application can now use this token to access the secrets allowed by its Vault policies.

Production troubleshooting (5 minutes)

Consider this scenario:

Your EC2 instance has the IAM role:

DevOps-EC2-Role


But Vault has been configured with:

App-EC2-Role


The application runs:
vault login -method=aws role=ec2-devops

<img width="946" height="482" alt="image" src="https://github.com/user-attachments/assets/bd93bd60-32b3-41d7-8187-a253cb2e3f01" />



Authentication fails because the AWS identity does not satisfy the Vault role's configured principal restrictions.
