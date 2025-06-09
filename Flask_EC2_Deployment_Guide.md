
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


2. **Move your .pem file into the ec2 folder.** For example:

     Nexus-Bloom.pem → C:\Users\<YourUsername>\ec2\Nexus-Bloom.pem

3. **Open Command Prompt and navigate to the folder:**

   ```bash
   cd C:\Users\<YourUsername>\ec2
   
(Optional) If you are using Git Bash or WSL, you can use the following Unix-style command to set permissions:
         chmod 400 Nexus-Bloom.pem
⚠️ On Windows, this step is not always required, but Git Bash or WSL might enforce it.

4. **SSH into your instance:**
   ```bash
   ssh -i Nexus-Bloom.pem ubuntu@<EC2-Public-IP>
Replace <EC2-Public-IP> with your actual EC2 instance's public IP address (e.g., 13.221.93.123).

## ✅ Option 2: Connect via Bash or Terminal (Linux/macOS)
1. **Move the .pem file to a safe directory:**
   ```bash
   mkdir ~/ec2
   mv ~/Downloads/Nexus-Bloom.pem ~/ec2/

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

---

## 🔥 Step 6: Run Flask in Production (Gunicorn)

Install Gunicorn:

```bash
pip install gunicorn
```

If using app factory pattern (`create_app()`):

```bash
gunicorn --bind 0.0.0.0:5000 'app:create_app()'
```

---

## 🌐 Step 7: Configure Nginx

Install Nginx:

```bash
sudo apt install nginx -y
```

Create a config file:

```bash
sudo nano /etc/nginx/sites-available/nexus-bloom
```

Paste:

```nginx
server {
    listen 80;
    server_name <EC2-Public-IP>;

    location / {
        include proxy_params;
        proxy_pass http://127.0.0.1:5000;
    }
}
```

Enable config:

```bash
sudo ln -s /etc/nginx/sites-available/nexus-bloom /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl restart nginx
```

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

| Term        | Meaning                                                                 |
|-------------|-------------------------------------------------------------------------|
| EC2         | Virtual server (cloud-based machine) on AWS                             |
| SSH         | Secure remote access to servers                                         |
| Flask       | Python micro web framework                                              |
| Gunicorn    | Production WSGI server for running Python web apps                      |
| Nginx       | High-performance reverse proxy and web server                           |
| venv        | Python virtual environment manager                                      |
| systemctl   | Tool to manage services like Gunicorn on startup                        |
| `scp`       | Secure file transfer between local and remote machine                   |

### Learn More:
- [Gunicorn Docs](https://docs.gunicorn.org)
- [Nginx Docs](https://nginx.org/en/docs/)
- [AWS EC2 Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html)
- [Deploy Flask with Gunicorn & Nginx](https://flask.palletsprojects.com/en/2.3.x/deploying/)
