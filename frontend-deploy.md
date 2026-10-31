# Deploy a Next.js Frontend to AWS EC2

This guide deploys a Next.js application through GitHub Actions:

```text
GitHub push -> GitHub Actions SSH -> Ubuntu EC2 -> Next.js + PM2 -> Nginx :80/:443
```

The application listens privately on `127.0.0.1:3000`. Nginx is the public reverse proxy. Port `3000` must not be opened in the EC2 security group.

## Values used in this guide

Replace the example values before deploying. The repository and directory names below match the documentation in this repository:

| Value | Example |
|---|---|
| EC2 public IPv4 | `YOUR_EC2_PUBLIC_IP` |
| Deployment user | `ubuntu` |
| Application directory | `/home/ubuntu/apps/itinerary-management-frontend` |
| Frontend repository | `https://github.com/satyam809/itinerary-management-frontend.git` |
| PM2 process name | `itinerary-management-frontend` |
| Nginx configuration | `/etc/nginx/sites-available/itinerary-management-frontend` |

If the frontend repository is different, update every occurrence consistently.

# Part A: One-time AWS and server setup

## 1. Create the Ubuntu EC2 instance

1. Launch an Ubuntu 24.04 LTS EC2 instance in the AWS Region where the application will run.
2. Select an instance type appropriate for the application, for example `t3.small`.
3. Create an SSH key pair, download the private key, and keep it outside the repository.
4. Configure the instance security group inbound rules:

| Type | Port | Source |
|---|---:|---|
| SSH | 22 | Your administrator IP, initially |
| HTTP | 80 | `0.0.0.0/0`, and `::/0` if IPv6 is enabled |
| HTTPS | 443 | `0.0.0.0/0`, and `::/0` if IPv6 is enabled |

Do not expose port `3000` or database ports in this frontend security group. Restrict SSH after configuring deployment automation.

Connect from Windows PowerShell:

```powershell
ssh -i "C:\path\to\your-admin-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

## 2. Install server software

Run the following commands on EC2:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git nginx curl ca-certificates

curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2

node --version
npm --version
nginx -v
pm2 --version
```

Use Node.js 20 unless the frontend declares another supported Node.js version in `package.json` or CI. The Node.js major version used by builds must be the same on the server and in GitHub Actions.

Create the application directory:

```bash
mkdir -p /home/ubuntu/apps
```

Verify the default Nginx page at `http://YOUR_EC2_PUBLIC_IP`.

## 3. Configure GitHub access from EC2

This step is required for a private repository. For a public repository, skip to **Step 4**.

1. On EC2, create an SSH key used only to read the repository:

```bash
ssh-keygen -t ed25519 -a 100 -C "frontend-deploy-key" -f ~/.ssh/github_deploy
chmod 700 ~/.ssh
chmod 600 ~/.ssh/github_deploy
cat ~/.ssh/github_deploy.pub
```

2. In the frontend repository on GitHub, open **Settings** > **Deploy keys**, add the public key, and grant read-only access.

3. Configure GitHub SSH to use the deployment key:

```bash
nano ~/.ssh/config
```

Add:

```text
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_deploy
    IdentitiesOnly yes
```

Secure the configuration and verify the host and key:

```bash
chmod 600 ~/.ssh/config
ssh-keyscan github.com >> ~/.ssh/known_hosts
chmod 600 ~/.ssh/known_hosts
ssh -T git@github.com
```

4. Clone with SSH:

```bash
cd /home/ubuntu/apps
git clone git@github.com:satyam809/itinerary-management-frontend.git
cd itinerary-management-frontend
```

Use the repository's actual owner and name everywhere. For a public repository, clone over HTTPS:

```bash
cd /home/ubuntu/apps
git clone https://github.com/satyam809/itinerary-management-frontend.git
cd itinerary-management-frontend
```

## 4. Create SSH access for GitHub Actions

Use a separate deployment key rather than the administrator `.pem` key. Create it from Windows PowerShell:

```powershell
ssh-keygen -t ed25519 -a 100 -f "$env:USERPROFILE\.ssh\github_actions_ec2" -C "github-actions-ec2"
Get-Content "$env:USERPROFILE\.ssh\github_actions_ec2.pub"
```

