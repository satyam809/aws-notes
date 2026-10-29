# Deploy a Next.js Frontend to AWS EC2

## Step 1: Create an EC2 server

1. In AWS, create an **Ubuntu EC2 instance**.
2. Create and download an RSA `.pem` key pair. Keep it private.
3. Create these security-group inbound rules:

   - Open **EC2** > **Instances**, select the instance, and open the **Security** tab.
   - Click the linked security-group name, then choose **Edit inbound rules** > **Add rule**.
   - Add the following rules and select **Save rules**:

| Type | Port | Source type | Source |
| --- | --- | --- | --- |
| SSH | 22 | My IP | Your public IP address |
| HTTP | 80 | Anywhere-IPv4 | `0.0.0.0/0` |
| HTTPS | 443 | Anywhere-IPv4 | `0.0.0.0/0` |

4. Copy the instance's public IPv4 address.

## Step 2: Connect to EC2 and install software

1. SSH into the instance with the `.pem` key.
2. Install Nginx, Git, Node.js, and PM2.
3. Confirm the Nginx page opens at `http://YOUR_EC2_PUBLIC_IP`.

## Step 3: Choose the deployment user and app directory

You can use the default `ubuntu` user for a personal or learning project. If you already have `/home/ubuntu/apps`, keep using it; do not move existing apps.

1. Create a separate frontend directory alongside any existing apps:

   ```bash
   mkdir -p /home/ubuntu/apps/YOUR_FRONTEND_REPOSITORY
   ```

   For example: `/home/ubuntu/apps/itinerary-management-frontend`.

2. Set the GitHub Actions secret `EC2_USER` to `ubuntu`.
3. Use `/home/ubuntu/apps/YOUR_FRONTEND_REPOSITORY` in the deployment workflow and Nginx configuration.

## Step 4: Set up GitHub Actions SSH access

This guide uses the existing default `ubuntu` user. The Actions key is separate from the EC2 `.pem` key: the `.pem` key remains for administrator access, while the new key is used only by GitHub Actions deployments.

1. On your local Windows computer, create the GitHub Actions key. Leave the passphrase empty when prompted because GitHub Actions cannot enter it interactively:

   ```powershell
   ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\github_actions_ec2" -C "github-actions-ec2"
   ```

   This creates a private key at `C:\Users\YOUR_WINDOWS_USER\.ssh\github_actions_ec2` and a public key ending in `.pub`.

2. Copy the public key from your Windows computer:

   ```powershell
   Get-Content "$env:USERPROFILE\.ssh\github_actions_ec2.pub"
   ```

3. On EC2, logged in as `ubuntu`, add that complete one-line public key to the `ubuntu` account's authorized keys:

   ```bash
   nano ~/.ssh/authorized_keys
   chmod 700 ~/.ssh
   chmod 600 ~/.ssh/authorized_keys
   ```

   Save the file after pasting the key on a new line. Never put the private key or the EC2 `.pem` key in `authorized_keys`.

4. From your Windows computer, test the new key before adding it to GitHub:

   ```powershell
   ssh -i "$env:USERPROFILE\.ssh\github_actions_ec2" -o IdentitiesOnly=yes ubuntu@YOUR_EC2_PUBLIC_IP
   ```

   Run `whoami` after signing in. It must return `ubuntu`.

5. In GitHub repository secrets, add:

   - Open the GitHub repository, then go to **Settings** > **Secrets and variables** > **Actions**.
   - Select **New repository secret**, enter the secret name and value below, then choose **Add secret**.
   - The workflow will connect as `ubuntu@YOUR_EC2_PUBLIC_IP`.

| Secret | Value |
| --- | --- |
| `EC2_HOST` | EC2 public IPv4 address |
| `EC2_USER` | `ubuntu` |
| `EC2_SSH_KEY` | Private contents of the GitHub Actions SSH key |

   Copy the private-key value with the following command, then paste the complete output (including the `BEGIN` and `END` lines) as the `EC2_SSH_KEY` secret:

   ```powershell
   Get-Content -Raw "$env:USERPROFILE\.ssh\github_actions_ec2"
   ```

6. Ensure the instance security group permits inbound SSH (port `22`) from the GitHub Actions runner. An SSH rule set to **My IP** permits only your own computer and will block GitHub-hosted runners. For a production system, prefer a self-hosted runner or AWS Systems Manager rather than broadly opening SSH access.

## Step 5: Add production environment variables

1. On EC2, create `/home/ubuntu/apps/YOUR_REPOSITORY/.env.production`.
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
