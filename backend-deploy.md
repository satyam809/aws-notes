# Deploy NestJS and RDS MySQL on AWS EC2

This guide deploys a NestJS API from GitHub to Ubuntu 24.04 using PM2 and Nginx. The API listens privately on `127.0.0.1:3001`.

## Example values

| Name | Value |
|---|---|
| EC2 public IP | `YOUR_EC2_PUBLIC_IP` |
| Application directory | `/home/ubuntu/apps/itinerary-management-backend` |
| Repository | `https://github.com/satyam809/itinerary-management-backend.git` |
| PM2 process | `itinerary-management-backend` |

## 1. Create EC2 and security groups

Launch Ubuntu 24.04 and configure the EC2 security group:

| Type | Port | Source |
|---|---:|---|
| SSH | 22 | Your administrator IP or deployment runner |
| HTTP | 80 | `0.0.0.0/0` |
| HTTPS | 443 | `0.0.0.0/0` |

Do not expose port `3001`.

```powershell
ssh -i "C:\path\to\your-admin-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

## 2. Install server software

```bash
sudo apt update
sudo apt install -y git nginx curl ca-certificates mysql-client
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2
mkdir -p /home/ubuntu/apps
```

Use a Node.js major version supported by the application and GitHub Actions.

## 3. Create the private RDS database

In the AWS console:

1. Create a MySQL RDS instance in the same VPC as EC2 and set public accessibility to **No**.
2. Create a private database and a dedicated application user with a strong password; do not use the master account by the application.
3. Create an RDS security group that allows TCP port `3306` only from the EC2 security group.
4. Attach that security group to RDS and record the endpoint, database name, username, and password.

From EC2, verify private connectivity:

```bash
mysql -h YOUR_RDS_ENDPOINT -P 3306 -u YOUR_DB_USER -p
```

Run `SHOW DATABASES;`, then exit. Database credentials belong only in the EC2 environment file, not in GitHub or this guide.

## 4. Clone the repository

For a private repository, create a read-only deploy key on EC2:

```bash
ssh-keygen -t ed25519 -a 100 -C "backend-deploy-key" -f ~/.ssh/github_deploy
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
git clone git@github.com:satyam809/itinerary-management-backend.git itinerary-management-backend
cd itinerary-management-backend
```

For a public repository, clone with HTTPS instead:

```bash
cd /home/ubuntu/apps
git clone https://github.com/satyam809/itinerary-management-backend.git itinerary-management-backend
cd itinerary-management-backend
```

## 5. Configure deployment SSH

Create a dedicated key on Windows:

```powershell
ssh-keygen -t ed25519 -a 100 -f "$env:USERPROFILE\.ssh\github_actions_backend" -C "github-actions-backend"
Get-Content "$env:USERPROFILE\.ssh\github_actions_backend.pub"
```

On EC2, add the public key as one line in `~/.ssh/authorized_keys` and secure it:

```bash
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Test the key from Windows:

```powershell
ssh -i "$env:USERPROFILE\.ssh\github_actions_backend" ubuntu@YOUR_EC2_PUBLIC_IP
```

In the backend repository, add these **Actions secrets**:

| Secret | Value |
|---|---|
| `DEPLOY_HOST` | EC2 public IP or deployment hostname |
| `DEPLOY_USER` | `ubuntu` |
| `DEPLOY_SSH_KEY` | Complete private deployment key |
| `SSH_KNOWN_HOSTS` | Output of `ssh-keyscan -H YOUR_EC2_PUBLIC_IP` |

Keep SSH restricted because GitHub-hosted runner addresses can change. Do not open port `22` to the internet.

## 6. Configure production environment

```bash
nano /home/ubuntu/apps/itinerary-management-backend/.env
chmod 600 /home/ubuntu/apps/itinerary-management-backend/.env
```

Use the variable names read by the application's configuration modules, for example:

```env
NODE_ENV=production
PORT=3001
DB_HOST=YOUR_RDS_ENDPOINT
DB_PORT=3306
DB_USERNAME=YOUR_DB_USER
DB_PASSWORD=YOUR_DB_PASSWORD
DB_NAME=itinerary_db
TYPEORM_SYNCHRONIZE=false
```

