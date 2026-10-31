# Deploy a NestJS Backend and RDS MySQL to AWS EC2

This guide deploys a NestJS API with GitHub Actions:

```text
GitHub push -> GitHub Actions -> Ubuntu EC2 -> NestJS + PM2 -> Nginx -> HTTPS
                                     |
                                     -> Amazon RDS MySQL
```

The API listens privately on `127.0.0.1:3001`. Nginx provides the public HTTP and HTTPS endpoint. Do not expose port `3001` or the RDS MySQL port to the internet.

## Values used in this guide

Replace the example values before deployment. Existing repository documentation uses these names:

| Value | Example |
|---|---|
| Backend directory on EC2 | `/home/ubuntu/apps/itinerary-management-backend` |
| Backend repository | `https://github.com/satyam809/itinerary-management-backend.git` |
| PM2 process name | `backend` |
| Application port | `3001` |
| Nginx configuration | `/etc/nginx/sites-available/itinerary-management-backend` |

Use the actual repository owner, repository name, application directory, PM2 name, and port consistently.

# Part A: One-time AWS and server setup

## 1. Create the EC2 instance

1. Launch an Ubuntu 24.04 LTS EC2 instance in the AWS Region where the application will run.
2. Select a suitable instance size, for example `t3.small` for a small production service.
3. Create an SSH key pair, download the private key, and keep it outside the repository.
4. Create or use a security group with these inbound rules:

| Type | Port | Source |
|---|---:|---|
| SSH | 22 | Your administrator IP |
| HTTP | 80 | `0.0.0.0/0`; add `::/0` only when IPv6 is configured |
| HTTPS | 443 | `0.0.0.0/0`; add `::/0` only when IPv6 is configured |

Do not create inbound rules for port `3001`, unless a load balancer or restricted private health check requires it. Use an Elastic IP or stable DNS name when public reachability must survive instance replacement.

Connect with the administrator key:

```powershell
ssh -i "C:\path\to\your-admin-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

## 2. Install Node.js, PM2, Nginx, and Git

Run on EC2:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git nginx curl ca-certificates mysql-client

curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2

mkdir -p /home/ubuntu/apps
node --version
npm --version
nginx -v
pm2 --version
```

Use Node.js 20 unless the backend's supported engines, lockfile workflow, or CI configuration requires another release. The server and GitHub Actions runner should use the same major Node.js version.

Verify Nginx:

```bash
sudo systemctl enable nginx
sudo systemctl status nginx --no-pager
```

## 3. Create and secure Amazon RDS MySQL

1. In the AWS console, create a MySQL RDS database in the same VPC and subnet structure as EC2.
2. Set public accessibility to **No**.
3. Create a dedicated database, application username, and strong password. Do not use the database master account for the application.
4. Create an RDS security group. Add an inbound MySQL/Aurora rule for port `3306` whose source is the EC2 security group ID. Do not use `0.0.0.0/0`.
5. Attach the RDS security group to the database. Attach the database subnet group to private subnets when the EC2 instance is also in an appropriate private network layout.
6. Record the RDS endpoint, database name, username, and password.

Do not store the database password in this document, GitHub Actions, EC2 user data, or an image. Store it only in the EC2 production environment file.

Test database connectivity from EC2:

```bash
mysql -h YOUR_RDS_ENDPOINT -P 3306 -u YOUR_DB_USER -p
```

At the MySQL prompt:

```sql
SHOW DATABASES;
EXIT;
```

If the command times out, verify DNS, route tables, subnet placement, the RDS security group, and that port `3306` is allowed from the EC2 security group.

## 4. Give EC2 read access to a private GitHub repository

For a public repository, skip to **Step 5**. For a private repository:

```bash
ssh-keygen -t ed25519 -a 100 -C "backend-deploy-key" -f ~/.ssh/github_deploy
chmod 700 ~/.ssh
chmod 600 ~/.ssh/github_deploy
cat ~/.ssh/github_deploy.pub
```

In the backend repository, open **Settings** > **Deploy keys**, add the public key, and keep write access disabled unless the server explicitly needs it.

Configure Git SSH:

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

Secure and test the configuration:

```bash
chmod 600 ~/.ssh/config ~/.ssh/known_hosts 2>/dev/null || true
ssh-keyscan github.com >> ~/.ssh/known_hosts
chmod 600 ~/.ssh/known_hosts
ssh -T git@github.com
```

Clone and enter the repository:

```bash
cd /home/ubuntu/apps
git clone git@github.com:satyam809/itinerary-management-backend.git itinerary-management-backend
cd itinerary-management-backend
```

For a public repository, clone with HTTPS:

```bash
cd /home/ubuntu/apps
git clone https://github.com/satyam809/itinerary-management-backend.git itinerary-management-backend
```

## 5. Create a dedicated deployment key for GitHub Actions

The EC2 administrator key must remain separate from the key used by GitHub Actions.

On Windows PowerShell, create the key:

```powershell
ssh-keygen -t ed25519 -a 100 -f "$env:USERPROFILE\.ssh\github_actions_backend" -C "github-actions-backend"
Get-Content "$env:USERPROFILE\.ssh\github_actions_backend.pub"
```

