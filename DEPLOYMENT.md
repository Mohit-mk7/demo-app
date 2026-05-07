# DEPLOYMENT.md — Nilavan Realtors (lt-nilavan)

Server: AWS EC2 t3.micro — Ubuntu 24.04 LTS  
Live URL: https://nilavan-mohit.duckdns.org  
Stack: Next.js 15 · Node.js 18 · PM2 · Nginx · Let's Encrypt (Certbot)

---

# Step 1 — Launch EC2 Instance

1. Log in to AWS Console → EC2 → Launch Instance
2. Select Ubuntu Server 24.04 LTS (Free Tier eligible)**
3. Instance type: t3.micro
4. Create a new key pair → download `.pem` file
5. Security Group — allow inbound:
   - Port 2222 (SSH)
   - Port 80 (HTTP)
   - Port 443 (HTTPS)
   - Port 3000 (HTTPS)
6. Launch the instance, note the Public IPv4 address

---

# Step 2 — Initial SSH Connection

# Fix key permissions (required on Linux/macOS)
chmod 400 your-key.pem

# Connect as default ubuntu user
ssh -i your-key.pem ubuntu@<YOUR_EC2_PUBLIC_IP>


# Step 3 — Create Non-Root User

# Create deploy user
sudo adduser mohit
sudo usermod -aG sudo mohit

# Copy SSH keys to new user
sudo mkdir -p /home/mohit/.ssh
sudo cp ~/.ssh/authorized_keys /home/mohit/.ssh/
sudo chown -R mohit:mohit /home/mohit/.ssh
sudo chmod 700 /home/mohit/.ssh
sudo chmod 600 /home/mohit/.ssh/authorized_keys

# Switch to the deploy user for all remaining steps
su - mohit

---

## Step 4 — Harden SSH

sudo nano /etc/ssh/sshd_config

Change or confirm these values:

Port 2222
PasswordAuthentication no
PermitRootLogin no

sudo systemctl restart ssh


# Step 5 — Configure UFW Firewall

sudo ufw allow 2222/tcp    # Custom SSH port
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS
sudo ufw enable
sudo ufw status

Expected output:

Status: active
To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere


# Step 6 — Install Node.js 18 via nvm

# Install nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Reload shell
source ~/.bashrc

# Install Node 18
nvm install 18
nvm use 18
nvm alias default 18

# Verify
node -v   # Should output v18.x.x
npm -v

# Step 7 — Install PM2 and Nginx

# PM2 — process manager
npm install -g pm2

# Nginx + Certbot
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx

# Step 8 — Clone and Build the Application

# Clone the repository (must be added as collaborator first)
cd ~
git clone https://github.com/Leadtap/lt-nilavan.git
cd lt-nilavan

# Create environment file
nano .env.local

SENDGRID_API_KEY=SG.your_key_here
SENDGRID_TO_EMAIL=your_verified_email@gmail.com

# Install dependencies
npm install

# Build production bundle
npm run build

# Step 9 — Start Application with PM2

# Start the Next.js app
pm2 start npm --name "nilavan" -- start

# Configure auto-start on server reboot
pm2 startup
# Run the command it outputs (copy-paste it)
pm2 save

# Verify it's running
pm2 list
pm2 logs nilavan --lines 20

Expected PM2 output:

┌─────┬──────────┬─────────────┬─────────┬─────────┬──────────┐
│ id  │ name     │ namespace   │ version │ mode    │ status   │
├─────┼──────────┼─────────────┼─────────┼─────────┼──────────┤
│ 0   │ nilavan  │ default     │ N/A     │ fork    │ online   │
└─────┴──────────┴─────────────┴─────────┴─────────┴──────────┘

# Step 10 — Set Up Free Domain (DuckDNS)

1. Go to https://duckdns.org → login with Google
2. Enter a subdomain name (e.g. `nilavan-mohit`)
3. Enter your EC2 Public IP → click Update IP
4. Your domain: `nilavan-mohit.duckdns.org` now points to your server

# Step 11 — Configure Nginx as Reverse Proxy

sudo nano /etc/nginx/sites-available/nilavan

Paste this configuration:

server {
    listen 80;
    server_name nilavan-mohit.duckdns.org;

    # Security: Block .git directory from public access
    location ~ /\.git {
        deny all;
        return 404;
    }

    location /api/sendgrid {
        limit_req zone=contact_limit burst=2 nodelay;
        limit_req_status 429;

        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

   }
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;

        # Security headers
        add_header X-Frame-Options "DENY";
        add_header X-Content-Type-Options "nosniff";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
        add_header Content-Security-Policy "frame-ancestors 'none'";
    }
}

# Enable the site
sudo ln -s /etc/nginx/sites-available/nilavan /etc/nginx/sites-enabled/

# Remove default site
sudo rm /etc/nginx/sites-enabled/default

# Test config
sudo nginx -t

# Reload
sudo systemctl reload nginx

# Step 12 — Obtain SSL Certificate (Let's Encrypt)


sudo certbot --nginx -d nilavan-mohit.duckdns.org


Follow the prompts:
- Enter your email address
- Agree to Terms of Service (A)
- Choose whether to share email with EFF (optional)

Certbot will automatically:
- Obtain the certificate
- Modify Nginx config to add HTTPS block
- Add HTTP → HTTPS redirect


# Verify renewal works
sudo certbot renew --dry-run

# Step 13 — Verify Everything Works

# Test HTTPS
curl -I https://nilavan-mohit.duckdns.org
# Expect: HTTP/2 200
--------------------------------------------------------
# Test HTTP redirects to HTTPS
curl -I http://nilavan-mohit.duckdns.org
# Expect: HTTP/1.1 301 Moved Permanently, Location: https://...
--------------------------------------------------------
# Test .git is blocked
curl -I https://nilavan-mohit.duckdns.org/.git/config
# Expect: HTTP/2 404
--------------------------------------------------------
# Test X-Frame-Options header
curl -sI https://nilavan-mohit.duckdns.org | grep -i x-frame
# Expect: x-frame-options: DENY
--------------------------------------------------------
# Test PM2 is running
pm2 list

# Test UFW status
sudo ufw status verbose

