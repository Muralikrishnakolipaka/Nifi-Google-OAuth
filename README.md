# Nifi-OAuth
Nifi integrated with Google OAuth
# NiFi Integration with Google OAuth and Nginx Configuration

This documentation provides a step-by-step guide for integrating Apache NiFi with Google OAuth and configuring Nginx as a reverse proxy.

---

## Prerequisites

Ensure the following are installed on your system:

- **Java**: Version 21
- **Packages**: `wget`, `unzip`, `keytool`, `nginx`, `certbot`

### Update and Install Necessary Tools

```bash
sudo dnf update -y
sudo dnf upgrade -y
sudo dnf install wget unzip

### Install Java
```bash
sudo dnf install java-21-openjdk
```
### Set the Java path
```bash
export JAVA_HOME="/usr/lib/jvm/java-21-openjdk-21.0.5.0.11-2.el9.x86_64"
```
## Install and Configure Nginx
### Install Nginx and Certbot
```bash
sudo dnf install epel-release -y
sudo dnf install nginx -y
sudo dnf install certbot python3-certbot-nginx -y
```
### Start and Enable Nginx
```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```
### Install NiFi
## Download and extract NiFi
```bash
cd /opt
wget https://dlcdn.apache.org/nifi/2.1.0/nifi-2.1.0-bin.zip
unzip nifi-2.1.0-bin.zip
mv nifi-2.1.0 nifi
```
### Create SSL directories
```bash
mkdir -p /etc/nifi/ssl/
```
## Generate Keystore and Truststore
### Create Keystore
```bash
keytool -genkey -alias nifi-prod.bluedotspace.io \
    -keyalg RSA -keystore /etc/nifi/ssl/nifi.jks \
    -keysize 2048 -validity 3650
```
Enter and confirm a keystore password.
### Export Certificate
```bash
keytool -export -alias nifi-prod.bluedotspace.io -keystore /etc/nifi/ssl/nifi.jks \
    -file /etc/nifi/ssl/nifi.crt
```
### Create Truststore
```bash
keytool -import -alias nifi-prod.bluedotspace.io -file /etc/nifi/ssl/nifi.crt \
    -keystore /etc/nifi/ssl/truststore.jks -trustcacerts
```
### Set Permissions
```bash
sudo chmod 644 /etc/nifi/ssl/nifi.jks /etc/nifi/ssl/truststore.jks
```
### Configure Google OAuth
```bash
Log in to the Google Developers Console and create a new project.

Go to OAuth consent screen:

Select User Type: Internal
Fill out App Information:
App Name: <NiFi>
Support Email: example@gmail.com
Application Homepage: https://<nifi-domain>/nifi
Privacy Policy: https://<nifi-domain>/privacy-policy
Terms of Service: https://<nifi-domain>/terms-of-service
Add Authorized Domain: example.io
Save and proceed to Scopes, then add test users.
Create OAuth Credentials:
Application Type: Web Application
Authorized Redirect URI: https://<nifi-domain>/nifi-api/access/oidc/callback
Copy the Client ID and Client Secret for use in NiFi configuration.
```
### NiFi Configuration
## Update the NiFi configuration file
```bash
nifi.web.https.host=<nifi-domain>
nifi.web.https.port=9443
nifi.security.keystore=/etc/nifi/ssl/nifi.jks
nifi.security.keystoreType=PKCS12
nifi.security.keystorePasswd=<keystorePasswd>
nifi.security.truststore=/etc/nifi/ssl/truststore.jks
nifi.security.truststoreType=PKCS12
nifi.security.truststorePasswd=<truststorePasswd>
nifi.security.user.oidc.discovery.url=https://accounts.google.com/.well-known/openid-configuration
nifi.security.user.oidc.client.id=<Client ID>
nifi.security.user.oidc.client.secret=<Client Secret>
```
### Update authorizers.xml
```bash
<userGroupProvider>
    <property name="Initial User Identity 1">example@gmail.com</property>
</userGroupProvider>

<accessPolicyProvider>
    <property name="Initial Admin Identity">example@gmail.com</property>
</accessPolicyProvider>
```
## Set the Java path in nifi-env.sh
```bash
export JAVA_HOME="/usr/lib/jvm/java-21-openjdk-21.0.5.0.11-2.el9.x86_64"
```
## Create a systemd service file 
```bash
[Unit]
Description=Apache NiFi
After=network.target
 
[Service]
Type=forking
User=root
Group=root
PIDFile=/opt/nifi/run/nifi.pid
ExecStart=/opt/nifi/bin/nifi.sh start
ExecStop=/opt/nifi/bin/nifi.sh stop
Restart=on-failure
RestartSec=5s
 
[Install]
WantedBy=multi-user.target
```

### Nginx Configuration
## Create a configuration file for your NiFi domain
```bash
sudo vi /etc/nginx/conf.d/nifi-domain.conf
```
## Add the following content
```bash
server {
    server_name <nifi-domain>;

    location / {
        proxy_pass https://<private_IP>:9443;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
    }

    listen 80;
}
```
## Obtain an SSL certificate using Certbot
```bash
sudo certbot --nginx -d <nifi-domain>
```
## Reload and restart Nginx
```bash
sudo systemctl reload nginx
sudo systemctl restart nginx
```
### Start Nifi service
```bash
systemctl start nifi
systemctl enable nifi
```

