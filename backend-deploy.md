# Deploy NestJS + RDS

> Setup: EC2 runs the backend, RDS runs MySQL, and Nginx is public.

```text
Internet -> Nginx :80/:443 -> NestJS :3001 -> RDS MySQL :3306
```

## 1. EC2

- Ubuntu instance with a public IP
- Security group inbound: `22` (My IP), `80` and `443` (anywhere)
- Do **not** open `3001` or `3306` to the internet.

```bash
ssh -i key.pem ubuntu@EC2_PUBLIC_IP
sudo apt update && sudo apt upgrade -y
sudo apt install -y git nginx curl mysql-client

# Install the Node LTS version required by the projects
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm i -g pm2

mkdir -p ~/apps
```

Check: `node -v`, `npm -v`, `pm2 -v`, `nginx -v`.

## 2. RDS MySQL

Create a MySQL RDS database in the **same VPC** as EC2.

- Public access: **No**
- Create an RDS security group.
- RDS SG inbound rule: MySQL/Aurora `3306` from the **EC2 security group** (not `0.0.0.0/0`).
- Note the endpoint, database name, username, and password.

From EC2, test it before deploying the app:

```bash
mysql -h RDS_ENDPOINT -P 3306 -u DB_USER -p
```

```sql
SHOW DATABASES;
EXIT;
```

## 3. Give EC2 access to private GitHub repositories

On EC2:

```bash
ssh-keygen -t ed25519 -C "ec2-deploy" -f ~/.ssh/github_deploy
cat ~/.ssh/github_deploy.pub
```

Add the displayed public key as a read-only **Deploy key** in the backend GitHub repository.

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
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config ~/.ssh/github_deploy
ssh -T git@github.com
```

## 4. Backend (NestJS)

```bash
cd ~/apps
git clone git@github.com:OWNER/BACKEND.git backend
cd backend
npm ci
nano .env
```

Example `.env` — use the exact variable names your project reads:

```env
NODE_ENV=production
PORT=3001
DATABASE_HOST=RDS_ENDPOINT
DATABASE_PORT=3306
DATABASE_USERNAME=DB_USER
DATABASE_PASSWORD=DB_PASSWORD
DATABASE_NAME=DB_NAME
```

```bash
chmod 600 .env
npm run build
pm2 start dist/main.js --name backend
pm2 logs backend --lines 50
```

Notes:

- The backend must use the RDS endpoint, not `localhost`.
- Keep it private: bind to `127.0.0.1` if the app supports it.
- Run migrations deliberately; do not use destructive schema sync in production.

## 5. Keep PM2 running after reboot

```bash
pm2 startup
# Run the sudo command PM2 prints
pm2 save
pm2 status
```

The `backend` process should show `online`.

## 6. Nginx reverse proxy

```bash
sudo nano /etc/nginx/sites-available/app
```

```nginx
server {
    listen 80;
    server_name YOUR_DOMAIN_OR_EC2_IP;

    location /api/ {
        proxy_pass http://127.0.0.1:3001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/app
sudo nginx -t
sudo systemctl reload nginx
```

Test:

```bash
curl -I http://127.0.0.1:3001/api
curl -I http://EC2_PUBLIC_IP/api
```

`/api/` is passed to NestJS with `/api` preserved. Match it to your Nest global prefix.

## 7. Domain + HTTPS

1. Point a DNS `A` record to the EC2 public IP.
2. Wait until the domain resolves.
3. Run:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d YOUR_DOMAIN
sudo certbot renew --dry-run
```

## 8. CI/CD with GitHub Actions

The workflow below runs checks for every push and pull request, then deploys only a successful push to `main`. It connects to EC2 over SSH; the server keeps the production `.env`, so database credentials never enter GitHub Actions.

### One-time GitHub setup

Create these repository **Actions secrets**:

- `DEPLOY_HOST` — EC2 public IP or DNS name
- `DEPLOY_USER` — normally `ubuntu`
- `DEPLOY_SSH_KEY` — private key whose public half is in `~/.ssh/authorized_keys` for that user
- `SSH_KNOWN_HOSTS` — output of `ssh-keyscan -H EC2_PUBLIC_IP` collected and verified from a trusted network

Use a dedicated deployment key, not the personal SSH key used for administration. On EC2, make sure its public key is added to `~/.ssh/authorized_keys` and that `~/apps/backend` is already cloned as above.

### Create `DEPLOY_SSH_KEY`

On a trusted computer (not in the repository), create a dedicated key pair:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions_deploy
```

Log in to EC2 using the existing administrative key, then add the new **public** key:

```bash
ssh -i key.pem ubuntu@EC2_PUBLIC_IP
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Append the contents of `~/.ssh/github_actions_deploy.pub`, save, then run:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Test the new key from the trusted computer before using it in GitHub:

```bash
ssh -i ~/.ssh/github_actions_deploy ubuntu@EC2_PUBLIC_IP
```

Create the GitHub `DEPLOY_SSH_KEY` secret by pasting the full contents of `~/.ssh/github_actions_deploy` (the private key, including its `BEGIN` and `END` lines). Never commit this file or paste the `.pub` file into that secret.

### Create `SSH_KNOWN_HOSTS`

First obtain EC2's ED25519 host-key fingerprint through the existing, trusted SSH connection:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

On the trusted computer, retrieve the public host key and compare its fingerprint to the value above:

```bash
ssh-keyscan -t ed25519 EC2_PUBLIC_IP | ssh-keygen -lf -
```

Only if the fingerprints match, generate the value for the GitHub `SSH_KNOWN_HOSTS` secret:

```bash
ssh-keyscan -H EC2_PUBLIC_IP
```

Paste all of that command's output into the secret. This prevents a GitHub Actions runner from accepting an unexpected SSH server.

Create `.github/workflows/ci-cd.yml` in the backend repository:

```yaml
name: CI/CD

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: production-deploy
  cancel-in-progress: true

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
    needs: ci
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: webfactory/ssh-agent@v0.9.1
        with:
          ssh-private-key: ${{ secrets.DEPLOY_SSH_KEY }}
      - run: |
          mkdir -p ~/.ssh
          printf '%s\\n' "${{ secrets.SSH_KNOWN_HOSTS }}" > ~/.ssh/known_hosts
          chmod 600 ~/.ssh/known_hosts
      - run: |
          ssh -o StrictHostKeyChecking=yes "${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }}" \
            'cd ~/apps/backend && \
             git fetch origin main && \
             git checkout main && \
             git pull --ff-only origin main && \
             npm ci && \
             npm run build && \
             pm2 reload backend --update-env && \
             pm2 save'
```

If the project needs migrations, add its **non-destructive, production-safe** migration command immediately before `npm run build`; do not guess the command or enable schema synchronization. Protect the `production` environment in GitHub with required reviewers if deployments need approval.

## Deploy updates manually

```bash
# backend
cd ~/apps/backend && git pull --ff-only && npm ci && npm run build && pm2 reload backend

pm2 save
```

## Quick troubleshooting

```bash
pm2 status
pm2 logs backend --lines 100
sudo nginx -t
sudo systemctl status nginx
sudo ss -lntp | grep ':3001'
sudo tail -n 100 /var/log/nginx/error.log
```

## Final check

- [ ] EC2 can connect to RDS.
- [ ] Backend is `online` in PM2.
- [ ] The API is reachable through Nginx.
- [ ] `/api` reaches NestJS.
- [ ] HTTPS works and `certbot renew --dry-run` passes.
