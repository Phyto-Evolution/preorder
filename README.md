# TC Plants India - Tissue Culture Laboratory Platform

A specialized tissue culture laboratory platform for pre-ordering carnivorous plants, orchids, exotic tropical fruits, and rare genetics. Built with modern web technologies and deployed on Cloudflare.

## Features

- **Plant Catalog**: Browse available tissue culture plants with detailed information
- **Bulk Pre-Orders**: Customers can pre-order plants with no upfront payment
- **Order Tracking**: Real-time tracking of order progress through production stages
- **Admin Dashboard**: Complete plant management and order processing system
- **Responsive Design**: Mobile-friendly interface with modern UI

## Tech Stack

- **Frontend**: React 19, TypeScript, Tailwind CSS, Vite
- **Backend**: Hono.js (Edge Runtime)
- **Database**: Cloudflare D1 (SQLite)
- **Deployment**: Cloudflare Workers
- **Icons**: Lucide React

## Local Development

1. **Clone the repository**
   ```bash
   git clone <your-repo-url>
   cd tc-plants-india
   ```

2. **Install dependencies**
   ```bash
   npm install
   ```

3. **Set up Cloudflare D1 database locally**
   ```bash
   # Create a local D1 database
   npx wrangler d1 create tc-plants-db
   
   # Update wrangler.jsonc with your database ID
   # Run migrations locally
   npx wrangler d1 migrations apply tc-plants-db --local
   ```

4. **Start development server**
   ```bash
   npm run dev
   ```

5. **Access the application**
   - Frontend: `http://localhost:5173`
   - Admin Panel: `http://localhost:5173/admin`
   - Order Tracking: `http://localhost:5173/track`

## Ubuntu VPS Deployment Guide

### Prerequisites

1. **Ubuntu Server** (20.04 LTS or later)
2. **Node.js 18+** and **npm**
3. **Domain name** pointed to your VPS
4. **Cloudflare account** with D1 database

### Step 1: Server Setup

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install Node.js (using NodeSource repository)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install PM2 for process management
sudo npm install -g pm2

# Install Nginx for reverse proxy
sudo apt install nginx -y

# Install certbot for SSL certificates
sudo apt install certbot python3-certbot-nginx -y
```

### Step 2: Application Deployment

```bash
# Create application directory
sudo mkdir -p /var/www/tc-plants
sudo chown $USER:$USER /var/www/tc-plants

# Clone your repository
cd /var/www/tc-plants
git clone <your-repo-url> .

# Install dependencies
npm install

# Build the application
npm run build
```

### Step 3: Cloudflare D1 Database Setup

```bash
# Install Wrangler CLI globally
npm install -g wrangler

# Login to Cloudflare
wrangler login

# Create production D1 database
wrangler d1 create tc-plants-production

# Note the database ID and update wrangler.jsonc
# Run migrations on production database
wrangler d1 migrations apply tc-plants-production
```

### Step 4: Environment Configuration

Create a `.env` file in your project root:

```bash
# .env
NODE_ENV=production
PORT=3000
DATABASE_URL=<your-d1-database-url>
```

### Step 5: Process Management with PM2

```bash
# Create PM2 ecosystem file
cat > ecosystem.config.js << EOF
module.exports = {
  apps: [{
    name: 'tc-plants',
    script: 'npm',
    args: 'start',
    cwd: '/var/www/tc-plants',
    instances: 'max',
    exec_mode: 'cluster',
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    },
    error_file: '/var/log/tc-plants/error.log',
    out_file: '/var/log/tc-plants/access.log',
    log_file: '/var/log/tc-plants/combined.log'
  }]
}
EOF

# Create log directory
sudo mkdir -p /var/log/tc-plants
sudo chown $USER:$USER /var/log/tc-plants

# Start application with PM2
pm2 start ecosystem.config.js

# Save PM2 configuration
pm2 save

# Setup PM2 to start on boot
pm2 startup
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u $USER --hp $HOME
```

### Step 6: Nginx Configuration

```bash
# Create Nginx site configuration
sudo tee /etc/nginx/sites-available/tc-plants << EOF
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        proxy_cache_bypass \$http_upgrade;
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied expired no-cache no-store private must-revalidate auth;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss application/javascript;
}
EOF

# Enable the site
sudo ln -s /etc/nginx/sites-available/tc-plants /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default

# Test Nginx configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
```

### Step 7: SSL Certificate with Let's Encrypt

```bash
# Obtain SSL certificate
sudo certbot --nginx -d your-domain.com -d www.your-domain.com

# Test auto-renewal
sudo certbot renew --dry-run
```

### Step 8: Firewall Configuration

```bash
# Enable UFW firewall
sudo ufw enable

# Allow SSH, HTTP, and HTTPS
sudo ufw allow ssh
sudo ufw allow 'Nginx Full'

# Check firewall status
sudo ufw status
```

### Step 9: Cloudflare Workers Deployment (Alternative)

For better performance, you can deploy directly to Cloudflare Workers:

```bash
# Deploy to Cloudflare Workers
wrangler deploy

# Set up custom domain in Cloudflare dashboard
# Add your domain as a route in Workers settings
```

### Step 10: Monitoring and Maintenance

```bash
# Monitor application logs
pm2 logs tc-plants

# Monitor system resources
pm2 monit

# Update application
cd /var/www/tc-plants
git pull origin main
npm install
npm run build
pm2 restart tc-plants

# Database backup (create a script)
cat > backup-db.sh << EOF
#!/bin/bash
DATE=\$(date +%Y%m%d_%H%M%S)
wrangler d1 export tc-plants-production --output=/var/backups/tc-plants-\$DATE.sql
find /var/backups -name "tc-plants-*.sql" -mtime +7 -delete
EOF

chmod +x backup-db.sh

# Add to crontab for daily backups
echo "0 2 * * * /var/www/tc-plants/backup-db.sh" | crontab -
```

## Production Checklist

- [ ] Server updated and secured
- [ ] Domain DNS pointing to server
- [ ] SSL certificate installed
- [ ] Database migrations applied
- [ ] Environment variables configured
- [ ] PM2 process running
- [ ] Nginx configured and running
- [ ] Firewall enabled
- [ ] Backup script set up
- [ ] Monitoring configured

## Common Commands

```bash
# Application management
pm2 restart tc-plants    # Restart application
pm2 stop tc-plants       # Stop application
pm2 logs tc-plants       # View logs
pm2 status              # Check status

# Database operations
wrangler d1 migrations list tc-plants-production    # List migrations
wrangler d1 query tc-plants-production "SELECT * FROM plants LIMIT 5"  # Query database

# Server maintenance
sudo nginx -t           # Test Nginx config
sudo systemctl reload nginx  # Reload Nginx
sudo certbot renew     # Renew SSL certificates
```

## Support

For deployment issues or questions, check:
- Application logs: `pm2 logs tc-plants`
- Nginx logs: `sudo tail -f /var/log/nginx/error.log`
- System logs: `sudo journalctl -f`

## License

This project is proprietary software for TC Plants India.
