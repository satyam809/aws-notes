# Deploy Next.js on AWS EC2

This guide deploys Next.js to Ubuntu 24.04 using GitHub Actions, PM2, and Nginx. The app listens privately on `127.0.0.1:3000`; Nginx handles public HTTP/HTTPS traffic.

## Example values

| Name | Value |
|---|---|
| EC2 public IP | `YOUR_EC2_PUBLIC_IP` |
| Application directory | `/home/ubuntu/apps/itinerary-management-frontend` |
| Repository | `https://github.com/satyam809/itinerary-management-frontend.git` |
| PM2 process | `itinerary-management-frontend` |

## 1. Create the EC2 server

Launch Ubuntu 24.04 and configure this security group:

| Type | Port | Source |
|---|---:|---|
| SSH | 22 | Your administrator IP or deployment runner |
| HTTP | 80 | `0.0.0.0/0` |
| HTTPS | 443 | `0.0.0.0/0` |

Do not expose port `3000`.

```powershell
ssh -i "C:\path\to\your-admin-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

## 2. Install server software

```bash
sudo apt update
sudo apt install -y git nginx curl ca-certificates
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2
mkdir -p /home/ubuntu/apps
```

Use a Node.js major version supported by the application and GitHub Actions.

## 3. Clone the repository

For a private repository, create a read-only deploy key on EC2:

```bash
ssh-keygen -t ed25519 -a 100 -C "frontend-deploy-key" -f ~/.ssh/github_deploy
cat ~/.ssh/github_deploy.pub
```

Add the public key under **Repository Settings > Deploy keys** with write access disabled, then configure Git:

```bash
nano ~/.ssh/config
```

```text
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_deploy
    IdentitiesOnly yes
```

```bash
chmod 600 ~/.ssh/config
ssh-keyscan github.com >> ~/.ssh/known_hosts
chmod 600 ~/.ssh/known_hosts
cd /home/ubuntu/apps
git clone git@github.com:satyam809/itinerary-management-frontend.git
cd itinerary-management-frontend
```

For a public repository, clone with HTTPS instead:

```bash
cd /home/ubuntu/apps
git clone https://github.com/satyam809/itinerary-management-frontend.git
cd itinerary-management-frontend
```

## 4. Configure deployment SSH

On Windows, create a dedicated key:

```powershell
ssh-keygen -t ed25519 -a 100 -f "$env:USERPROFILE\.ssh\github_actions_ec2" -C "github-actions-ec2"
Get-Content "$env:USERPROFILE\.ssh\github_actions_ec2.pub"
```

On EC2, add the public key as one line in `~/.ssh/authorized_keys`, then secure the file:

```bash
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Test it from Windows:

```powershell
ssh -i "$env:USERPROFILE\.ssh\github_actions_ec2" ubuntu@YOUR_EC2_PUBLIC_IP
```

In GitHub, add these repository **Actions secrets**:

| Secret | Value |
|---|---|
| `EC2_HOST` | EC2 public IP or deployment hostname |
| `EC2_USER` | `ubuntu` |
| `EC2_SSH_KEY` | Complete private deployment key |

Keep port `22` restricted to known administrator addresses or controlled deployment runners; GitHub-hosted runner IPs can change.

## 5. Configure production environment

```bash
nano /home/ubuntu/apps/itinerary-management-frontend/.env.production
chmod 600 /home/ubuntu/apps/itinerary-management-frontend/.env.production
```

Example:

```env
NODE_ENV=production
NEXT_PUBLIC_API_URL=https://api.example.com
```

Use only the variable names required by the application. `NEXT_PUBLIC_*` values are included in the browser bundle, so never put secrets in them. Do not commit environment files.

## 6. Configure Nginx

```bash
sudo nano /etc/nginx/sites-available/itinerary-management-frontend
```

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name YOUR_DOMAIN_OR_IP;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/itinerary-management-frontend /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl enable --now nginx
```

## 7. Start the application

```bash
cd /home/ubuntu/apps/itinerary-management-frontend
npm ci
npm run build
PORT=3000 pm2 start npm --name itinerary-management-frontend -- start
pm2 save
pm2 startup systemd
```

Run the exact `sudo env ... pm2 startup systemd` command printed by PM2, then save and verify:

```bash
pm2 save
pm2 status
curl -I http://127.0.0.1:3000
```

## 8. Add the deployment workflow

Create `.github/workflows/deploy.yml` in the frontend repository:

```yaml
name: Deploy frontend to EC2

on:
  push:
    branches: [main]

permissions:
  contents: read

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

            cd "$APP_DIR"
            git fetch origin main
            git reset --hard origin/main
            npm ci
            npm run build

            if pm2 describe "$APP_NAME" >/dev/null 2>&1; then
              PORT=3000 pm2 restart "$APP_NAME" --update-env
            else
              PORT=3000 pm2 start npm --name "$APP_NAME" -- start
            fi

            sleep 5
            curl --fail http://127.0.0.1:3000 >/dev/null
            pm2 save
            sudo systemctl reload nginx
```

For a private repository, set the workflow server checkout to use SSH instead of an HTTPS clone. Do not place the deploy key in GitHub Actions.

## 9. Deploy and verify

```bash
git add .github/workflows/deploy.yml
git commit -m "Add frontend deployment workflow"
git push origin main
```

After the GitHub Actions run completes, verify on EC2:

```bash
pm2 status
curl -I http://127.0.0.1:3000
curl -I http://YOUR_EC2_PUBLIC_IP
```

## 10. Configure HTTPS

Point an `A` record for the domain to the EC2 public IPv4 address, then:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d YOUR_DOMAIN
sudo certbot renew --dry-run
curl -I https://YOUR_DOMAIN
```

## Troubleshooting

- **502:** Check `pm2 status`, `pm2 logs itinerary-management-frontend`, and port `3000`.
- **Wrong site:** Run `sudo nginx -T` and remove duplicate enabled sites.
- **Build failure:** Verify Node.js versions match and run `npm run build` on EC2.
- **SSH failure:** Verify the public key in `authorized_keys`, `EC2_SSH_KEY`, and security-group access to port 22.

Keep `.env.production` private, do not expose port `3000`, restrict SSH, and keep Node.js, Ubuntu, PM2, and Nginx updated.
