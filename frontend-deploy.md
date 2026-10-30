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

1. Open a new site configuration file:

   ```bash
   sudo nano /etc/nginx/sites-available/itinerary-management-frontend
   ```

2. Add the following configuration. Replace `YOUR_DOMAIN_OR_IP` with the domain name pointing to the EC2 instance, or with its public IPv4 address:

   ```nginx
   server {
       listen 80;
       server_name YOUR_DOMAIN_OR_IP;

       location / {
           proxy_pass http://127.0.0.1:3000;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection "upgrade";
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```

3. Save the file, then enable the site by creating a symbolic link:

   ```bash
   sudo ln -s /etc/nginx/sites-available/itinerary-management-frontend /etc/nginx/sites-enabled/
   ```

4. If the link already exists, update it instead:

   ```bash
   sudo ln -sf /etc/nginx/sites-available/itinerary-management-frontend /etc/nginx/sites-enabled/
   ```

5. Test the Nginx configuration. Fix any reported errors before continuing:

   ```bash
   sudo nginx -t
   ```

6. Reload Nginx to apply the configuration:

   ```bash
   sudo systemctl reload nginx
   ```

7. Confirm that the app is available through Nginx:

   ```text
   http://YOUR_DOMAIN_OR_IP
   ```

   Nginx receives requests on port `80` and forwards them to the frontend running on port `3000`.

## Step 7: Add the GitHub Actions deployment workflow

1. Create the directory `.github/workflows` in the project if it does not already exist.
2. Create `.github/workflows/deploy.yml`.
3. Add the following workflow:

   ```yaml
   name: Deploy Frontend to EC2

   on:
     push:
       branches:
         - main

   jobs:
     deploy:
       runs-on: ubuntu-latest

       steps:
         - name: Deploy to EC2
           uses: appleboy/ssh-action@v1.2.0
           with:
             host: ${{ secrets.EC2_HOST }}
             username: ${{ secrets.EC2_USER }}
             key: ${{ secrets.EC2_SSH_KEY }}
             script_stop: true
             script: |
               APP_DIR=/home/${{ secrets.EC2_USER }}/apps/itinerary-management-frontend

               if [ -d "$APP_DIR/.git" ]; then
                 cd "$APP_DIR"
                 git fetch origin main
                 git reset --hard origin/main
               else
                 mkdir -p /home/${{ secrets.EC2_USER }}/apps
                 BACKUP_DIR="$APP_DIR.local-backup"
                 rm -rf "$BACKUP_DIR"
                 if [ -d "$APP_DIR" ]; then
                   mv "$APP_DIR" "$BACKUP_DIR"
                 fi
                 git clone https://github.com/satyam809/itinerary-management-frontend.git "$APP_DIR"
                 if [ -d "$BACKUP_DIR" ]; then
                   find "$BACKUP_DIR" -maxdepth 1 -type f -name '.env*' -exec cp {} "$APP_DIR/" \;
                   rm -rf "$BACKUP_DIR"
                 fi
                 cd "$APP_DIR"
               fi

               npm ci
               npm run build

               if pm2 describe itinerary-management-frontend >/dev/null 2>&1; then
                 PORT=3000 pm2 restart itinerary-management-frontend --update-env
               else
                 PORT=3000 pm2 start npm --name itinerary-management-frontend -- start
               fi

               pm2 save
               sudo systemctl reload nginx
   ```

   The workflow works in this order:

   - `on.push.branches.main` runs the workflow only when code is pushed to `main`.
   - `appleboy/ssh-action` opens an SSH connection to EC2 using the three secrets created in Step 4.
   - `script_stop: true` stops the remote script when a command fails.
   - If the repository directory already exists, the workflow downloads the latest commits with `git fetch` and resets it to `origin/main`.
   - If the directory does not exist or has no `.git` directory, it preserves `.env*` files, then clones `https://github.com/satyam809/itinerary-management-frontend.git`.
   - `npm ci` installs packages exactly as recorded in `package-lock.json`.
   - `npm run build` creates the production build.
   - PM2 starts the app on port `3000` the first time and restarts it on later deployments.
   - `pm2 save` stores the PM2 process so it can be restored after a reboot.
   - `sudo systemctl reload nginx` applies Nginx changes and resumes proxying requests to the app.

4. A directory that exists but does not contain `.git` is treated as a new checkout. The workflow moves that directory aside, clones the repository, restores files whose names start with `.env`, and removes the backup. This permits the deployment directory to contain `.env.production` or other environment files before the first deployment.

5. Ensure the repository URL, app name, app directory, and username match your server:

   - GitHub repository URL: replace it if your repository is hosted under a different account or has a different name.
   - App directory: `/home/ubuntu/apps/itinerary-management-frontend` is used in this example.
   - PM2 process name: `itinerary-management-frontend` can be changed to a unique application name.
   - Deployment user: the workflow uses `EC2_USER`, currently expected to be `ubuntu`.

6. Commit the workflow and push it to `main`:

   ```bash
   git add .github/workflows/deploy.yml
   git commit -m "Add EC2 deployment workflow"
   git push origin main
   ```

7. Open the repository's **Actions** tab in GitHub. The deployment run should show an SSH connection, dependency installation, production build, PM2 update, and Nginx reload.

## Step 8: Deploy and verify

1. Commit and push the workflow to the `main` branch.
2. Open the **Actions** tab in GitHub and confirm the deployment succeeds.
3. Run PM2 startup configuration once so the app starts after an EC2 reboot.
4. Open `http://YOUR_EC2_PUBLIC_IP` and confirm that the frontend works.

## Important

- Do not open port `3000` in the EC2 security group.
- Do not share or commit private keys, `.pem` files, or `.env.production`.
- For a custom domain, point its DNS `A` record to the EC2 public IP, then add HTTPS with Certbot.