Do not commit `.env`, credentials, or private keys. Keep TypeORM schema synchronization disabled and run only reviewed migrations.

## 7. Start NestJS for the first time

```bash
cd /home/ubuntu/apps/itinerary-management-backend
npm ci
npm run build
PORT=3001 pm2 start dist/main.js --name itinerary-management-backend
pm2 save
pm2 startup systemd
```

Run the exact `sudo env ... pm2 startup systemd` command printed by PM2, then verify:

```bash
pm2 save
pm2 status
curl -I http://127.0.0.1:3001/api
```

## 8. Configure Nginx

```bash
sudo nano /etc/nginx/sites-available/itinerary-management-backend
```

For an application with the global `/api` prefix:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name YOUR_DOMAIN_OR_IP;

    location /api/ {
        proxy_pass http://127.0.0.1:3001/api/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

If NestJS has no `/api` prefix, use `location /api/ { proxy_pass http://127.0.0.1:3001/; }`. Confirm trailing-slash behavior before using this variant.

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/itinerary-management-backend /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl enable --now nginx
curl -I http://127.0.0.1/api
```

## 9. Add the CI/CD workflow

Create `.github/workflows/ci-cd.yml`:

```yaml
name: CI and deploy backend

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run lint --if-present
      - run: npm test --if-present
      - run: npm run build

  deploy:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: ci
    runs-on: ubuntu-latest
    environment: production
    timeout-minutes: 20
    steps:
      - uses: webfactory/ssh-agent@v0.9.1
        with:
          ssh-private-key: ${{ secrets.DEPLOY_SSH_KEY }}

      - name: Configure SSH host
        env:
          SSH_KNOWN_HOSTS: ${{ secrets.SSH_KNOWN_HOSTS }}
        run: |
          mkdir -p ~/.ssh
          printf '%s\n' "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
          chmod 600 ~/.ssh/known_hosts

      - name: Deploy to EC2
        run: |
          ssh -o StrictHostKeyChecking=yes "${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }}" 'set -eu
            APP_DIR="/home/${{ secrets.DEPLOY_USER }}/apps/itinerary-management-backend"
            APP_NAME="itinerary-management-backend"

            cd "$APP_DIR"
            git fetch origin main
            git reset --hard origin/main
            npm ci
            npm run build

            if pm2 describe "$APP_NAME" >/dev/null 2>&1; then
              PORT=3001 pm2 restart "$APP_NAME" --update-env
            else
              PORT=3001 pm2 start dist/main.js --name "$APP_NAME"
            fi

            sleep 5
            curl --fail http://127.0.0.1:3001/api >/dev/null
            pm2 save
          '
```

Run migrations separately with the repository's reviewed, verified migration command before restarting the application. Never enable schema synchronization in production.

## 10. Deploy and verify

```bash
git add .github/workflows/ci-cd.yml
git commit -m "Add backend deployment workflow"
git push origin main
```

After the GitHub Actions run completes, verify on EC2:

```bash
pm2 status
curl -I http://127.0.0.1:3001/api
curl -I http://YOUR_EC2_PUBLIC_IP/api
```

## 11. Configure HTTPS

Point an `A` record for the API domain to the EC2 public IPv4 address, then:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d YOUR_API_DOMAIN
sudo certbot renew --dry-run
curl -I https://YOUR_API_DOMAIN/api
```

Update the frontend API URL to the HTTPS backend origin.

## Troubleshooting

- **502:** Check `pm2 status`, `pm2 logs itinerary-management-backend`, and port `3001`.
- **Database connection failure:** Verify the `.env` values, private route, and that the RDS security group permits TCP `3306` from the EC2 security group.
- **Wrong route:** Check the NestJS global prefix and compare it with the Nginx `location` and `proxy_pass`.
- **SSH failure:** Verify `DEPLOY_SSH_KEY`, `authorized_keys`, `SSH_KNOWN_HOSTS`, and security-group access to port `22`.

Keep RDS private, port `3001` private, `.env` permissions at `600`, SSH restricted, and Ubuntu, Node.js, Nginx, and dependencies updated.
