Vault Policies and Tokens:
==========================

Today's objective: Learn how to control which applications, Jenkins pipelines and EC2 instances can access specific secrets inside Vault.

The most important concept today: Authentication tells Vault who you are. Authorization tells Vault what you are allowed to do.

Theory – What are Vault policies? (5 minutes)

Imagine you have three applications running in AWS:

    - Empower application.
    - Payment application.
    - Employee application.
All three applications need database credentials from Vault.

But there is a security problem.

Should the Empower application be allowed to read Payment application credentials?

No.

This is why we use Vault policies.


A Vault policy defines:
-----------------------

    - Which Vault paths an application can access.
    - Whether it can read secrets.
    - Whether it can create or update secrets.
    - Whether it can delete secrets.
Vault policies follow the deny-by-default principle. If you don't explicitly grant permission, access is denied.

Understand the architecture:
----------------------------

<img width="1645" height="1439" alt="image" src="https://github.com/user-attachments/assets/4927fe68-e966-4c8d-8d4e-1c64e3195e50" />

Consider this flow:

    1. EC2 authenticates with Vault using AWS IAM.
    2. Vault verifies the IAM identity.
    3. Vault issues a token with an attached policy.
    4. The application uses that token to request a secret.
    5. Vault checks the policy.
    6. If permission exists, Vault returns the secret. Otherwise, it denies access.
Remember: A valid token does not mean unlimited access.


Vault policy capabilities (5 minutes):
---------------------------------------

Vault policies are written using HCL (HashiCorp Configuration Language).

These are the important capabilities:

<img width="583" height="497" alt="image" src="https://github.com/user-attachments/assets/9e4bbb4f-2b3b-44a8-9514-62190fcb6c1b" />

For an AWS DevOps engineer, remember these three first:
- read
- update
- delete


Practical implementation – Create Vault policies (10 minutes)
-------------------------------------------------------------

Let's use a real-world production scenario.

Assume we have these secrets:

    Vault
      |
      +-- devops/
           |
           +-- production/database
           |
           +-- development/database
           |
           +-- payment/database

Our requirement:

The production application should read only its production database credentials.

Step 1: Create a policy file
-------------------------------

Create a file:

    vi production-read-policy.hcl

Add this policy:

    path "devops/data/production/database" {
      capabilities = ["read"]
    }

Important: Why are we using devops/data/ instead of just devops/?


Because our devops secrets engine uses KV v2.

KV v2 uses:
- devops/data/ for reading and writing secret data.
- devops/metadata/ for secret metadata and listing.


Step 2: Upload the policy to Vault:
------------------------------------

vault policy write production-read-policy production-read-policy.hcl

Verify:

vault policy list

You should see:

    default
    production-read-policy
    root

Step 3: Create a token with this policy

    vault token create \
      -policy=production-read-policy \
      -ttl=1h


Example output:

Key             Value
---             -----
token           hvs.xxxxx
token_duration  1h
token_renewable true
token_policies  [default production-read-policy]


This token has the production-read-policy attached.

Step 4: Test access:
-----------------------

Suppose the production secret exists:

    vault kv get devops/production/database


With the newly created token, this operation should succeed.

But try reading the development secret:
vault kv get devops/development/database


Expected result
Success: production database
Error: permission denied (development database)


The token can read production credentials but cannot read development credentials.

This is called Least Privilege Access.

Vault tokens (5 minutes):
---------------------------

Now let's understand tokens.

A Vault token is like an identity card used by an application to access Vault.

When an application authenticates successfully, Vault issues a token.

Each token can have:

    - Policies.
    - TTL (Time To Live).
    - Renewable status.
    - Expiration.
    - Access permissions.

<img width="1012" height="693" alt="image" src="https://github.com/user-attachments/assets/d7eace31-40e9-4571-99c3-a7953aaf872c" />

<img width="983" height="667" alt="image" src="https://github.com/user-attachments/assets/55a0aa7e-babf-4ee0-9647-610ca9fcab98" />

<img width="1007" height="665" alt="image" src="https://github.com/user-attachments/assets/6d4a0032-1d6b-4ff9-9e1f-799b018c9dbb" />



What is a Lease in HashiCorp Vault?
-------------------------------------
Simple definition: A lease is a time-bound validity period associated with a secret or credential issued by Vault.

Imagine your application needs database credentials.

Your application requests credentials from Vault.

Vault generates:

    Username: app_user
    Password: XXXXXX

Lease ID: database/creds/app/abc123
Lease Duration: 30 minutes
Renewable: true


This means:

- Vault issued database credentials.
- Those credentials have a 30-minute validity period.
- They may be renewable.
- When the lease expires, Vault revokes the dynamic credentials if supported by that secrets engine.

3. Real-world AWS DevOps example
Imagine Jenkins needs temporary database credentials for deployment.

The flow:

    1. Jenkins requests database credentials from Vault.
    2. Vault generates temporary database credentials.
    3. Vault provides a lease ID and TTL.
    4. Jenkins uses the credentials.
    5. Before expiration, Jenkins can renew the lease if permitted.
    6. When the lease expires or is revoked, Vault revokes the generated credentials.
<img width="624" height="340" alt="image" src="https://github.com/user-attachments/assets/9a56aa6e-f4bd-4271-9be3-5c2cae946cb7" />


