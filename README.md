🚀 Simple AWS Deployment Guide

This guide explains how to deploy a React (Next.js) frontend, a NestJS backend, and a Postgres database using AWS services including S3, EC2, Docker, and Nginx.

📌 Overview

This deployment covers:

S3 bucket for static frontend hosting

S3 bucket for backend storage

EC2 instance for NestJS backend + Dockerized Postgres

Nginx as a reverse proxy

Connecting frontend & backend

1. ✅ Prepare AWS Account

Ensure:

IAM user has AdministratorAccess

AWS CLI installed & configured:

aws configure

2. 🎨 Setup S3 Bucket – Frontend Hosting
2.1 Create Bucket

Name: your-frontend-dev

Disable Block Public Access (for public frontend hosting)

Enable Static Website Hosting

Index: index.html

Error: index.html

2.2 Bucket Policy

Replace with your bucket name:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::your-frontend-dev/*"
    }
  ]
}

3. 🗂️ Setup S3 Bucket – Backend File Storage

Create another bucket:

Name: your-backend-storage-dev

Keep private access.

4. ⚙️ Launch EC2 Instance for Backend

Recommended:

Ubuntu 22.04 LTS

t3.medium

Allow ports:
22 (SSH), 80 (HTTP), 443 (HTTPS)

5. 📦 Install Required Software on EC2
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx git curl build-essential

Install Node.js (LTS)
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs

Install Docker + Docker Compose
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker --now
sudo usermod -aG docker $USER

6. 🐘 Setup Postgres (Dockerized)

Inside EC2:

mkdir ~/postgres && cd ~/postgres


Create docker-compose.yml:

version: '3.8'
services:
  db:
    image: postgres:15
    restart: always
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpass
      POSTGRES_DB: devdb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:


Start Postgres:

docker-compose up -d

7. 🧩 Clone & Setup Backend (NestJS)
cd ~
git clone -b dev <your-repo-url> backend
cd backend
npm install


Create .env:

DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USER=devuser
DATABASE_PASSWORD=devpass
DATABASE_NAME=devdb

AWS_S3_BUCKET=your-backend-storage-dev
AWS_REGION=your-region
AWS_ACCESS_KEY_ID=xxxx
AWS_SECRET_ACCESS_KEY=xxxx

PORT=3000


Test:

npm run start:dev


Backend runs at:

http://<EC2_PUBLIC_IP>:3000

8. 🌐 Configure Nginx Reverse Proxy

Remove default config:

sudo rm /etc/nginx/sites-enabled/default


Create backend config:

sudo nano /etc/nginx/sites-available/backend


Paste:

server {
    listen 80;
    server_name _;

    root /var/www/react;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:3000/;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 600s;
        proxy_read_timeout    600s;
        proxy_send_timeout    600s;
        proxy_buffering off;
    }
}


Enable & restart:

sudo ln -s /etc/nginx/sites-available/backend /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

9. 🎯 Deploy Frontend (Next.js) on S3

On your local machine:

git clone -b dev <frontend-repo> frontend
cd frontend

npm install
npm run build
npm run export


Upload:

aws s3 sync out/ s3://your-frontend-dev --delete

10. 🔗 Connect Frontend to Backend

Update frontend .env:

NEXT_PUBLIC_API_URL=http://<EC2_PUBLIC_IP>


Rebuild & redeploy to S3.