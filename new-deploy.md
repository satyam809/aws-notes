Let’s use the easiest beginner setup first:

`Your computer → GitHub → GitHub Action → EC2 server → Docker runs your app`

You do **not** need ECR or complicated AWS permissions for the first deployment.

## What each thing means

- **Git**: saves versions of your code locally.
- **GitHub**: stores your code online.
- **Docker**: runs your frontend, backend, and database in containers.
- **EC2**: a virtual Linux computer rented from AWS.
- **GitHub Actions**: automatically tells your EC2 server to update whenever you push code to GitHub.

## Step 1: Test Docker locally first

From `C:\project\itinerary`:

```powershell
docker compose up --build
```

Then open:

- Frontend: `http://localhost:3000`
- Backend: `http://localhost:3001`

Stop it with:

```powershell
docker compose down
```

Do not move to AWS until this works locally.

## Step 2: Create a GitHub repository

1. Go to [GitHub](https://github.com).
2. Click **New repository**.
3. Name it `itinerary`.
4. Keep it private.
5. Do not add a README, `.gitignore`, or license.
6. Click **Create repository**.

Then in PowerShell, inside `C:\project\itinerary`:

```powershell
git init
git add .
git commit -m "First version"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/itinerary.git
git push -u origin main
```

Replace `YOUR-USERNAME`.

Before using `git add .`, create a root `.gitignore` file containing:

```gitignore
.env
.env.*
!.env.example
```

Your `.env` contains secrets and must never go to GitHub.

## Step 3: Create an AWS EC2 server

1. Sign in to AWS.
2. Open **EC2**.
3. Click **Launch instance**.
4. Name it `itinerary-server`.
5. Choose **Ubuntu Server 24.04 LTS**.
6. Choose `t3.small` for a reasonable starting point.
7. Create and download a key pair named `itinerary-key.pem`.
   - Keep this file safe.
   - Never upload it to GitHub.
8. In Security Group settings, allow:
   - SSH, port `22`, source: **My IP**
   - Custom TCP, port `3000`, source: `0.0.0.0/0`
   - Custom TCP, port `3001`, source: `0.0.0.0/0`

Launch the instance.

AWS can cost money, so stop or terminate unused instances.

## Step 4: Connect to your EC2 server

In AWS EC2, copy the server’s **Public IPv4 address**.

From PowerShell:

```powershell
ssh -i "C:\path\to\itinerary-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

On the server, install Docker and Git:

```bash
sudo apt update
sudo apt install -y git docker.io docker-compose-v2
sudo usermod -aG docker ubuntu
exit
```

Connect again after that:

```powershell
ssh -i "C:\path\to\itinerary-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

## Step 5: Download your code onto EC2

On EC2:

```bash
git clone https://github.com/YOUR-USERNAME/itinerary.git
cd itinerary
```

Create the production environment file:

```bash
nano .env
```

Paste values similar to this:

```env
MYSQL_DATABASE=itinerary_db

PORT=3001
FRONTEND_URL=http://YOUR_EC2_PUBLIC_IP:3000
JWT_SECRET=put-a-long-random-secret-here

OLLAMA_API_KEY=your-key
OLLAMA_URL=https://ollama.com/api/generate
OLLAMA_MODEL=gpt-oss:120b
UNSPLASH_ACCESS_KEY=your-key

NEXT_PUBLIC_API_URL=http://YOUR_EC2_PUBLIC_IP:3001
```

Save Nano:

- Press `Ctrl + O`, then Enter
- Press `Ctrl + X`

Start the application:

```bash
docker compose up --build -d
```

Check it:

```bash
docker compose ps
```

Visit:

```text
http://YOUR_EC2_PUBLIC_IP:3000
```

## Step 6: Create a GitHub Action

This makes deployment automatic whenever you push to `main`.

Create this file in your project:

`.github/workflows/deploy.yml`

```yaml
name: Deploy to EC2

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Deploy application
        env:
          SSH_PRIVATE_KEY: ${{ secrets.EC2_SSH_KEY }}
          EC2_HOST: ${{ secrets.EC2_HOST }}
        run: |
          mkdir -p ~/.ssh
          echo "$SSH_PRIVATE_KEY" > ~/.ssh/ec2_key
          chmod 600 ~/.ssh/ec2_key

          ssh -o StrictHostKeyChecking=no -i ~/.ssh/ec2_key ubuntu@$EC2_HOST << 'EOF'
            cd ~/itinerary
            git pull origin main
            docker compose up --build -d
          EOF
```

## Step 7: Add GitHub secrets

In GitHub:

1. Open your repository.
2. Go to **Settings** → **Secrets and variables** → **Actions**.
3. Click **New repository secret**.

Add these two secrets:

| Secret name | Value |
|---|---|
| `EC2_HOST` | Your EC2 public IPv4 address |
| `EC2_SSH_KEY` | Entire contents of `itinerary-key.pem` |

To view your key file locally:

```powershell
Get-Content "C:\path\to\itinerary-key.pem"
```

Copy everything, including:

```text
-----BEGIN ...-----
...
-----END ...-----
```

## Step 8: Deploy by pushing code

Whenever you change code:

```powershell
git add .
git commit -m "Describe what changed"
git push
```

Then GitHub Actions will:

1. Connect to EC2.
2. Run `git pull`.
3. Rebuild Docker containers.
4. Restart your app.

Check progress in GitHub under the **Actions** tab.

## Later improvements

Once this first version works, improve it with:

- A domain name such as `app.yourdomain.com`
- HTTPS with Nginx or Caddy
- Amazon RDS instead of the MySQL container
- AWS ECR for storing built Docker images
- Removing public access to ports `3001`, `3307`, and `8080`
- Replacing automatic TypeORM schema sync with migrations

For now, focus only on getting the basic EC2 deployment working.