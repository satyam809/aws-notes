# Next.js on AWS EC2 with GitHub Actions

This guide deploys a Next.js app on Ubuntu EC2 without Docker. Nginx receives web requests, PM2 keeps Next.js running, and GitHub Actions deploys every push to `main`.

> Complete one major step at a time. At every **Stop here**, share the requested non-secret output before continuing.

Replace `YOUR_REPOSITORY`, `YOUR_GITHUB_USERNAME`, and `YOUR_EC2_PUBLIC_IP` with your own values everywhere.

## 1. Create the EC2 server

On **your Windows computer**, confirm `package.json` has production scripts:

~~~powershell
Get-Content package.json
~~~

It should include `"build": "next build"` and `"start": "next start"`.

In the **AWS Console**:

1. Open **EC2 → Key Pairs → Create key pair**. Name it `nextjs-ec2-key`, select **RSA** and **.pem**, and store the downloaded key privately.
2. Open **EC2 → Instances → Launch instances**. Choose **Ubuntu Server 24.04 LTS**, select the key pair, and choose an instance size (a small learning app can start with `t3.micro`).
3. Create a security group with these inbound rules:

| Type | Port | Source | Reason |
| --- | --- | --- | --- |
| SSH | 22 | **My IP** | Only your current IP can administer the server. |
| HTTP | 80 | `0.0.0.0/0` | Visitors can open the site. |
| HTTPS | 443 | `0.0.0.0/0` | For HTTPS later. |

Do **not** open port 3000 publicly: Nginx will be the public entry point. Launch the instance, wait for **Running** and two passed status checks, and copy its **Public IPv4 address**.

**Stop here.** Confirm you copied the IP. Never share the `.pem` file.

## 2. Connect and install Node.js, Nginx, and PM2

On **your Windows computer**, run in PowerShell (adjust the file path if needed):

~~~powershell
$keyPath = "$HOME\Downloads\nextjs-ec2-key.pem"
icacls $keyPath /inheritance:r
icacls $keyPath /grant:r "$env:USERNAME`:(R)"
ssh -i $keyPath ubuntu@YOUR_EC2_PUBLIC_IP
~~~

The `icacls` commands make the private key private enough for SSH. Type `yes` if asked to trust the server. You are now at the **EC2 terminal**.

On the **EC2 terminal**, run:

~~~bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y nginx git curl
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2
sudo systemctl enable --now nginx
node --version
npm --version
pm2 --version
~~~

This updates Ubuntu; installs Nginx, Git, and Node.js 22 LTS (including npm); then installs PM2. Visit `http://YOUR_EC2_PUBLIC_IP`; the Nginx welcome page should appear.

**Stop here.** Share the three version outputs and whether the Nginx page opens.

## 3. Create the deployment account

On the **EC2 terminal**:

~~~bash
sudo adduser --disabled-password --gecos "" deploy
echo 'deploy ALL=(ALL) NOPASSWD: /bin/systemctl reload nginx' | sudo tee /etc/sudoers.d/deploy-nginx
sudo chmod 440 /etc/sudoers.d/deploy-nginx
sudo -u deploy mkdir -p /home/deploy/apps/YOUR_REPOSITORY
sudo -u deploy mkdir -p /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
id deploy
ls -ld /home/deploy/apps/YOUR_REPOSITORY
~~~

GitHub Actions will use this limited `deploy` account. It can reload Nginx but does not have unrestricted passwordless administrator access.

**Stop here.** Share the final two outputs.

## 4. Create an SSH key for GitHub Actions

Create a *new deployment-only key* on **your Windows computer**. Do not reuse the EC2 `.pem` login key:

~~~powershell
$actionsKeyPath = "$HOME\.ssh\github-actions-ec2"
ssh-keygen -t ed25519 -C "github-actions-ec2" -f $actionsKeyPath
Get-Content "$actionsKeyPath.pub"
~~~

Leave the passphrase blank because Actions cannot type one. Copy the printed public-key line (begins with `ssh-ed25519`). On the **EC2 terminal**, substitute that complete public key:

~~~bash
echo 'PASTE_FULL_PUBLIC_KEY_HERE' | sudo tee -a /home/deploy/.ssh/authorized_keys
sudo chown deploy:deploy /home/deploy/.ssh/authorized_keys
sudo chmod 600 /home/deploy/.ssh/authorized_keys
~~~

Test from **your Windows computer**:

~~~powershell
ssh -i $actionsKeyPath deploy@YOUR_EC2_PUBLIC_IP "whoami"
~~~

It should print `deploy`. In **GitHub**, go to **Repository → Settings → Secrets and variables → Actions → New repository secret**, and add:

| Secret name | Value |
| --- | --- |
| `EC2_HOST` | EC2 public IPv4 address |
| `EC2_USER` | `deploy` |
| `EC2_SSH_KEY` | Full private contents of `$actionsKeyPath`, not the `.pub` file |

To copy the private key, only on **your Windows computer**:

~~~powershell
Get-Content $actionsKeyPath -Raw
~~~

Never put that key in a commit, workflow, or message.

**Stop here.** Share the `whoami` result and confirm only that the three secret *names* are saved.

## 5. Store environment variables securely

Your production environment file stays only on EC2. On the **EC2 terminal**:

~~~bash
sudo -u deploy nano /home/deploy/apps/YOUR_REPOSITORY/.env.production
~~~

Add your real values, for example:

