Jenkins + HashiCorp Vault Integration:
======================================

Today's objective: Build a Jenkins pipeline that retrieves application credentials securely from HashiCorp Vault without hardcoding passwords in the Jenkinsfile.

1. Theory – Why integrate Jenkins with Vault? (5 minutes)

* First, understand the problem.

* Imagine your company has a Jenkins CI/CD pipeline that deploys applications to AWS EC2 or EKS.

Your application needs:

      - Database username.
      - Database password.
      - API keys.
      - Application credentials.
* A common but bad practice is storing passwords directly inside Jenkinsfiles.

Example of what NOT to do:

    pipeline {
        environment {
            DB_USER = 'admin'
            DB_PASSWORD = 'Password123'
        }
      }

What is wrong here?

    - Credentials are exposed in source code.
    - Developers may accidentally push passwords to Git.
    - Password rotation becomes difficult.
    - Multiple pipelines may duplicate credentials.

Solution: Jenkins + Vault


<img width="800" height="533" alt="image" src="https://github.com/user-attachments/assets/0ae4a109-d6b6-41b6-85b7-ce6f00601fb4" />

Instead of storing passwords inside Jenkinsfiles:

    1. Store secrets in Vault.
    2. Jenkins authenticates with Vault.
    3. Vault verifies Jenkins identity.
    4. Vault checks the attached policy.
    5. Jenkins retrieves the permitted secrets.
    6. Jenkins uses those secrets during deployment.
Important: Jenkins should receive only the secrets required for its job, not unrestricted Vault access.

<img width="800" height="627" alt="image" src="https://github.com/user-attachments/assets/a827abfc-a523-4561-9f77-e5f8e032ea52" />

3. Configure Jenkins to connect with Vault (5 minutes)

We need two Jenkins plugins:

    1. HashiCorp Vault Plugin – connects Jenkins to Vault and retrieves secrets.
    2. HashiCorp Vault Pipeline Plugin – provides pipeline steps such as withVault.

Install them from:

Manage Jenkins → Plugins → Available plugins

Search for:

    - HashiCorp Vault
    - HashiCorp Vault Pipeline
Official documentation: HashiCorp Vault Jenkins Plugin

<img width="1003" height="752" alt="image" src="https://github.com/user-attachments/assets/dd9a137f-8f4b-42f5-8331-8eeb14e1affa" />

4. Practical – Retrieve secrets inside Jenkinsfile (10 minutes)

Let's assume Vault already contains this KV v2 secret:

Vault mount: devops
Secret path: production/database

username = dbadmin
password = Test@123


* The Vault policy allows Jenkins to read only this production database secret.

Jenkinsfile example



      pipeline {
          agent any
      
          stages {
              stage('Deploy Application') {
                  steps {
                      withVault(
                          configuration: [
                              vaultUrl: 'https://vault.company.com:8200',
                              vaultCredentialId: 'jenkins-vault-aws',
                              engineVersion: 2
                          ],
                          vaultSecrets: [[
                              path: 'devops/production/database',
                              engineVersion: 2,
                              secretValues: [
                                  [
                                      vaultKey: 'username',
                                      envVar: 'DB_USER'
                                  ],
                                  [
                                      vaultKey: 'password',
                                      envVar: 'DB_PASSWORD'
                                  ]
                              ]
                          ]]
                      ) {
                          sh '''
                              echo "Starting application deployment"
      
                              ./deploy.sh
                      
                          '''
                      }
                  }
              }
          }
      }

<img width="797" height="423" alt="image" src="https://github.com/user-attachments/assets/01565245-bc00-4abc-940e-ccd89b5237ba" />

The withVault block makes the requested secrets available to the enclosed pipeline steps. The plugin handles the KV v2 API path internally. 
Jenkins+1


Production warning: Never print DB_PASSWORD using echo, even if Jenkins masks secrets. Masking is not a guarantee against credential exposure. Use the credentials only in the deployment command, and ensure the deployment script doesn't log them.


5. Understand the complete execution flow

![Uploading image.png…]()
      
      Developer pushes code to Git
                |
                v
           Jenkins Pipeline
                |
                v
         Jenkins Agent uses
         AWS IAM credentials
                |
                v
          HashiCorp Vault
                |
                v
          AWS IAM Authentication
                |
                v
          Vault Policy Check
                |
                v
          Retrieve DB Credentials
                |
                v
          Application Deployment

One important security point: Jenkins should retrieve secrets only when required. The credentials should not be stored permanently in the workspace or deployment artifacts.