Connect to EC2 with the administrator key, create its SSH directory if needed, and append the deployment public key:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
ssh-keygen -lf ~/.ssh/authorized_keys
```

Test the dedicated key from Windows:

```powershell
ssh -i "$env:USERPROFILE\.ssh\github_actions_backend" -o IdentitiesOnly=yes ubuntu@YOUR_EC2_PUBLIC_IP
```

GitHub-hosted runners can originate from changing IP ranges. Restrict SSH to trusted addresses or use a self-hosted runner in a controlled network. Do not permanently open port `22` to `0.0.0.0/0`.

## 6. Configure GitHub Actions secrets

In the backend repository, open **Settings** > **Secrets and variables** > **Actions**, then add these repository secrets:

| Secret | Value |
|---|---|
| `DEPLOY_HOST` | EC2 public IPv4 address or trusted deployment DNS name |
| `DEPLOY_USER` | `ubuntu` |
| `DEPLOY_SSH_KEY` | Complete contents of the dedicated private key, including BEGIN and END lines |
| `SSH_KNOWN_HOSTS` | Verified output of `ssh-keyscan -H DEPLOY_HOST` |

On Windows, display the private key for copying:

```powershell
Get-Content -Raw "$env:USERPROFILE\.ssh\github_actions_backend"
```

To verify the server host key first, get its expected fingerprint through the trusted administrator connection:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

From Windows, collect and inspect the matching public key:

```powershell
ssh-keyscan -t ed25519 YOUR_EC2_PUBLIC_IP | ssh-keygen -lf -
```

Only if the fingerprint matches, create the GitHub secret with the output of:

```powershell
ssh-keyscan -H YOUR_EC2_PUBLIC_IP
```

## 7. Configure production environment variables

Create `/home/ubuntu/apps/itinerary-management-backend/.env` on EC2:

```bash
cd /home/ubuntu/apps/itinerary-management-backend
nano .env
chmod 600 .env
```

Example variable names; verify every variable against the actual NestJS configuration modules:

```env
NODE_ENV=production
PORT=3001
DB_HOST=YOUR_RDS_ENDPOINT
DB_PORT=3306
DB_USERNAME=YOUR_DB_USER
DB_PASSWORD=YOUR_DB_PASSWORD
DB_NAME=itinerary_db
TYPEORM_SYNC=false
```

Some projects use `DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_USERNAME`, `DATABASE_PASSWORD`, and `DATABASE_NAME`, while others use a `DATABASE_URL`. Use exactly the names read by the application and never mix the formats inconsistently.

Example URL format only:

```text
mysql://YOUR_DB_USER:YOUR_DB_PASSWORD@YOUR_RDS_ENDPOINT:3306/itinerary_db
```

Never commit `.env`, credentials, private keys, or deployment keys. Add the following entries to `.gitignore` when appropriate:

```gitignore
.env
.env.*
!.env.example
```

Keep TypeORM schema synchronization disabled in production. Use reviewed migrations for schema changes. Run migrations only if the repository documents a safe command such as `npm run migration:run`.

## 8. Install and start NestJS for the first time

Build and start the backend manually only once:

```bash
cd /home/ubuntu/apps/itinerary-management-backend
npm ci
npm run build
PORT=3001 node dist/main.js
```

Stop the test process with `Ctrl+C`, then configure PM2:

```bash
PORT=3001 pm2 start dist/main.js --name backend
pm2 save
pm2 startup systemd
```

Run the exact `sudo env ... pm2 startup systemd` command printed by PM2, then run:

```bash
pm2 save
pm2 status
pm2 logs backend --lines 50
curl -I http://127.0.0.1:3001
```

The process should report `online`, and the curl response must match the actual NestJS route prefix or health endpoint.

## 9. Configure Nginx

Create `/etc/nginx/sites-available/itinerary-management-backend`:

```bash
sudo nano /etc/nginx/sites-available/itinerary-management-backend
```

Configure the API prefix used by the NestJS application. This example assumes a global `/api` prefix and preserves it upstream:

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

If NestJS has no global `/api` prefix, use:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3001/;
}
```

That form changes `/api/example` to `/example` upstream. Use only after confirming the backend route prefix.

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/itinerary-management-backend /etc/nginx/sites-enabled/itinerary-management-backend
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl enable nginx
sudo systemctl reload nginx
```

Verify directly and through Nginx:

```bash
curl -i http://127.0.0.1:3001/api
curl -i http://127.0.0.1/api
```

# Part B: CI/CD deployment

## 10. Add the GitHub Actions workflow

Create `.github/workflows/ci-cd.yml` in the backend repository and set `NODE_VERSION` to the supported Node.js major release:

```yaml
name: CI and deploy backend

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: backend-production
  cancel-in-progress: false

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

      - name: Configure verified SSH host
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
            APP_NAME="backend"
            REPOSITORY="https://github.com/satyam809/itinerary-management-backend.git"

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
              PORT=3001 pm2 restart "$APP_NAME" --update-env
            else
              PORT=3001 pm2 start dist/main.js --name "$APP_NAME"
            fi

            sleep 5
            curl --fail --silent --show-error --max-time 20 http://127.0.0.1:3001/api >/dev/null
            pm2 save
          '
