
# CI/CD-Nginx

Sample CI/CD Project - Using GitHub Actions with SCP and Nginx on Ubuntu 22.04

## Deployment

Steps to deploy this project:

**Step 1 (Genrerate SSH Key):**

Create a secure directory for SSH keys (~/.ssh) and restricts access to the authorized_keys file.
```bash
  mkdir ~/.ssh
  chmod 700 ~/.ssh
  touch ~/.ssh/authorized_keys
  chmod 600 ~/.ssh/authorized_keys
```

Generate SSH key:
```bash
  cd ~/.ssh
  ssh-keygen -t rsa -b 4096 -C "test@gmail.com"
```
- assign name and passphrase for the SSH key

Save the public SSH key in the server:
```bash
  cat github-actions.pub >> ~/.ssh/authorized_keys
```

View the private SHH key:
```bash
  nano github-actions
```
- copy and save the private SSH key (which will be used as GitHub secret in the future)

Enable SSH key authentication for the SSH key type:
```bash
  nano /etc/ssh/sshd_config
```
- add this line of command: `PubkeyAcceptedKeyTypes=+ssh-rsa`
- restart the SSH service:
```bash
  sudo systemctl restart ssh
```

**Step 2 (Install Dependencies):**

Install the .NET SDK 8.0 on Ubuntu 22.04 by adding the Microsoft repository and updating/installing packages:
```bash
  apt update -y
  apt upgrade -y
  wget -q https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb
  sudo dpkg -i packages-microsoft-prod.deb
  sudo add-apt-repository universe
  sudo apt install apt-transport-https -y
  sudo apt update -y
  sudo apt install dotnet-sdk-8.0 -y
```
Configure a systemd service named cicdsample.service to run a .NET application (CICD-API.dll) as the www-data user:
```bash
  sudo nano /etc/systemd/system/cicdsample.service
```
- copy and paste this sample:
```
[Unit]
Description=CICD

[Service]
WorkingDirectory=/var/www/shopnet
ExecStart=/usr/bin/dotnet /var/www/shopnet/CICD-API.dll
Restart=always
RestartSec=10
SyslogIdentifier=CICD
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target
```
- enable this service:
```bash
  sudo systemctl enable cicdsample
```

Install Nginx, Certbot with the Nginx plugin, check Nginx status, enable it to start at boot:
```bash
  sudo apt install nginx certbot python3-certbot-nginx -y
  systemctl status nginx
  sudo systemctl enable nginx 
```

Create a directory for the shopnet website, and open a new server block configuration file for it:
```bash
  sudo mkdir /var/www/shopnet
  nano /etc/nginx/sites-enabled/shopnet
```
- copy and paste this sample:
```
server {
    root /var/www/shopnet/;

    index index.html index.htm index.nginx-debian.html;

    server_name shopnet.lol;

    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
- restart Nginx:
```bash
  sudo systemctl restart nginx
```

**Step 3 (Define GitHub Secrets):**

Define the secrets that will be used in the GitHub Actions YAML file:

```
REMOTE_HOST : server_ip
REMOTE_USER : root
REMOTE_SSH_KEY : private_key
PASSPHRASE : some_password
REMOTE_TARGET : /var/www/shopnet/
```

**Step 4 (Define GitHub Actions):**

Define the GitHub Actions YAML file (use this project YAML file).

**Step 5 (Obtain SSL Certificate):**

Connect your domain to the server's IP by using an A record.

Install Certbot as a Snap package, create a symbolic link to make it globally accessible, and then use Certbot to obtain an SSL certificate for the Nginx server:
```bash
  sudo snap install --classic certbot
  sudo ln -s /snap/bin/certbot /usr/bin/certbot
  sudo certbot --nginx
```
- add your email address and domain and also accept the rules (Y).

**Step 6 (Re-run the GitHub action):**

Re-run all jobs of your last GitHub push and then go to `https://your_domain/swagger/index.html` to view your website.
