# Deploy the application to AWS

This guide deploys the Dockerized frontend and NestJS API with this release path:

```text
Push to GitHub → GitHub Actions builds images → Amazon ECR stores images
→ EC2 pulls images → Docker Compose restarts the application
```

Use Amazon RDS for MySQL in production. Do not deploy the development MySQL container with an empty root password, and do not expose phpMyAdmin publicly.

## Before you start

You need:

- A GitHub repository for the project.
- An AWS account and a domain name.
- `app.yourdomain.com` for the frontend and `api.yourdomain.com` for the API.
- A root `.gitignore` that includes `.env`.

Never commit `.env` files, JWT secrets, database credentials, or API keys. The frontend variable `NEXT_PUBLIC_API_URL` is embedded when its Docker image is built, so its production value must be `https://api.yourdomain.com`.

## Step 1: Push the project to GitHub

From the project root, run the following once. Replace the repository URL with your own.

```powershell
git init
git add .
git commit -m "Initial deployment setup"
git branch -M main
git remote add origin https://github.com/YOUR_GITHUB_USER/itinerary.git
git push -u origin main
```

## Step 2: Create the AWS resources

Create these resources in the AWS Region you will deploy to (this guide uses `ap-south-1`):

1. Two private Amazon ECR repositories: `itinerary-backend` and `itinerary-frontend`.
2. An Ubuntu EC2 instance, at least `t3.small`.
3. An Amazon RDS MySQL database.
4. DNS records for `app.yourdomain.com` and `api.yourdomain.com`.
5. An HTTPS reverse proxy: Nginx or Caddy on EC2, or an Application Load Balancer.

Attach an IAM instance role to the EC2 instance with these managed policies:

- `AmazonEC2ContainerRegistryReadOnly`
- `AmazonSSMManagedInstanceCore`

For GitHub Actions, create an IAM role that uses GitHub OIDC. Do not create long-lived AWS access keys. Restrict the role trust policy to your specific GitHub repository and `main` branch. See [AWS IAM OIDC guidance](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html) and [GitHub's AWS OIDC guidance](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws).

## Step 3: Set up the EC2 host

Install Docker, Docker Compose, the AWS CLI, and the SSM Agent on the EC2 instance. Then create the deployment directory:

```bash
sudo mkdir -p /opt/itinerary
sudo chown "$USER":"$USER" /opt/itinerary
cd /opt/itinerary
```

Create `/opt/itinerary/docker-compose.prod.yml`:

```yaml
services:
  backend:
    image: ${ECR_REGISTRY}/itinerary-backend:${IMAGE_TAG}
    restart: unless-stopped
    env_file: .env
    ports:
      - "3001:3001"

  frontend:
    image: ${ECR_REGISTRY}/itinerary-frontend:${IMAGE_TAG}
    restart: unless-stopped
    ports:
      - "3000:3000"
    depends_on:
      - backend
```

Create `/opt/itinerary/.env` on the server only:

```env
PORT=3001
FRONTEND_URL=https://app.yourdomain.com
JWT_SECRET=use-a-long-random-secret
OLLAMA_API_KEY=...
OLLAMA_URL=https://ollama.com/api/generate
OLLAMA_MODEL=gpt-oss:120b
UNSPLASH_ACCESS_KEY=...
DATABASE_URL=mysql://USER:PASSWORD@RDS_ENDPOINT:3306/itinerary_db
TYPEORM_SYNC=false
```

Set `TYPEORM_SYNC=false` in production and apply schema changes with migrations.

## Step 4: Configure GitHub repository secrets

In the repository's **Settings → Secrets and variables → Actions**, add:

| Secret | Value |
| --- | --- |
| `AWS_DEPLOY_ROLE_ARN` | ARN of the GitHub Actions IAM role created in Step 2 |
| `EC2_INSTANCE_ID` | ID of the EC2 instance, for example `i-0123456789abcdef0` |
| `NEXT_PUBLIC_API_URL` | `https://api.yourdomain.com` |

## Step 5: Add the deployment workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to AWS

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

env:
  AWS_REGION: ap-south-1
  BACKEND_REPOSITORY: itinerary-backend
  FRONTEND_REPOSITORY: itinerary-frontend

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push backend
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t "$ECR_REGISTRY/$BACKEND_REPOSITORY:$IMAGE_TAG" ./itinerary-management-backend
          docker push "$ECR_REGISTRY/$BACKEND_REPOSITORY:$IMAGE_TAG"

      - name: Build and push frontend
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
          API_URL: ${{ secrets.NEXT_PUBLIC_API_URL }}
        run: |
          docker build \
            --build-arg NEXT_PUBLIC_API_URL="$API_URL" \
            -t "$ECR_REGISTRY/$FRONTEND_REPOSITORY:$IMAGE_TAG" \
            ./itinerary-management-frontend
          docker push "$ECR_REGISTRY/$FRONTEND_REPOSITORY:$IMAGE_TAG"

      - name: Deploy through AWS Systems Manager
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
          INSTANCE_ID: ${{ secrets.EC2_INSTANCE_ID }}
        run: |
          aws ssm send-command \
            --instance-ids "$INSTANCE_ID" \
            --document-name "AWS-RunShellScript" \
            --parameters "commands=[
              'cd /opt/itinerary',
              'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY',
              'ECR_REGISTRY=$ECR_REGISTRY IMAGE_TAG=$IMAGE_TAG docker compose -f docker-compose.prod.yml pull',
              'ECR_REGISTRY=$ECR_REGISTRY IMAGE_TAG=$IMAGE_TAG docker compose -f docker-compose.prod.yml up -d'
            ]"
```

The workflow runs on Linux. Use Bash continuations (`\`), not PowerShell backticks, in its `run` blocks.

The ECR repositories must exist before the first workflow run. See [Push an image to Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-push-ecr-image.html).

## Step 6: Configure HTTPS routing

Configure Nginx or Caddy to route:

- `https://app.yourdomain.com` to `http://localhost:3000`
- `https://api.yourdomain.com` to `http://localhost:3001`

Ensure these values match your domains:

```env
FRONTEND_URL=https://app.yourdomain.com
NEXT_PUBLIC_API_URL=https://api.yourdomain.com
```

The completed request path is:

```text
Route 53/DNS → HTTPS proxy or load balancer → Docker frontend and API → RDS MySQL
```

## Step 7: Deploy

Commit and push the workflow. Each push to `main` builds new images tagged with the commit SHA, pushes them to ECR, and tells EC2 to pull and start that version.

```powershell
git add .github/workflows/deploy.yml
git commit -m "Add AWS deployment workflow"
git push
```

Open the GitHub Actions run to verify the deployment. If it fails, check the command output in the workflow first, then inspect the EC2 service logs:

```bash
cd /opt/itinerary
docker compose -f docker-compose.prod.yml logs --follow
```
