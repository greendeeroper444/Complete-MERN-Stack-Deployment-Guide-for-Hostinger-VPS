# Complete MERN Stack Deployment Guide for Hostinger VPS

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Initial Server Setup](#initial-server-setup)
3. [Install Required Software](#install-required-software)
4. [Deploy Your Application](#deploy-your-application)
5. [Configure Nginx](#configure-nginx)
6. [Set Up Domain and SSL](#set-up-domain-and-ssl)
7. [Update Application Configuration](#update-application-configuration)
8. [Maintenance and Updates](#maintenance-and-updates)
9. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### What You Need:
- Hostinger VPS account with Ubuntu 24 (KVM 2 or higher)
- SSH access credentials (sent via email from Hostinger)
- Your MERN stack application code (GitHub repository or local files)
- Domain name (optional but recommended for SSL)

### Your VPS Information:
- **IP Address**: Found in Hostinger panel
- **SSH Port**: 22 (default)
- **Username**: root
- **Password**: Provided by Hostinger

---

## Initial Server Setup

### 1. Connect to Your VPS via SSH

**Windows (PowerShell or Command Prompt):**
```bash
ssh root@YOUR_VPS_IP
```

**Mac/Linux (Terminal):**
```bash
ssh root@YOUR_VPS_IP
```

Enter your password when prompted.

### 2. Update System Packages

```bash
apt update && apt upgrade -y
```

### 3. Set Up Firewall

```bash
# Allow SSH (CRITICAL - do this first!)
ufw allow 22/tcp

# Allow HTTP
ufw allow 80/tcp

# Allow HTTPS
ufw allow 443/tcp

# Enable firewall
ufw enable

# Check status
ufw status
```

When asked "Command may disrupt existing ssh connections. Proceed with operation (y|n)?", type `y` and press Enter.

---

## Install Required Software

### 1. Install Node.js and npm

```bash
# Install Node.js 20.x (LTS)
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# Verify installation
node -v
npm -v
```

### 2. Install Yarn (if your project uses it)

```bash
npm install -g yarn
yarn --version
```

### 3. Install MongoDB

```bash
# Import MongoDB public key
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | gpg --dearmor -o /usr/share/keyrings/mongodb-server-7.0.gpg

# Add MongoDB repository
echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Update package list
apt update

# Install MongoDB
apt install -y mongodb-org

# Start MongoDB
systemctl start mongod
systemctl enable mongod

# Verify MongoDB is running
systemctl status mongod
```

### 4. Install Nginx

```bash
apt install -y nginx
systemctl start nginx
systemctl enable nginx
systemctl status nginx
```

### 5. Install PM2 (Process Manager)

```bash
npm install -g pm2
```

### 6. Install Git

```bash
apt install -y git
git --version
```

---

## Deploy Your Application

### 1. Clone Your Repository

```bash
# Navigate to web directory
cd /var/www

# Clone your repository
git clone YOUR_REPOSITORY_URL

# Example:
# git clone https://github.com/yourusername/your-project.git

# Navigate to your project
cd YOUR_PROJECT_NAME
```

**OR Upload Files via SFTP:**
- Use FileZilla or WinSCP
- Connect to: `YOUR_VPS_IP`
- Port: `22`
- Username: `root`
- Upload files to: `/var/www/YOUR_PROJECT_NAME`

### 2. Set Up Backend

```bash
# Navigate to backend directory
cd /var/www/YOUR_PROJECT_NAME/server

# Install dependencies
yarn install
# or npm install

# Install bcrypt (if needed)
yarn add bcrypt
# or npm install bcrypt

# Create .env file
nano .env
```

**Add the following to .env:**
```env
PORT=3000
NODE_ENV=production
MONGODB_URI=mongodb://localhost:27017/YOUR_DATABASE_NAME
JWT_SECRET=your_strong_secret_key_here
JWT_EXPIRATION=1d

# Add any other environment variables your app needs
```

Save: `CTRL+X`, then `Y`, then `Enter`

### 3. Set Up Frontend

```bash
# Navigate to frontend directory
cd /var/www/YOUR_PROJECT_NAME/client

# Install dependencies
yarn install
# or npm install

# Create .env file
nano .env
```

**Add the following to .env (temporary - we'll update after domain setup):**
```env
VITE_API_BASE_URL=http://YOUR_VPS_IP/api
```

Save: `CTRL+X`, then `Y`, then `Enter`

**Build the frontend:**
```bash
yarn build
# or npm run build
```

This creates a `dist` folder (Vite) or `build` folder (Create React App)

---

## Configure MongoDB

### 1. Secure MongoDB with Authentication

```bash
# Open MongoDB shell
mongosh
```

**In MongoDB shell, run:**
```javascript
// Switch to admin database
use admin

// Create admin user
db.createUser({
  user: "admin",
  pwd: "YourStrongPassword123!",
  roles: ["root"]
})

// Switch to your application database
use YOUR_DATABASE_NAME

// Create application user
db.createUser({
  user: "appuser",
  pwd: "YourAppPassword123!",
  roles: ["readWrite"]
})

// Exit
exit
```

### 2. Enable MongoDB Authentication

```bash
nano /etc/mongod.conf
```

**Add/modify the security section:**
```yaml
security:
  authorization: enabled
```

Save and restart MongoDB:
```bash
systemctl restart mongod
```

### 3. Update Backend .env with Authentication

```bash
nano /var/www/YOUR_PROJECT_NAME/server/.env
```

**Update MongoDB URI:**
```env
MONGODB_URI=mongodb://appuser:YourAppPassword123%21@localhost:27017/YOUR_DATABASE_NAME?authSource=YOUR_DATABASE_NAME
```

Note: Special characters in passwords need URL encoding (`!` becomes `%21`, `@` becomes `%40`, etc.)

---

## Start Backend with PM2

```bash
cd /var/www/YOUR_PROJECT_NAME/server

# Start your backend (adjust filename if needed)
pm2 start index.js --name "your-app-backend"
# or: pm2 start server.js --name "your-app-backend"
# or: pm2 start app.js --name "your-app-backend"

# Save PM2 configuration
pm2 save

# Set PM2 to start on boot
pm2 startup

# Copy and run the command that PM2 outputs
```

**Check PM2 status:**
```bash
pm2 status
pm2 logs your-app-backend
```

---

## Configure Nginx

### 1. Create Nginx Configuration File

```bash
nano /etc/nginx/sites-available/your-app
```

**For Vite projects (uses `dist` folder):**
```nginx
server {
    listen 80;
    server_name YOUR_VPS_IP;  # We'll change this to domain later
    
    # Frontend (React build)
    location / {
        root /var/www/YOUR_PROJECT_NAME/client/dist;
        try_files $uri $uri/ /index.html;
    }
    
    # Backend API
    location /api {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**For Create React App projects (uses `build` folder):**
Replace `/var/www/YOUR_PROJECT_NAME/client/dist` with `/var/www/YOUR_PROJECT_NAME/client/build`

Save: `CTRL+X`, then `Y`, then `Enter`

### 2. Enable the Site

```bash
# Create symbolic link
ln -s /etc/nginx/sites-available/your-app /etc/nginx/sites-enabled/

# Remove default site
rm /etc/nginx/sites-enabled/default

# Test configuration
nginx -t

# Restart Nginx
systemctl restart nginx
```

### 3. Set Proper Permissions

```bash
# For Vite projects
chown -R www-data:www-data /var/www/YOUR_PROJECT_NAME/client/dist
chmod -R 755 /var/www/YOUR_PROJECT_NAME/client/dist

# For Create React App
# chown -R www-data:www-data /var/www/YOUR_PROJECT_NAME/client/build
# chmod -R 755 /var/www/YOUR_PROJECT_NAME/client/build
```

### 4. Test Your Site

Visit `http://YOUR_VPS_IP` in your browser. Your site should be live!

---

## Set Up Domain and SSL

### 1. Configure DNS Records in Hostinger

1. Go to Hostinger control panel
2. Navigate to: **Domains** → **DNS / Nameservers**
3. Add these DNS records:

**A Record for root domain:**
- Type: `A`
- Name: `@` (or leave blank)
- Points to: `YOUR_VPS_IP`
- TTL: `14400`

**A Record for www:**
- Type: `A`
- Name: `www`
- Points to: `YOUR_VPS_IP`
- TTL: `14400`

**Delete any conflicting CNAME records for www**

### 2. Update Nginx Configuration for Domain

```bash
nano /etc/nginx/sites-available/your-app
```

**Change the server_name line:**
```nginx
server_name yourdomain.com www.yourdomain.com;
```

Save and restart:
```bash
nginx -t
systemctl restart nginx
```

### 3. Wait for DNS Propagation

DNS changes take 5-60 minutes to propagate. Check status at:
- https://www.whatsmydns.net/#A/yourdomain.com

Or from your server:
```bash
nslookup yourdomain.com
```

### 4. Install SSL Certificate (Free with Let's Encrypt)

**Install Certbot:**
```bash
apt update
apt install -y certbot python3-certbot-nginx
```

**Get SSL Certificate:**
```bash
certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Follow the prompts:
1. Enter your email address
2. Agree to terms (type `Y`)
3. Share email with EFF (optional - type `Y` or `N`)
4. Choose option `2` to redirect HTTP to HTTPS

**Verify Auto-Renewal:**
```bash
certbot renew --dry-run
```

Your site is now secured with HTTPS! 🔒

---

## Update Application Configuration

### 1. Update Frontend Environment Variables

```bash
nano /var/www/YOUR_PROJECT_NAME/client/.env
```

**Change to:**
```env
VITE_API_BASE_URL=https://yourdomain.com/api
```

### 2. Update Backend CORS Configuration

```bash
nano /var/www/YOUR_PROJECT_NAME/server/src/config/index.js
```

**Update CORS to include your domain:**
```javascript
corsOptions: {
    origin: [
        'http://localhost:5173',
        'https://yourdomain.com',
        'https://www.yourdomain.com'
    ],
    credentials: true, 
    methods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allowedHeaders: ["Content-Type", "Authorization", "Cookie"],
}
```

**Or use environment variable approach:**

Update your config to:
```javascript
corsOptions: {
    origin: process.env.CORS_ORIGIN?.split(',') || 'http://localhost:5173',
    credentials: true,
    methods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allowedHeaders: ["Content-Type", "Authorization", "Cookie"],
}
```

Then update your backend `.env`:
```bash
nano /var/www/YOUR_PROJECT_NAME/server/.env
```

Add:
```env
CORS_ORIGIN=https://yourdomain.com,https://www.yourdomain.com
```

### 3. Rebuild and Restart Everything

```bash
# Rebuild frontend
cd /var/www/YOUR_PROJECT_NAME/client
yarn build

# Set permissions
chown -R www-data:www-data dist
chmod -R 755 dist

# Restart backend
pm2 restart your-app-backend

# Restart Nginx
systemctl restart nginx

# Check status
pm2 status
systemctl status nginx
```

---

## Import Your Database Data

### Method 1: Using MongoDB Compass (GUI)

1. **Enable Remote MongoDB Access (temporarily):**
```bash
ufw allow 27017/tcp
```

2. **Connect from MongoDB Compass:**
   - Connection string: `mongodb://admin:YourStrongPassword123!@YOUR_VPS_IP:27017/?authSource=admin`

3. **Import your data:**
   - Click your database → collection
   - Click "ADD DATA" → "Import JSON or CSV file"
   - Select your file and import

4. **Close port after import (recommended):**
```bash
ufw delete allow 27017/tcp
```

### Method 2: Using Command Line

**Upload your JSON file:**
```bash
# From your local machine
scp /path/to/data.json root@YOUR_VPS_IP:/tmp/
```

**Import on server:**
```bash
# For JSON array
mongoimport --db YOUR_DATABASE_NAME --collection YOUR_COLLECTION_NAME --file /tmp/data.json --jsonArray --authenticationDatabase admin -u admin -p

# For MongoDB dump
mongorestore --db YOUR_DATABASE_NAME /path/to/dump/YOUR_DATABASE_NAME/ --authenticationDatabase admin -u admin -p
```

---

## Maintenance and Updates

### Deploy Script (Automated Updates)

Create a deployment script for easy updates:

```bash
nano /var/www/deploy.sh
```

**Add this content:**
```bash
#!/bin/bash

echo "=== Starting Deployment ==="

# Navigate to project
cd /var/www/YOUR_PROJECT_NAME

# Pull latest changes
echo "Pulling latest code..."
git pull origin main

# Update Backend
echo "Updating backend..."
cd server
yarn install
pm2 restart your-app-backend

# Update Frontend
echo "Updating frontend..."
cd ../client
yarn install
yarn build
chown -R www-data:www-data dist
chmod -R 755 dist

# Restart services
echo "Restarting services..."
systemctl restart nginx

echo "=== Deployment Complete ==="
pm2 status
```

**Make it executable:**
```bash
chmod +x /var/www/deploy.sh
```

**Run it anytime:**
```bash
/var/www/deploy.sh
```

### Common PM2 Commands

```bash
# View all processes
pm2 status

# View logs
pm2 logs your-app-backend
pm2 logs your-app-backend --lines 100

# Restart application
pm2 restart your-app-backend

# Stop application
pm2 stop your-app-backend

# Delete application from PM2
pm2 delete your-app-backend

# Monitor in real-time
pm2 monit
```

### Nginx Commands

```bash
# Test configuration
nginx -t

# Reload configuration (no downtime)
systemctl reload nginx

# Restart Nginx
systemctl restart nginx

# Check status
systemctl status nginx

# View error logs
tail -f /var/log/nginx/error.log

# View access logs
tail -f /var/log/nginx/access.log
```

### MongoDB Commands

```bash
# Check status
systemctl status mongod

# Restart MongoDB
systemctl restart mongod

# View logs
tail -f /var/log/mongodb/mongod.log

# Access MongoDB shell
mongosh "mongodb://admin:YourPassword@localhost:27017/?authSource=admin"
```

---

## Troubleshooting

### Backend Issues

**Problem: PM2 shows "errored" status**

```bash
# Check logs
pm2 logs your-app-backend --lines 50

# Common fixes:
# 1. Check .env file exists
cat /var/www/YOUR_PROJECT_NAME/server/.env

# 2. Check MongoDB is running
systemctl status mongod

# 3. Check port is not in use
netstat -tulpn | grep :3000

# 4. Reinstall dependencies
cd /var/www/YOUR_PROJECT_NAME/server
rm -rf node_modules
yarn install

# 5. Restart PM2
pm2 restart your-app-backend
```

**Problem: "Cannot find module" errors**

```bash
# Install missing module
cd /var/www/YOUR_PROJECT_NAME/server
yarn add MODULE_NAME

# Or reinstall all dependencies
rm -rf node_modules package-lock.json yarn.lock
yarn install

pm2 restart your-app-backend
```

### Frontend Issues

**Problem: 500 Internal Server Error**

```bash
# Check if build folder exists
ls -la /var/www/YOUR_PROJECT_NAME/client/dist

# If missing, rebuild
cd /var/www/YOUR_PROJECT_NAME/client
yarn build

# Set permissions
chown -R www-data:www-data dist
chmod -R 755 dist

systemctl restart nginx
```

**Problem: API calls failing (CORS errors)**

```bash
# Check backend CORS configuration
nano /var/www/YOUR_PROJECT_NAME/server/src/config/index.js

# Ensure your domain is in the origin array
# Restart backend
pm2 restart your-app-backend
```

**Problem: Frontend shows old version**

```bash
# Rebuild frontend
cd /var/www/YOUR_PROJECT_NAME/client
yarn build

# Clear browser cache
# Press CTRL+SHIFT+R in browser
```

### Nginx Issues

**Problem: 502 Bad Gateway**

```bash
# Check if backend is running
pm2 status

# Check backend logs
pm2 logs your-app-backend

# Restart backend
pm2 restart your-app-backend

# Restart Nginx
systemctl restart nginx
```

**Problem: 403 Forbidden**

```bash
# Fix permissions
chown -R www-data:www-data /var/www/YOUR_PROJECT_NAME/client/dist
chmod -R 755 /var/www/YOUR_PROJECT_NAME/client/dist

systemctl restart nginx
```

### SSL Issues

**Problem: SSL certificate failed**

```bash
# Check DNS propagation first
nslookup yourdomain.com

# Ensure ports 80 and 443 are open
ufw allow 80/tcp
ufw allow 443/tcp

# Try again
certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Check Certbot logs
tail -50 /var/log/letsencrypt/letsencrypt.log
```

**Problem: SSL certificate expired**

```bash
# Renew manually
certbot renew

# Check auto-renewal
certbot renew --dry-run

# Restart Nginx
systemctl restart nginx
```

### MongoDB Issues

**Problem: Authentication failed**

```bash
# Test connection
mongosh "mongodb://admin:YourPassword@localhost:27017/?authSource=admin"

# If password is wrong, reset it:
# 1. Stop MongoDB
systemctl stop mongod

# 2. Edit config to disable auth temporarily
nano /etc/mongod.conf
# Comment out: # authorization: enabled

# 3. Restart MongoDB
systemctl restart mongod

# 4. Reset password
mongosh
use admin
db.changeUserPassword("admin", "NewPassword123!")
exit

# 5. Re-enable auth
nano /etc/mongod.conf
# Uncomment: authorization: enabled

# 6. Restart
systemctl restart mongod
```

### Database Issues

**Problem: Empty collections after import**

```bash
# Check if data was imported
mongosh "mongodb://admin:YourPassword@localhost:27017/?authSource=admin"
use YOUR_DATABASE_NAME
db.YOUR_COLLECTION.countDocuments()

# If zero, reimport with correct flags
mongoimport --db YOUR_DATABASE_NAME --collection YOUR_COLLECTION --file /tmp/data.json --jsonArray --authenticationDatabase admin -u admin -p
```

---

## Security Best Practices

### 1. Change Default SSH Port (Optional)

```bash
nano /etc/ssh/sshd_config

# Change Port 22 to something else (e.g., 2222)
# Add: Port 2222

systemctl restart sshd

# Update firewall
ufw allow 2222/tcp
ufw delete allow 22/tcp
```

### 2. Disable Root Login (After creating sudo user)

```bash
# Create new user
adduser yourusername
usermod -aG sudo yourusername

# Test login as new user first!
# Then disable root login:
nano /etc/ssh/sshd_config
# Change: PermitRootLogin no

systemctl restart sshd
```

### 3. Keep System Updated

```bash
# Update packages regularly
apt update && apt upgrade -y

# Auto-update security patches
apt install unattended-upgrades
dpkg-reconfigure --priority=low unattended-upgrades
```

### 4. Monitor Server Resources

```bash
# Check disk space
df -h

# Check memory usage
free -h

# Check CPU usage
top
# Press 'q' to quit

# Check running processes
ps aux | grep node
```

### 5. Set Up Backup (Important!)

```bash
# Create backup directory
mkdir -p /var/backups/mongodb

# Manual MongoDB backup
mongodump --db YOUR_DATABASE_NAME --out /var/backups/mongodb/$(date +%Y%m%d) --authenticationDatabase admin -u admin -p

# Create automated backup script
nano /root/backup.sh
```

**Add to backup.sh:**
```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/var/backups/mongodb"
mkdir -p $BACKUP_DIR

# Backup MongoDB
mongodump --db YOUR_DATABASE_NAME --out $BACKUP_DIR/$DATE --authenticationDatabase admin -u admin -p YOUR_PASSWORD

# Keep only last 7 days of backups
find $BACKUP_DIR -type d -mtime +7 -exec rm -rf {} +

echo "Backup completed: $DATE"
```

Make executable and add to cron:
```bash
chmod +x /root/backup.sh

# Edit crontab
crontab -e

# Add daily backup at 2 AM
0 2 * * * /root/backup.sh >> /var/log/mongodb-backup.log 2>&1
```

---

## Performance Optimization

### 1. Enable Gzip Compression in Nginx

```bash
nano /etc/nginx/nginx.conf
```

**Add inside http block:**
```nginx
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss application/json application/javascript;
```

Restart Nginx:
```bash
systemctl restart nginx
```

### 2. Enable PM2 Cluster Mode

```bash
# Stop current app
pm2 delete your-app-backend

# Start in cluster mode (uses all CPU cores)
pm2 start index.js --name "your-app-backend" -i max

pm2 save
```

### 3. Add MongoDB Indexes

```bash
mongosh "mongodb://admin:YourPassword@localhost:27017/?authSource=admin"
```

```javascript
use YOUR_DATABASE_NAME

// Add indexes based on your queries
// Example: Index on email field
db.users.createIndex({ email: 1 })

// Example: Compound index
db.appointments.createIndex({ date: 1, status: 1 })
```

---

## Monitoring and Logging

### Set Up PM2 Monitoring

```bash
# View real-time monitoring
pm2 monit

# Enable PM2 web dashboard
pm2 web
# Access at: http://YOUR_VPS_IP:9615
```

### Centralized Logging

```bash
# View all PM2 logs
pm2 logs

# View logs for specific app
pm2 logs your-app-backend --lines 200

# Save logs to file
pm2 logs your-app-backend --out /var/log/app-out.log --error /var/log/app-error.log
```

---

## Quick Reference Commands

### Full Deployment Checklist

```bash
# 1. Pull latest code
cd /var/www/YOUR_PROJECT_NAME
git pull origin main

# 2. Update backend
cd server
yarn install
pm2 restart your-app-backend

# 3. Update frontend
cd ../client
yarn install
yarn build
chown -R www-data:www-data dist
chmod -R 755 dist

# 4. Restart services
systemctl restart nginx

# 5. Verify
pm2 status
systemctl status nginx
curl -I https://yourdomain.com
```

### Health Check Commands

```bash
# Check all services
systemctl status nginx
systemctl status mongod
pm2 status

# Check ports
netstat -tulpn | grep LISTEN

# Check disk space
df -h

# Check memory
free -h

# Check logs
pm2 logs --lines 50
tail -f /var/log/nginx/error.log
```

---

## Additional Resources

### Official Documentation
- **Node.js**: https://nodejs.org/docs
- **MongoDB**: https://docs.mongodb.com
- **Nginx**: https://nginx.org/en/docs/
- **PM2**: https://pm2.keymetrics.io/docs
- **Let's Encrypt**: https://letsencrypt.org/docs/

### Useful Tools
- **MongoDB Compass**: https://www.mongodb.com/products/compass
- **Postman**: https://www.postman.com (API testing)
- **FileZilla**: https://filezilla-project.org (SFTP client)

### Hostinger Resources
- **Knowledge Base**: https://support.hostinger.com
- **Control Panel**: https://hpanel.hostinger.com

---

## Conclusion

You now have a fully deployed MERN stack application with:
- ✅ Secure HTTPS connection
- ✅ Automated process management with PM2
- ✅ Reverse proxy with Nginx
- ✅ Secured MongoDB with authentication
- ✅ Auto-renewal SSL certificates
- ✅ Production-ready configuration

### Need Help?

If you encounter issues:
1. Check the [Troubleshooting](#troubleshooting) section
2. Review application logs: `pm2 logs`
3. Check Nginx logs: `tail -f /var/log/nginx/error.log`
4. Verify all services are running: `pm2 status`, `systemctl status nginx`, `systemctl status mongod`

---

**Last Updated**: February 2026  
**Version**: 1.0

Happy Deploying! 🚀
