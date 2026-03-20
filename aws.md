# AWS — GitHub Actions Reference

## OIDC Authentication (preferred over access keys)

Use `aws-actions/configure-aws-credentials` with OIDC so no long-lived secrets are stored in GitHub.

### IAM OIDC Provider (one-time setup)
```hcl
# Terraform — create once per AWS account
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

resource "aws_iam_role" "github_actions" {
  name = "github-actions-deploy"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.github.arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringLike = {
          "token.actions.githubusercontent.com:sub" = "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }]
  })
}
```

### Workflow snippet
```yaml
permissions:
  id-token: write
  contents: read

steps:
  - name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@010d0da01d0b5a38af31e9c3470dbfdabdecca3a  # v4.0.1
    with:
      role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
      aws-region: ${{ vars.AWS_REGION }}
```

---

## ECR — Push Docker image

```yaml
  - name: Login to Amazon ECR
    id: login-ecr
    uses: aws-actions/amazon-ecr-login@062b18b96a7aff071d4dc91bc00c4c1a7945b076  # v2.0.1

  - name: Build, tag, and push image to ECR
    env:
      REGISTRY: ${{ steps.login-ecr.outputs.registry }}
      REPOSITORY: my-app
      IMAGE_TAG: ${{ github.sha }}
    run: |
      docker build -t $REGISTRY/$REPOSITORY:$IMAGE_TAG .
      docker push $REGISTRY/$REPOSITORY:$IMAGE_TAG
      echo "image=$REGISTRY/$REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
```

---

## ECS — Deploy to Fargate

```yaml
  - name: Download task definition
    run: |
      aws ecs describe-task-definition --task-definition my-app \
        --query taskDefinition > task-definition.json

  - name: Update ECS task definition with new image
    id: task-def
    uses: aws-actions/amazon-ecs-render-task-definition@69a2ab71b176c4a5a3a82c5e1b45e38e8abc6d64  # v1.3.0
    with:
      task-definition: task-definition.json
      container-name: my-app
      image: ${{ steps.build.outputs.image }}

  - name: Deploy to ECS
    uses: aws-actions/amazon-ecs-deploy-task-definition@3cc43061dd30ad47b99a6960b6b7a4b564e46b7e  # v1.5.0
    with:
      task-definition: ${{ steps.task-def.outputs.task-definition }}
      service: my-app-service
      cluster: my-cluster
      wait-for-service-stability: true
```

---

## Lambda — Deploy function

```yaml
  - name: Deploy Lambda
    run: |
      zip -r function.zip .
      aws lambda update-function-code \
        --function-name my-function \
        --zip-file fileb://function.zip
```

---

## Terraform state backend (S3 + DynamoDB)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "env/production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}
```

Secrets needed:
| Secret | Value |
|--------|-------|
| `AWS_ROLE_ARN` | ARN of the OIDC role |
| `TF_STATE_BUCKET` | S3 bucket name |
| `TF_LOCK_TABLE` | DynamoDB table name |
