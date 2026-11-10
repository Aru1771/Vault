Day 1 – HashiCorp Vault: KV v2 Secrets Management
==================================================

1. Theory (5 minutes)
What is HashiCorp Vault?

        HashiCorp Vault is a secrets management tool.

It securely stores and controls access to sensitive information such as:

      - Database usernames and passwords.
      - AWS access credentials.
      - API keys.
      - Application secrets.
      - TLS certificates.
      
Real-world DevOps problem

* Imagine you have an application running on AWS EKS.

* Your application needs database credentials to connect to an RDS database.

Without Vault, developers might store credentials in:

    - Kubernetes Secret.
    - Jenkins credentials.
    - Configuration files.
    - Environment variables.
* These approaches require careful access control and secret rotation.

With Vault: You store credentials centrally and allow authorized applications to retrieve them.

2. Understand the architecture

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/aae99e3a-c761-4376-8cd3-18733a0d0a47" />

Example:

    1. DevOps engineer stores database credentials in Vault.
    2. Application authenticates with Vault.
    3. Vault checks the application's permissions.
    4. Vault returns the permitted secret.
    5. Application connects to the database.
<img width="1052" height="476" alt="image" src="https://github.com/user-attachments/assets/5e1a435b-e80e-40cc-adc6-ef206a5baa01" />

<img width="997" height="502" alt="image" src="https://github.com/user-attachments/assets/2791e853-3725-4c56-8b2c-cdb377138fb7" />

4. Practical implementation (10 minutes)

First, we will use Vault CLI.

For practice, you need a running Vault server in dev mode. Never use dev mode for production.

Start Vault:

    vault server -dev


In another terminal, configure the environment using the root token printed by Vault:

    export VAULT_ADDR='http://127.0.0.1:8200'
    
    export VAULT_TOKEN='<root-token>'


Enable KV v2:

    vault secrets enable -path=devops kv-v2

Store application credentials:

    vault kv put devops/empower \
    username=admin \
    password=Pass@123

Read the secret:

    vault kv get devops/empower


Update the password:

    vault kv put devops/empower \
    username=admin \
    password=NewPass@456

Check secret versions:

    vault kv metadata get devops/empower


Read the previous version:

    vault kv get -version=1 devops/empower

Notice the path structure:

    devops/empower


- devops = secrets engine mount.
- empower = secret path.
- username and password = secret data.


<img width="601" height="358" alt="image" src="https://github.com/user-attachments/assets/ab544ba7-170f-4585-a974-845b229a8aa4" />

    Vault
     |
     +-- devops/
          |
          +-- empower/
          |     +-- username
          |     +-- password
          |
          +-- payment/
          |     +-- username
          |     +-- password
          |
          +-- employee/
                +-- api_key
Each application can have separate access permissions.

