# Deploy a Next.js Frontend to AWS EC2

## Step 1: Create an EC2 server

1. In AWS, create an **Ubuntu EC2 instance**.
2. Create and download an RSA `.pem` key pair. Keep it private.
3. Create these security-group inbound rules:

| Type | Port | Source |
| --- | --- | --- |
| SSH | 22 | My IP |
| HTTP | 80 | Anywhere (`0.0.0.0/0`) |
| HTTPS | 443 | Anywhere (`0.0.0.0/0`) |

4. Copy the instance's public IPv4 address.

## Step 2: Connect to EC2 and install software

1. SSH into the instance with the `.pem` key.
2. Install Nginx, Git, Node.js, and PM2.
3. Confirm the Nginx page opens at `http://YOUR_EC2_PUBLIC_IP`.

## Step 3: Create a deployment user

1. Create a Linux user named `deploy` on EC2.
2. Create `/home/deploy/apps/YOUR_REPOSITORY`.
3. Give this user permission to reload Nginx.

## Step 4: Set up GitHub Actions SSH access

1. Create a new SSH key only for GitHub Actions. Do not reuse the EC2 `.pem` key.
2. Add its public key to `/home/deploy/.ssh/authorized_keys` on EC2.
3. Test that the key can sign in as `deploy`.
4. In GitHub repository secrets, add:

| Secret | Value |
| --- | --- |
| `EC2_HOST` | EC2 public IPv4 address |
| `EC2_USER` | `deploy` |
| `EC2_SSH_KEY` | Private contents of the GitHub Actions SSH key |

## Step 5: Add production environment variables

1. On EC2, create `/home/deploy/apps/YOUR_REPOSITORY/.env.production`.
2. Add all production environment variables and secrets there.
3. Never commit `.env.production` or other secret files to GitHub.

## Step 6: Configure Nginx

1. Create an Nginx site configuration for the app.
2. Configure it to forward port `80` requests to `http://127.0.0.1:3000`.
3. Enable the site, run `sudo nginx -t`, then reload Nginx.

## Step 7: Add the GitHub Actions deployment workflow

1. Create `.github/workflows/deploy.yml` in the project.
2. Configure it to run when code is pushed to `main`.
3. The workflow must:
   - Connect to EC2 using the GitHub secrets.
   - Download the latest code from `main`.
   - Run `npm ci`.
   - Run `npm run build`.
   - Start or restart the app with PM2 on port `3000`.
   - Reload Nginx.
4. Replace every placeholder such as `YOUR_REPOSITORY` and `YOUR_GITHUB_USERNAME`.

## Step 8: Deploy and verify

1. Commit and push the workflow to the `main` branch.
2. Open the **Actions** tab in GitHub and confirm the deployment succeeds.
3. Run PM2 startup configuration once so the app starts after an EC2 reboot.
4. Open `http://YOUR_EC2_PUBLIC_IP` and confirm that the frontend works.

## Important

- Do not open port `3000` in the EC2 security group.
- Do not share or commit private keys, `.pem` files, or `.env.production`.
- For a custom domain, point its DNS `A` record to the EC2 public IP, then add HTTPS with Certbot.