~~~env
NODE_ENV=production
NEXT_PUBLIC_API_URL=https://api.your-domain.example
DATABASE_URL=replace-with-real-secret
~~~

Save with `Ctrl+O`, Enter, then `Ctrl+X`. Variables beginning `NEXT_PUBLIC_` are visible in browser JavaScript, so never use that prefix for passwords, private API keys, or database credentials. Protect the file:

~~~bash
sudo chown deploy:deploy /home/deploy/apps/YOUR_REPOSITORY/.env.production
sudo chmod 600 /home/deploy/apps/YOUR_REPOSITORY/.env.production
~~~

On **your Windows computer**, stop environment files being committed:

~~~powershell
Add-Content .gitignore "`n.env`n.env.*`n!.env.example"
git add .gitignore
git commit -m "Ignore environment files"
git push origin main
~~~

You may commit an `.env.example` containing names only, never secret values.

**Stop here.** Confirm `.env.production` exists on EC2 and secrets were not committed.

## 6. Configure Nginx as reverse proxy

On the **EC2 terminal**:

~~~bash
sudo nano /etc/nginx/sites-available/YOUR_REPOSITORY
~~~

Paste this. Keep `server_name _` while you use a public IP:

~~~nginx
server {
    listen 80;
    listen [::]:80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
~~~

Enable, test, then reload Nginx:

~~~bash
sudo ln -s /etc/nginx/sites-available/YOUR_REPOSITORY /etc/nginx/sites-enabled/YOUR_REPOSITORY
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
~~~

`nginx -t` must report success before reloading. A 502 response is expected until Next.js is deployed.

**Stop here.** Share the `nginx -t` output.

## 7. Create the GitHub Actions workflow

On **your Windows computer**, at the repository root:

~~~powershell
New-Item -ItemType Directory -Force .github\workflows
notepad .github\workflows\deploy.yml
~~~

Paste this into `.github/workflows/deploy.yml`, replacing the repository and GitHub name placeholders:

~~~yaml
name: Deploy Next.js to EC2

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy over SSH
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script_stop: true
          script: |
            set -e
            APP_DIR=/home/deploy/apps/YOUR_REPOSITORY
            REPO_URL=https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git

            if [ ! -d "$APP_DIR/.git" ]; then
              git clone "$REPO_URL" "$APP_DIR"
            fi

            cd "$APP_DIR"
            git fetch origin main
            git checkout -B main origin/main
            npm ci
            npm run build
            pm2 start npm --name "my-next-app" -- start -- -p 3000 || pm2 restart "my-next-app"
            pm2 save
            sudo systemctl reload nginx
~~~

The `on` section makes a push to `main` trigger deployment. GitHub creates a temporary Linux runner, securely uses the three secrets to SSH, clones only on the first run, then fetches later versions. `npm ci` installs locked dependency versions; the build reads EC2 `.env.production`; PM2 starts or restarts the app and remembers it.

### Private GitHub repository only

EC2 needs read access for the initial clone. Create a fine-grained GitHub personal access token restricted to this repository with **Contents: Read-only**. Save it as the Actions secret `REPO_READ_TOKEN`, then change `REPO_URL` to:

~~~yaml
REPO_URL=https://x-access-token:${{ secrets.REPO_READ_TOKEN }}@github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git
~~~

For public repositories, retain the original URL.

Commit and push from **your Windows computer**:

~~~powershell
git add .github/workflows/deploy.yml
git commit -m "Add EC2 deployment workflow"
git push origin main
~~~

In **GitHub**, open **Actions → Deploy Next.js to EC2** and inspect the newest run.

**Stop here.** Share the exact non-secret error if it fails, or confirm it completed.

## 8. Make PM2 survive reboots and verify

After a successful first deployment, run this on the **EC2 terminal**:

~~~bash
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u deploy --hp /home/deploy
sudo -u deploy pm2 save
sudo -u deploy pm2 status
sudo -u deploy pm2 logs my-next-app --lines 50
~~~

The first command creates a systemd startup service so PM2 returns after an EC2 reboot. The process should show `online`. Visit `http://YOUR_EC2_PUBLIC_IP`; future pushes to main now deploy automatically.

**Stop here.** Confirm that the app opens through its public IP.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Browser times out | Instance runs, has a public IP, and port 80 allows `0.0.0.0/0`. |
| SSH times out | Security-group port 22 must allow your current public IP. |
| Actions cannot SSH | Verify `EC2_HOST`, `EC2_USER=deploy`, private `EC2_SSH_KEY`, and permissions on `authorized_keys`. |
| `npm ci` fails | Commit `package-lock.json` and read the action log. |
| Build fails | Check all necessary EC2 `.env.production` values. |
| Nginx returns 502 | Next.js/PM2 is likely offline; inspect PM2 logs. |
| Deployment did not run | Confirm the commit reached `main`, then open Actions. |

Useful commands on the **EC2 terminal**:

~~~bash
sudo -u deploy pm2 status
sudo -u deploy pm2 logs my-next-app --lines 100
curl http://127.0.0.1:3000
sudo systemctl status nginx --no-pager
sudo nginx -t
sudo tail -n 100 /var/log/nginx/error.log
~~~

## Later: domain and HTTPS

After HTTP works, point a DNS `A` record at the EC2 public IP. Replace `server_name _` with your domain, test/reload Nginx, then use Certbot to obtain a free Let's Encrypt certificate.
