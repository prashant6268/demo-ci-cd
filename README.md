Step 1: Create an ECR Repository
Open AWS Console.
Go to Amazon ECR.
Click Create Repository.

Repository Name:

demo
Click Create Repository.

This repository will store your Docker images.

Step 2: Create an OIDC Provider
Open AWS Console.
Go to IAM → Identity Providers.
Click Add Provider.

Enter:

Provider Type: OpenID Connect

Provider URL:

https://token.actions.githubusercontent.com

Audience:

sts.amazonaws.com
Click Add Provider.

This allows GitHub Actions to authenticate with AWS securely.

Step 3: Create an IAM Role
Go to IAM → Roles.
Click Create Role.
Choose:
Trusted Entity Type: Web Identity
Identity Provider: token.actions.githubusercontent.com
Audience: sts.amazonaws.com
Click Next.

Role Name:

GitHubActionsRole
Create the role.
Step 4: Configure Trust Policy

Open:

IAM → Roles → GitHubActionsRole → Trust Relationships

Replace the trust policy with:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::738759745415:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:prashant6268/demo-ci-cd:ref:refs/heads/main"
        }
      }
    }
  ]
}

Replace demo-ci-cd with your actual repository name if different.

Step 5: Add Permissions to the IAM Role

Open:

IAM → Roles → GitHubActionsRole → Add Permissions → Create Inline Policy

Choose JSON and paste:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:RegisterTaskDefinition",
        "ecs:DescribeTaskDefinition",
        "ecs:DescribeServices",
        "ecs:DescribeClusters",
        "ecs:UpdateService"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}

Policy Name:

GitHubECSDeployPolicy
Step 6: Create ECS Cluster
Open Amazon ECS.
Click Clusters.
Click Create Cluster.

Cluster Name:

Cluster
Create the cluster.
Step 7: Create ECS Task Definition
Go to Task Definitions.
Click Create New Task Definition.
Select Fargate.

Task Definition Family:

task

Container Name:

Main

Container Image:

Temporary image URI from ECR
Save.
Step 8: Create ECS Service
Open your ECS Cluster.
Click Create Service.

Service Name:

task-service

Select Task Definition:

task

Desired Tasks:

1
Create Service.
Step 9: Configure GitHub Secrets

Open:

GitHub Repository → Settings → Secrets and Variables → Actions

Create these secrets:

Secret Name	Value
AWS_REGION	us-east-1
ECR_REPOSITORY	demo
ECS_CLUSTER	Cluster
ECS_SERVICE	task-service
ECS_TASK_DEFINITION	task
CONTAINER_NAME	Main

Do not create AWS access key secrets when using OIDC.

Step 10: Add GitHub Workflow

Create:

.github/workflows/aws.yml

Make sure the workflow contains:

permissions:
  id-token: write
  contents: read

and:

- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::738759745415:role/GitHubActionsRole
    aws-region: us-east-1
Step 11: Push Code

Commit and push to the main branch.

GitHub Actions will:

Authenticate to AWS using OIDC.
Build the Docker image.
Push the image to ECR.
Create a new ECS task revision.
Deploy the new version to ECS.
Step 12: Verify Deployment

Check:

GitHub → Actions → Workflow status
AWS ECS → Cluster → Services
Running Tasks count
Application URL / Load Balancer
Common Error

Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity

Usually caused by:

Incorrect repository name in trust policy
Incorrect IAM role ARN in workflow
Missing OIDC provider
Missing id-token: write permission
Branch mismatch (main vs another branch)

Verify those items first.