```

The workflow runs lint, tests, and build checks for pull requests and pushes. It deploys only a successful push to `main`. Use an SSH repository URL for private repositories when EC2 is configured with its read-only deploy key.

Add migrations to the deployment only after verifying the repository's migration command. Run a non-destructive migration before restarting PM2, for example when a project actually provides the command:

```text
npm run migration:run
```

Never enable `synchronize: true` or run a destructive database synchronization command in production.

## 11. Deploy and verify

Commit and push the workflow and application changes:

```bash
git add .github/workflows/ci-cd.yml
git commit -m "Add backend deployment workflow"
git push origin main
```

On GitHub, open **Actions** and confirm that CI and deployment complete. Then verify on EC2:

```bash
pm2 status
pm2 logs backend --lines 100
sudo ss -lntp | grep ':3001'
curl -i http://127.0.0.1:3001/api
curl -i http://127.0.0.1/api
curl -i http://YOUR_EC2_PUBLIC_IP/api
```

Use an authenticated health or information endpoint when the root API returns a `404` legitimately. Confirm database connectivity with application logs rather than by opening MySQL publicly.

## 12. Add a domain and HTTPS

1. Create a DNS `A` record from the backend domain to the EC2 public IPv4 address.
2. Update Nginx `server_name` to the domain.
3. Install Certbot:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d YOUR_API_DOMAIN
sudo certbot renew --dry-run
```

4. Test both the certificate and API:

```bash
curl -i https://YOUR_API_DOMAIN/api
sudo nginx -t
```

Update frontend configuration such as `NEXT_PUBLIC_API_URL` to the HTTPS backend origin after DNS and certificates work.

# Verification and troubleshooting

## PM2 and application status

```bash
pm2 status
pm2 logs backend --lines 100
sudo ss -lntp | grep ':3001'
curl -v http://127.0.0.1:3001/api
sudo systemctl status nginx --no-pager
```

## Nginx returns 502 Bad Gateway

```bash
pm2 status
sudo ss -lntp | grep ':3001'
curl -v http://127.0.0.1:3001/api
pm2 logs backend --lines 100
pm2 restart backend --update-env
pm2 save
sudo nginx -t
sudo systemctl reload nginx
```

Inspect errors:

```bash
sudo tail -n 100 /var/log/nginx/error.log
```

## Nginx returns 404 or routes incorrectly

Confirm the actual global prefix in NestJS and compare it with the Nginx location and `proxy_pass`:

```bash
sudo nginx -T | grep -n -A12 'server_name'
sudo nginx -t
```

A trailing slash changes path forwarding. Do not use a trailing slash upstream unless the path-rewriting behavior has been tested.

## RDS connection fails

```bash
mysql -h YOUR_RDS_ENDPOINT -P 3306 -u YOUR_DB_USER -p
pm2 logs backend --lines 100
```

Check database host, port, credentials, database name, and TLS requirements in the application configuration. In AWS, verify that the RDS inbound rule permits source security group `EC2_SECURITY_GROUP_ID`, and that EC2 and RDS route correctly.

## SSH or deployment fails

```bash
ssh -vvv -i "C:\path\to\github_actions_backend" ubuntu@YOUR_EC2_PUBLIC_IP
```

Verify that the public key corresponds to `DEPLOY_SSH_KEY`, then inspect permissions on EC2:

```bash
cat ~/.ssh/authorized_keys
stat -c '%a %U %G %n' ~/.ssh ~/.ssh/authorized_keys
```

Expected permissions are `700` for `~/.ssh` and `600` for `authorized_keys`. Verify `SSH_KNOWN_HOSTS`, the EC2 security group, and runner reachability on port `22`.

## PM2 does not survive reboot

```bash
pm2 startup systemd
pm2 save
sudo systemctl status pm2-ubuntu
pm2 resurrect
```

Run the exact `sudo` command printed by `pm2 startup systemd`; do not copy a stale startup command.

## Manual deployment

```bash
cd /home/ubuntu/apps/itinerary-management-backend
git pull --ff-only origin main
npm ci
npm run build
pm2 restart backend --update-env
pm2 save
```

If the repository requires migrations, run only the verified production migration command before restarting the application.

## Production security checklist

- Restrict SSH to administrators and controlled deployment runners.
- Do not expose port `3001` publicly.
- Keep RDS private, reachable only from the EC2 security group on port `3306`.
- Use a least-privilege database user and disable TypeORM schema synchronization.
- Keep `.env` permissions at `600` and never commit secrets.
- Use separate read-only repository deploy keys and deployment SSH keys.
- Verify SSH host keys in GitHub Actions.
- Protect the `production` GitHub environment and the `main` branch.
- Keep Ubuntu, Node.js, Nginx, Certbot, and npm packages supported and patched.
- Never store AWS administrator credentials or database passwords in GitHub Actions.
