# 🚀 Deploying a Flask Backend to AWS EC2 (Production-Ready)

This guide walks you step-by-step through hosting a Flask backend using an AWS EC2 instance, serving via **Gunicorn** and **Nginx**, accessible from the internet.

---

## 📌 Table of Contents

1. [Requirements](#requirements)
2. [Step 1: Launch an EC2 Instance](#step-1-launch-an-ec2-instance)
3. [Step 2: Connect to EC2 via SSH](#step-2-connect-to-ec2-via-ssh)
4. [Step 3: Copy Project to EC2](#step-3-copy-project-to-ec2)
5. [Step 4: Setup Flask Environment](#step-4-setup-flask-environment)
6. [Step 5: Run Flask Dev Server (Optional)](#step-5-run-flask-dev-server-optional)
7. [Step 6: Run Flask in Production (Gunicorn)](#step-6-run-flask-in-production-gunicorn)
8. [Step 7: Configure Nginx](#step-7-configure-nginx)
9. [Step 8: Update EC2 Security Group (Inbound Rules)](#step-8-update-ec2-security-group-inbound-rules)
10. [Step 9: Updating Your Code](#step-9-updating-your-code)
11. [Step 10: Delete EC2 Instance](#step-10-delete-ec2-instance)
12. [Step 11: Auto-start Flask App on Reboot with systemd](#step-11-auto-start-flask-app-on-reboot-with-systemd)
13. [Concepts & Resources](#concepts--resources)

---

## ✅ Requirements

- AWS Account
- Flask backend app (`app.py` or `create_app()`)
- Basic terminal skills
- Vercel frontend (optional)

---

## 🖥 Step 1: Launch an EC2 Instance

1. Go to [EC2 Console](https://console.aws.amazon.com/ec2).
2. Click `Launch Instance`
   - Name: `Flask-Backend`
   - OS: **Ubuntu 22.04 LTS**
   - Instance type: **t2.micro** (Free Tier eligible)
3. **Key Pair**: Create or use an existing `.pem` file. Download and keep it safe.
4. **Firewall/Security Group**:
   - Allow **SSH (22)** and **HTTP (80)**

---

## 🔐 Step 2: Connect to EC2 via SSH

Once your EC2 instance is running, follow these steps to connect to it from your local machine using the `.pem` key file.

---

### ✅ Option 1: Connect Using Command Prompt (Windows)

1. **Create a folder named `ec2`** inside your user directory:

   ```bash
   mkdir C:\Users\<YourUsername>\ec2

   ```

2. **Move your .pem file into the ec2 folder.** For example:

   Nexus-Bloom.pem → C:\Users\<YourUsername>\ec2\Nexus-Bloom.pem

3. **Open Command Prompt and navigate to the folder:**

   ```bash
   cd C:\Users\<YourUsername>\ec2

   ```

(Optional) If you are using Git Bash or WSL, you can use the following Unix-style command to set permissions:
chmod 400 Nexus-Bloom.pem
⚠️ On Windows, this step is not always required, but Git Bash or WSL might enforce it.

4. **SSH into your instance:**
   ```bash
   ssh -i Nexus-Bloom.pem ubuntu@<EC2-Public-IP>
   Replace <EC2-Public-IP> with your actual EC2 instance's public IP address (e.g., 13.*2*.9*.1**).
   ```

## ✅ Option 2: Connect via Bash or Terminal (Linux/macOS)

1. **Move the .pem file to a safe directory:**

   ```bash
   mkdir ~/ec2
   mv ~/Downloads/Nexus-Bloom.pem ~/ec2/

   ```

2. **Set proper permissions to prevent security warnings:**

chmod 400 ~/ec2/Nexus-Bloom.pem
SSH into the EC2 instance:

3. **ssh -i ~/ec2/Nexus-Bloom.pem ubuntu@<EC2-Public-IP>**
   💡 Tip: If your username is different (e.g., for Amazon Linux), replace ubuntu with the appropriate user (ec2-user, admin, etc.).

---

## 📦 Step 3: Copy Project to EC2

**Option 1: Using `git clone` (recommended)**

```bash
sudo apt update && sudo apt install git -y
git clone https://github.com/yourusername/nexus-bloom-backend.git
```

**Option 2: Using `scp`**

From your local terminal:

```bash
scp -i Nexus-Bloom.pem -r ./nexus-bloom-backend ubuntu@<EC2-IP>:/home/ubuntu/
```

---

## ⚙️ Step 4: Setup Flask Environment

```bash
cd nexus-bloom-backend
sudo apt install python3-pip python3-venv -y
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

---

## 🧪 Step 5: Run Flask Dev Server (Optional)

```bash
python app.py
```

Visit:

```
http://<EC2-Public-IP>:5000
```

## 🌐 Step 6: Configure Nginx

# Before We Start Let's Learn NGINX, HTTPS, and Let's Encrypt For A Moment (Skip to set up if you already farmiliar with this)

Let's pause. I want to share something with you for a bit — a story behind the decisions we made deploying this Flask app securely in production with nginx.

In software development, sometimes it's not just about writing code, it's about understanding **why** we do certain things — why we add layers like NGINX, why we bother with certificates, or even why we choose a `.nip.io` domain over a shiny `.com` one or why we don't move with a raw http ip address.

This small story captures that journey. From raw IPs to a secure HTTPS-ready production environment. Let’s dive in.

---

## Why Use HTTPS (SSL Certificates)

Initially, if your frontend is hosted on **Vercel**, or any other **server** served over `https://`, and your backend Flask API is on a public EC2 IP over plain `http://`. This will lead to what is called a **Mixed Content Error** and blocked API calls.

**Solution for this**: Add HTTPS to the backend. Why so?

- Encrypts communication between frontend ↔️ backend
- Prevents man-in-the-middle attacks
- Required by modern browsers
- Removes security warnings and ensures trust

So that's why we will turn to **Let's Encrypt**, a free SSL provider, to generate our SSL certificates.

---

## Why Use `nip.io`

We didn’t want to buy a domain. But SSL certificates need a domain to bind to.

Enter **nip.io**:

- Free DNS service that maps `13.*2*.9*.1**.nip.io` → `13.*2*.9*.1**`
- Acts like a domain, accepted by Let's Encrypt
- Perfect for temporary or low-budget deployments
- No DNS setup required

This made `13.*2*.9*.1**.nip.io` our production-ready public endpoint without buying a domain.

---

## Why Not Use `ngrok`

We thought about ngrok. It's fast, secure, and free (for dev).

But:

- It's **temporary**: Every tunnel expires or changes
- SSL is bound to `*.ngrok.io`, not your own subdomain
- Not suitable for stable production deployments
- Can't integrate with NGINX or serve custom routes

So while great for quick tests, it's not what you want in real deployments.

---

## Reverse Proxy with NGINX

Think of **NGINX** as the middleman.

- Client requests → NGINX → Flask backend
- NGINX listens on port **443** (HTTPS)
- It decrypts SSL, then **proxies** to Flask on port **5000** (HTTP)

This is why it’s called a **reverse proxy** — unlike a forward proxy (which protects the client), NGINX protects and optimizes access to the backend.

---

## Setup Steps

### 1. Install NGINX

```bash
sudo apt update
sudo apt install nginx
```

### 2. Check NGINX Status

```bash
sudo systemctl status nginx
```

### 3. Configure Your NGINX Site

```bash
sudo nano /etc/nginx/sites-available/nexus-bloom
```

Paste this config:

   ```nginx
   server {
      listen 80;
      server_name 13.***.9*.1*3.nip.io;

      location /.well-known/acme-challenge/ {
         root /var/www/html;
      }

      location / {
         return 301 https://$host$request_uri;
      }
   }


## Issue SSL Certificate with Certbot

### 1. Install Certbot

```bash
sudo apt install certbot python3-certbot-nginx
```

### 2. Run Certbot

```bash
sudo certbot --nginx -d 13.*2*.9*.1**.nip.io
```

Certbot will:

- Talk to Let's Encrypt
- Place a challenge in `/.well-known/acme-challenge/`
- Validate your domain
- Create SSL certs and configure NGINX
- Your /etc/nginx/sites-available/nexus-bloom will contain the code below


server {
    listen 443 ssl;
    server_name 13.*2*.9*.1**.nip.io;

    ssl_certificate /etc/letsencrypt/live/13.*2*.9*.1**.nip.io/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/13.*2*.9*.1**.nip.io/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location /api/ {
        proxy_pass http://localhost:5000/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
## Run and Test Your Flask API

Start Flask:

```bash
python app.py
```

Test in Postman or browser:

```
https://13.*2*.9*.1**.nip.io/api/membership/verify-email
```

✅ Should work without Mixed Content or SSL errors.

---

## Summary

- HTTPS avoids frontend–backend communication errors
- `nip.io` lets you use your EC2 IP like a domain
- Certbot + NGINX makes it secure with a free certificate
- NGINX listens on 443 and proxies to Flask on 5000

This stack is now **ready for production**, all using free tools — no domain needed.

---

# Set up Gunicorn WSGI in Production

## What is WSGI?

WSGI stands for **Web Server Gateway Interface**. It's a specification that allows Python web applications (like Flask) to communicate with web servers. Think of it as a bridge between your app and the outside world.

Flask is a **WSGI application**, meaning it needs a WSGI-compatible server like **Gunicorn** or **uWSGI** to run efficiently in production.

---

## Why Not Use Flask's Built-in Server?

Flask’s default development server (started via `python app.py`) is:

* Not optimized for handling multiple simultaneous requests
* Lacks features like graceful restarts or process management
* Single-threaded and intended only for development and debugging

---

## 🔧 Enter Gunicorn

Gunicorn is a **WSGI-compliant server** that serves your Flask app efficiently by using multiple **worker processes**.

### What Are Worker Processes?

A worker process is an independent instance of your app that can handle one or more requests. Multiple workers allow your app to handle more users concurrently.

### What Does Spawning Mean?

Spawning refers to the creation of multiple worker processes by the server. When you start Gunicorn, it **spawns** several workers to manage load and concurrency.

---

## How to Run Gunicorn

```bash
gunicorn -w 4 -b 127.0.0.1:5000 app:app
```

* `-w 4`: Spawn 4 worker processes
* `-b`: Bind to localhost on port 5000
* `app:app`: Refers to the Python file `app.py` and the Flask app instance named `app`

Gunicorn runs your Flask app, and NGINX (on the frontend) handles HTTPS traffic and forwards it to Gunicorn via HTTP.

---

## 🔄 Gunicorn vs Alternatives

| Server        | Description                                   |
| ------------- | --------------------------------------------- |
| **uWSGI**     | Powerful and flexible, but complex setup      |
| **Waitress**  | Lightweight, works on Windows too             |
| **Hypercorn** | Supports ASGI for async apps like FastAPI     |
| **Daphne**    | Good for Django Channels, supports WebSockets |

**Gunicorn** is the most common and reliable option for Flask apps.

---

## ✅ Summary

* **WSGI** lets Python apps talk to servers
* **Gunicorn** spawns multiple workers for performance
* **NGINX** handles HTTPS and proxies to Gunicorn
* **HTTPS** prevents frontend-backend security issues
* **nip.io** offers a free, IP-based domain
* **Certbot** adds free SSL certificates

This stack is now **ready for production**, fully open-source and budget-friendly — no custom domain needed!


- Auto-renew SSL:

```bash
sudo certbot renew --dry-run
```

---

Hope this helped you understand **not just how** to deploy Flask securely, but also **why** every decision mattered along the way.

---

## 🔓 Step 8: Update EC2 Security Group (Inbound Rules)

1. Go to EC2 Console → Instances → Select your instance
2. Scroll to **Security Groups** → Click on it
3. Go to **Inbound Rules → Edit**
4. Add Rule:
   - Type: HTTP
   - Port: 80
   - Source: Anywhere (0.0.0.0/0)

---

## 🔁 Step 9: Updating Your Code

On EC2:

```bash
cd nexus-bloom-backend
git pull origin main   # if using Git
```

Restart Gunicorn:

```bash
pkill gunicorn
gunicorn --bind 0.0.0.0:5000 'app:create_app()'
```

---

## ❌ Step 10: Delete EC2 Instance

Go to EC2 Console → Instances → Select instance → Actions → Instance State → Terminate

---

## ⚙️ Step 11: Auto-start Flask App on Reboot with systemd

Create a service file:

```bash
sudo nano /etc/systemd/system/nexus-bloom.service
```

Paste:

```ini
[Unit]
Description=Gunicorn instance to serve Nexus Bloom
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/nexus-bloom-backend
Environment="PATH=/home/ubuntu/nexus-bloom-backend/venv/bin"
ExecStart=/home/ubuntu/nexus-bloom-backend/venv/bin/gunicorn --workers 3 --bind 0.0.0.0:5000 'app:create_app()'

[Install]
WantedBy=multi-user.target
```

Enable and start service:

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable nexus-bloom
sudo systemctl start nexus-bloom
```

---

## 📚 Concepts & Resources

| Term      | Meaning                                               |
| --------- | ----------------------------------------------------- |
| EC2       | Virtual server (cloud-based machine) on AWS           |
| SSH       | Secure remote access to servers                       |
| Flask     | Python micro web framework                            |
| Gunicorn  | Production WSGI server for running Python web apps    |
| Nginx     | High-performance reverse proxy and web server         |
| venv      | Python virtual environment manager                    |
| systemctl | Tool to manage services like Gunicorn on startup      |
| `scp`     | Secure file transfer between local and remote machine |

### Learn More:

- [Gunicorn Docs](https://docs.gunicorn.org)
- [Nginx Docs](https://nginx.org/en/docs/)
- [AWS EC2 Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html)
- [Deploy Flask with Gunicorn & Nginx](https://flask.palletsprojects.com/en/2.3.x/deploying/)
