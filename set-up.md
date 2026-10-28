For AWS Free Tier learning, use **one EC2 instance** and run everything with Docker Compose.

Do not use RDS, ECS, Load Balancer, Route 53, or Lightsail yet—they add cost or complexity. Your learning architecture becomes:

```text
Browser → EC2 public IP → Docker Compose
                         ├─ Frontend (Next.js)
                         ├─ Backend (NestJS)
                         └─ MySQL
```

Choose:

- EC2: `t2.micro` or `t3.micro` (select the option marked **Free tier eligible**)
- OS: Ubuntu 24.04
- Storage: 20 GB `gp3` is enough for learning
- Region: Mumbai (`ap-south-1`) if it offers Free Tier eligibility for your account

Important: free-tier eligibility and duration depend on your AWS account type/date. Check the price shown in the AWS launch screen before creating anything.

## Simple learning deployment steps

1. AWS Console → EC2 → **Launch instance**.
2. Choose Ubuntu 24.04 and a Free Tier eligible instance type.
3. Create a key pair and download the `.pem` file.
4. Create a security group with these inbound rules:
   - SSH `22`: My IP
   - Custom TCP `3000`: Anywhere
   - Custom TCP `3001`: Anywhere temporarily for testing

5. Launch the instance.

6. Connect from Windows PowerShell:

```powershell
ssh -i C:\path\to\your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

7. Install Docker:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2 git
sudo usermod -aG docker ubuntu
exit
```

Reconnect, then copy your project to the server—preferably by pushing it to GitHub and cloning it:

```bash
git clone YOUR_GITHUB_REPOSITORY_URL itinerary
cd itinerary
```

8. Create a root `.env` file:

```bash
nano .env
```

```env
MYSQL_DATABASE=itinerary_db

PORT=3001
FRONTEND_URL=http://YOUR_EC2_PUBLIC_IP:3000
JWT_SECRET=create-a-long-random-secret

OLLAMA_API_KEY=your_key
OLLAMA_URL=https://ollama.com/api/generate
OLLAMA_MODEL=gpt-oss:120b

UNSPLASH_ACCESS_KEY=
NEXT_PUBLIC_API_URL=http://YOUR_EC2_PUBLIC_IP:3001
```

9. Start the app:

```bash
docker compose up -d --build
docker compose ps
```

10. Open:

```text
http://YOUR_EC2_PUBLIC_IP:3000
```

The frontend should load; it calls your API at port `3001`.

For this learning version, it is okay to use ports 3000/3001. Do not open MySQL port `3307` or phpMyAdmin port `8080` to the public internet unless you specifically need them; then remove those rules immediately afterward.

When you finish learning, terminate the EC2 instance and delete any EBS volumes and Elastic IPs you no longer need—those are common surprise-cost sources.