On EC2, add the complete public key as a new line in the `ubuntu` user's authorized keys file:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
ssh-keygen -lf ~/.ssh/authorized_keys
```

Test the key locally before storing the private key in GitHub:

```powershell
ssh -i "$env:USERPROFILE\.ssh\github_actions_ec2" -o IdentitiesOnly=yes ubuntu@YOUR_EC2_PUBLIC_IP
```

Add these repository **Actions secrets** under **Settings** > **Secrets and variables** > **Actions**:

| Secret | Value |
|---|---|
| `EC2_HOST` | EC2 public IPv4 address |
| `EC2_USER` | `ubuntu` |
| `EC2_SSH_KEY` | Complete private key, including BEGIN and END lines |

Read the private key in PowerShell with:

```powershell
Get-Content -Raw "$env:USERPROFILE\.ssh\github_actions_ec2"
```

GitHub-hosted runners use changing public IP addresses. Restricting SSH to your workstation will block them. Use a self-hosted Actions runner on a controlled network, allow only documented GitHub Actions ranges, or temporarily use a tightly managed runner-IP rule. Do not open SSH to `0.0.0.0/0`. Monitor and restore the restriction after deployment troubleshooting.

## 5. Configure production environment variables

Create `/home/ubuntu/apps/itinerary-management-frontend/.env.production` only on EC2:

```bash
cd /home/ubuntu/apps/itinerary-management-frontend
nano .env.production
chmod 600 .env.production
```

Use the exact variable names required by the frontend. Public browser variables must start with `NEXT_PUBLIC_` and are compiled into the browser bundle during the build. Never put passwords or tokens in `NEXT_PUBLIC_*` variables.

Example only:

```env
NODE_ENV=production
NEXT_PUBLIC_API_URL=https://api.example.com
```

Add server-only variables to the same file, but review the frontend source to identify every required name. Do not commit `.env`, `.env.production`, private keys, or credentials. Add ignored secret files to `.gitignore`:

```gitignore
.env
.env.local
.env.production
.env.*.local
```

## 6. Configure Nginx

Create `/etc/nginx/sites-available/itinerary-management-frontend`:

```bash
sudo nano /etc/nginx/sites-available/itinerary-management-frontend
```

Replace `YOUR_DOMAIN_OR_IP` with the domain or IP that serves the application:

```nginx
server {
    listen 80;
    listen [::]:80;
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

Enable the site and remove the conflicting default site:

```bash
sudo ln -s /etc/nginx/sites-available/itinerary-management-frontend /etc/nginx/sites-enabled/itinerary-management-frontend
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl enable nginx
sudo systemctl reload nginx
```

For IP-based testing, set the listening `server_name` to `_` or the EC2 address. A domain is recommended for TLS.

## 7. Configure PM2 startup

Build and start the application once:

```bash
cd /home/ubuntu/apps/itinerary-management-frontend
npm ci
npm run build
PORT=3000 pm2 start npm --name itinerary-management-frontend -- start
pm2 save
pm2 startup systemd
```

Run the exact `sudo env ... pm2 startup systemd` command printed by PM2. Then save and verify:

```bash
pm2 save
pm2 status
curl -I http://127.0.0.1:3000
```

The frontend must be `online`. If the project's Next.js version documents webpack builds, use `npm run build -- --webpack` consistently in this guide and workflow only after confirming the installed Next.js CLI supports that option.

# Part B: Automated deployment

## 8. Add the GitHub Actions workflow

Create `.github/workflows/deploy.yml` in the frontend repository, then add:

```yaml
name: Deploy frontend to EC2

on:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: frontend-production
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 20

    steps:
      - name: Deploy over SSH
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script_stop: true
          script: |
            set -eu

            APP_DIR="/home/${{ secrets.EC2_USER }}/apps/itinerary-management-frontend"
            APP_NAME="itinerary-management-frontend"
            REPOSITORY="https://github.com/satyam809/itinerary-management-frontend.git"

            mkdir -p "$(dirname "$APP_DIR")"

            if [ -d "$APP_DIR/.git" ]; then
              cd "$APP_DIR"
              git fetch origin main
              git reset --hard origin/main
            else
              if [ -d "$APP_DIR" ]; then
                mv "$APP_DIR" "$APP_DIR.local-backup"
              fi
              git clone "$REPOSITORY" "$APP_DIR"
              cd "$APP_DIR"
              if [ -d "$APP_DIR.local-backup" ]; then
                for file in "$APP_DIR.local-backup"/.env*; do
                  [ -f "$file" ] && cp "$file" "$APP_DIR/"
                done
                rm -rf "$APP_DIR.local-backup"
              fi
            fi

            npm ci
            npm run build

            if pm2 describe "$APP_NAME" >/dev/null 2>&1; then
              PORT=3000 pm2 restart "$APP_NAME" --update-env
            else
              PORT=3000 pm2 start npm --name "$APP_NAME" -- start
            fi

            sleep 5
            curl --fail --silent --show-error --max-time 20 http://127.0.0.1:3000 >/dev/null
            pm2 save
            sudo systemctl reload nginx
```

Make the repository URL an SSH URL for private repositories when the EC2 deploy key is configured. Do not put the private deploy key in GitHub Actions.

The workflow works as follows:

- Deployments run only for pushes to `main`.
- Concurrent deployments are serialized and an active deployment is not canceled.
- The server pulls the commit directly, so GitHub does not upload build artifacts.
- Existing `.env*` files remain on the server during initial setup.
- Dependencies are installed exactly from `package-lock.json` with `npm ci`.
- The health check runs before Nginx reloads.

## 9. Deploy and verify

Commit and push the workflow:

```bash
git add .github/workflows/deploy.yml
git commit -m "Add production deployment workflow"
git push origin main
```

In the frontend repository, open **Actions** and confirm the run completes successfully. On EC2, run:

```bash
pm2 status
pm2 logs itinerary-management-frontend --lines 50
sudo ss -lntp | grep ':3000'
curl -I http://127.0.0.1:3000
curl -I http://127.0.0.1
curl -I http://YOUR_EC2_PUBLIC_IP
```

Open `http://YOUR_EC2_PUBLIC_IP` in a browser and test a client-side route and server-rendered route if present.

## 10. Add a domain and HTTPS

1. Create a DNS `A` record from the domain to the EC2 public IPv4 address. Use IPv6 only when EC2 networking and security rules support it.
2. Update `server_name` in the Nginx configuration to include the domain.
3. Install and run Certbot:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d YOUR_DOMAIN
sudo certbot renew --dry-run
```

4. Confirm HTTPS works and the certificate renews:

```bash
curl -I https://YOUR_DOMAIN
sudo nginx -t
```

Use an Elastic IP to keep the public address stable, and update DNS if the instance address changes.

# Verification and troubleshooting

## Process and application checks

```bash
pm2 status
pm2 logs itinerary-management-frontend --lines 100
sudo ss -lntp | grep -E ':3000|:80|:443'
curl -I http://127.0.0.1:3000
```

## Nginx returns 502 Bad Gateway

1. Confirm PM2 and port `3000`:

```bash
pm2 status
sudo ss -lntp | grep ':3000'
curl -v http://127.0.0.1:3000
```

2. Inspect application errors and restart:

```bash
pm2 logs itinerary-management-frontend --lines 100
pm2 restart itinerary-management-frontend --update-env
pm2 save
```

3. Validate and reload Nginx:

```bash
sudo nginx -t
sudo nginx -T | grep -A20 'server_name'
sudo systemctl reload nginx
```

## Nginx returns 403 or the wrong site

List enabled configurations and remove duplicate sites that use the same domain:

```bash
sudo ls -la /etc/nginx/sites-enabled
sudo nginx -T | grep -n 'server_name'
sudo nginx -t
```

## The production build fails

```bash
cd /home/ubuntu/apps/itinerary-management-frontend
rm -rf .next
npm run build
```

Only if the installed Next.js version supports it, try webpack explicitly:

```bash
npm run build -- --webpack
```

Before starting Next.js, confirm that the build output exists:

```bash
ls -la .next
```

If the instance runs out of memory, prefer enough production memory or temporary swap. Verify that GitHub Actions and EC2 use the same supported Node major version.

## Deployment permissions or SSH fails

```bash
ssh -vvv -i "C:\path\to\github_actions_ec2" -o IdentitiesOnly=yes ubuntu@YOUR_EC2_PUBLIC_IP
```

Confirm the public key belongs to `EC2_SSH_KEY`, then inspect permissions:

```bash
cat ~/.ssh/authorized_keys
stat -c '%a %U %G %n' ~/.ssh ~/.ssh/authorized_keys
```

Expected permissions are `700` for `~/.ssh` and `600` for `authorized_keys`. Inspect the EC2 security group and verify that runner traffic can reach port `22`.

## PM2 does not survive reboot

```bash
pm2 startup systemd
pm2 save
sudo systemctl status pm2-ubuntu
pm2 resurrect
```

Run the exact `sudo` command printed by `pm2 startup systemd`; do not copy a stale startup command.

## Security checklist before production

- Keep SSH restricted to known administrator addresses and deployment runners.
- Do not expose port `3000` publicly or commit private keys.
- Use repository Deploy Keys instead of personal access tokens for private repositories.
- Keep `.env.production` owned by `ubuntu` with mode `600`.
- Keep Nginx and Node.js packages patched and use supported Node.js releases.
- Protect the `main` branch and the production Actions environment.
- Do not add AWS administrator credentials to GitHub or EC2 for this SSH deployment.
- Test firewall, Nginx, PM2, HTTPS, and application health after infrastructure changes.
