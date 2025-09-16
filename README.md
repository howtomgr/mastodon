# Mastodon Installation Guide

Mastodon is a free, open-source federated social network server that provides a decentralized alternative to proprietary platforms like Twitter/X. Built on Ruby on Rails and using ActivityPub protocol, Mastodon allows users to run their own social media instance while connecting to the broader fediverse.

## Prerequisites

### Hardware Requirements
- **CPU**: 2+ cores (4+ cores recommended for production)
- **RAM**: 4GB minimum (8GB+ recommended for production)
- **Storage**: 20GB minimum (SSD recommended, scale based on media storage needs)
- **Network**: Reliable internet connection with public IP (for federation)

### Software Dependencies
- **Operating System**: Linux (RHEL-based, Debian-based, Arch Linux, Alpine Linux)
- **Database**: PostgreSQL 12+ (PostgreSQL 15+ recommended)
- **Cache/Queue**: Redis 6.0+
- **Runtime**: Ruby 3.0+ (Ruby 3.2+ recommended)
- **Web Server**: nginx or Apache (for reverse proxy)
- **SSL Certificate**: Let's Encrypt or commercial certificate
- **Node.js**: 16+ (for asset compilation)
- **Yarn**: Package manager for Node.js dependencies

### Network Requirements
- **Ports**: 
  - 80/443 (HTTP/HTTPS)
  - 25/587 (SMTP - for email delivery)
  - 3000 (Mastodon web interface)
  - 4000 (Mastodon streaming API)
- **Domain**: Registered domain name with DNS control
- **Email**: SMTP server for notifications and user registration

### System Access
- Root or sudo access required for installation
- Dedicated user account for Mastodon service
- Firewall configuration access

## Supported Operating Systems

This guide supports installation on:
- **RHEL-based**: RHEL 8/9, CentOS Stream 8/9, Rocky Linux 8/9, AlmaLinux 8/9, Fedora 37+
- **Debian-based**: Debian 11/12, Ubuntu 20.04/22.04/24.04 LTS
- **Arch Linux**: Arch Linux (rolling), Manjaro
- **Alpine Linux**: 3.18+ (containerized deployments)
- **macOS**: 12+ (development only)
- **FreeBSD**: 13.0+ (experimental)

## Installation

### RHEL/CentOS/Rocky Linux/AlmaLinux

```bash
# Install EPEL and PowerTools repositories
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled powertools  # CentOS Stream 8
sudo dnf config-manager --set-enabled crb         # CentOS Stream 9, Rocky, AlmaLinux

# Install Node.js repository
curl -fsSL https://rpm.nodesource.com/setup_lts.x | sudo bash -

# Install Yarn repository
curl -sL https://dl.yarnpkg.com/rpm/yarn.repo | sudo tee /etc/yum.repos.d/yarn.repo

# Install PostgreSQL repository (for latest version)
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-$(rpm -E %{rhel})-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Install required packages
sudo dnf install -y \
    git curl wget gnupg2 \
    gcc gcc-c++ make automake autoconf libtool \
    postgresql15-server postgresql15-devel \
    redis \
    nodejs yarn \
    nginx \
    ImageMagick ImageMagick-devel \
    ffmpeg \
    libxml2-devel libxslt-devel \
    libidn-devel \
    openssl-devel \
    readline-devel \
    zlib-devel \
    libyaml-devel \
    gdbm-devel \
    ncurses-devel \
    libffi-devel

# Initialize PostgreSQL
sudo postgresql-15-setup initdb
sudo systemctl enable --now postgresql-15 redis

# Configure PostgreSQL
sudo -u postgres createuser mastodon --createdb
sudo -u postgres psql -c "ALTER USER mastodon WITH ENCRYPTED PASSWORD 'secure_random_password_here';"

# Create mastodon user
sudo useradd -m -s /bin/bash mastodon
sudo usermod -a -G mastodon nginx

# Install Ruby (using rbenv for version management)
sudo -u mastodon bash << 'EOF'
cd /home/mastodon
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
rbenv install 3.2.2
rbenv global 3.2.2
gem install bundler
EOF

# Clone Mastodon
sudo -u mastodon git clone https://github.com/mastodon/mastodon.git /home/mastodon/live
cd /home/mastodon/live
sudo -u mastodon git checkout $(git tag -l | grep -v 'rc' | sort -V | tail -n 1)

# Install Ruby dependencies
sudo -u mastodon bash << 'EOF'
cd /home/mastodon/live
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
bundle config deployment 'true'
bundle config without 'development test'
bundle install
EOF

# Install Node.js dependencies
sudo -u mastodon bash << 'EOF'
cd /home/mastodon/live
yarn install --pure-lockfile
EOF
```

### Debian/Ubuntu

```bash
# Update package list
sudo apt update

# Install curl and gnupg (if not already installed)
sudo apt install -y curl gnupg

# Add Node.js repository
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -

# Add Yarn repository
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list

# Add PostgreSQL repository (for latest version)
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list

# Update package list
sudo apt update

# Install required packages
sudo apt install -y \
    git curl wget gnupg \
    build-essential \
    postgresql-15 postgresql-contrib-15 postgresql-server-dev-15 \
    redis-server \
    nodejs yarn \
    nginx \
    imagemagick ffmpeg \
    libxml2-dev libxslt1-dev \
    libidn11-dev \
    libssl-dev \
    libreadline-dev \
    zlib1g-dev \
    libyaml-dev \
    libgdbm-dev \
    libncurses5-dev \
    libffi-dev \
    libpq-dev

# Start and enable services
sudo systemctl enable --now postgresql redis-server

# Configure PostgreSQL
sudo -u postgres createuser mastodon --createdb
sudo -u postgres psql -c "ALTER USER mastodon WITH ENCRYPTED PASSWORD 'secure_random_password_here';"

# Create mastodon user
sudo useradd -m -s /bin/bash mastodon
sudo usermod -a -G mastodon www-data

# Install Ruby (using rbenv)
sudo -u mastodon bash << 'EOF'
cd /home/mastodon
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
rbenv install 3.2.2
rbenv global 3.2.2
gem install bundler
EOF

# Clone Mastodon
sudo -u mastodon git clone https://github.com/mastodon/mastodon.git /home/mastodon/live
cd /home/mastodon/live
sudo -u mastodon git checkout $(git tag -l | grep -v 'rc' | sort -V | tail -n 1)

# Install dependencies
sudo -u mastodon bash << 'EOF'
cd /home/mastodon/live
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
bundle config deployment 'true'
bundle config without 'development test'
bundle install
yarn install --pure-lockfile
EOF
```

### Arch Linux

```bash
# Update system
sudo pacman -Syu

# Install required packages
sudo pacman -S --needed \
    git curl wget gnupg \
    base-devel \
    postgresql redis \
    nodejs npm yarn \
    nginx \
    imagemagick ffmpeg \
    libxml2 libxslt \
    libidn \
    openssl \
    readline \
    zlib \
    libyaml \
    gdbm \
    ncurses \
    libffi

# Initialize PostgreSQL
sudo -u postgres initdb -D /var/lib/postgres/data
sudo systemctl enable --now postgresql redis

# Configure PostgreSQL
sudo -u postgres createuser mastodon --createdb
sudo -u postgres psql -c "ALTER USER mastodon WITH ENCRYPTED PASSWORD 'secure_random_password_here';"

# Create mastodon user
sudo useradd -m -s /bin/bash mastodon
sudo usermod -a -G mastodon http

# Install Ruby using rbenv
sudo -u mastodon bash << 'EOF'
cd /home/mastodon
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
rbenv install 3.2.2
rbenv global 3.2.2
gem install bundler
EOF

# Clone and setup Mastodon
sudo -u mastodon git clone https://github.com/mastodon/mastodon.git /home/mastodon/live
cd /home/mastodon/live
sudo -u mastodon git checkout $(git tag -l | grep -v 'rc' | sort -V | tail -n 1)

# Install dependencies
sudo -u mastodon bash << 'EOF'
cd /home/mastodon/live
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
bundle config deployment 'true'
bundle config without 'development test'
bundle install
yarn install --pure-lockfile
EOF
```

### Alpine Linux

```bash
# Update package index
apk update

# Install required packages
apk add --no-cache \
    git curl wget gnupg \
    build-base \
    postgresql15 postgresql15-dev postgresql15-contrib \
    redis \
    nodejs npm yarn \
    nginx \
    imagemagick imagemagick-dev ffmpeg \
    libxml2-dev libxslt-dev \
    libidn-dev \
    openssl-dev \
    readline-dev \
    zlib-dev \
    yaml-dev \
    gdbm-dev \
    ncurses-dev \
    libffi-dev

# Initialize and start services
rc-service postgresql setup
rc-service postgresql start
rc-service redis start
rc-update add postgresql
rc-update add redis

# Configure PostgreSQL
sudo -u postgres createuser mastodon --createdb
sudo -u postgres psql -c "ALTER USER mastodon WITH ENCRYPTED PASSWORD 'secure_random_password_here';"

# Create mastodon user
adduser -D -s /bin/ash mastodon
addgroup mastodon nginx

# Install Ruby using rbenv
sudo -u mastodon ash << 'EOF'
cd /home/mastodon
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.profile
echo 'eval "$(rbenv init -)"' >> ~/.profile
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
rbenv install 3.2.2
rbenv global 3.2.2
gem install bundler
EOF

# Clone and setup Mastodon
sudo -u mastodon git clone https://github.com/mastodon/mastodon.git /home/mastodon/live
cd /home/mastodon/live
sudo -u mastodon git checkout $(git tag -l | grep -v 'rc' | sort -V | tail -n 1)

# Install dependencies
sudo -u mastodon ash << 'EOF'
cd /home/mastodon/live
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
bundle config deployment 'true'
bundle config without 'development test'
bundle install
yarn install --pure-lockfile
EOF
```

## Configuration

### Environment Configuration

```bash
# Switch to mastodon user
sudo -u mastodon bash

# Navigate to Mastodon directory
cd /home/mastodon/live

# Generate configuration
RAILS_ENV=production bundle exec rake mastodon:setup
```

This interactive setup will prompt for:
- Domain name (e.g., mastodon.example.com)
- Database configuration
- Redis configuration
- Email settings (SMTP)
- File storage settings
- Admin account creation

### Manual Environment Configuration

If you prefer manual configuration, create `/home/mastodon/live/.env.production`:

```bash
# Create environment file
sudo -u mastodon tee /home/mastodon/live/.env.production << 'EOF'
# Federation
LOCAL_DOMAIN=mastodon.example.com
WEB_DOMAIN=mastodon.example.com

# Redis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379

# PostgreSQL
DB_HOST=127.0.0.1
DB_USER=mastodon
DB_NAME=mastodon_production
DB_PASS=secure_random_password_here
DB_PORT=5432

# ElasticSearch (optional, for full-text search)
# ES_ENABLED=true
# ES_HOST=localhost
# ES_PORT=9200

# Secrets
# Generate with: bundle exec rake secret
SECRET_KEY_BASE=generate_with_rake_secret
OTP_SECRET=generate_with_rake_secret

# VAPID keys (for push notifications)
# Generate with: bundle exec rake mastodon:webpush:generate_vapid_key
VAPID_PRIVATE_KEY=
VAPID_PUBLIC_KEY=

# Sending mail
SMTP_SERVER=smtp.example.com
SMTP_PORT=587
SMTP_LOGIN=mastodon@example.com
SMTP_PASSWORD=smtp_password
SMTP_FROM_ADDRESS=mastodon@example.com

# File storage (local)
PAPERCLIP_ROOT_PATH=/home/mastodon/live/public/system

# Optional S3/compatible storage
# S3_ENABLED=true
# S3_BUCKET=mastodon
# AWS_ACCESS_KEY_ID=
# AWS_SECRET_ACCESS_KEY=
# S3_REGION=us-east-1
# S3_HOSTNAME=s3.amazonaws.com

# Optional CDN
# CDN_HOST=assets.example.com

# Streaming
STREAMING_CLUSTER_NUM=1
STREAMING_API_BASE_URL=wss://mastodon.example.com

# Advanced settings
MAX_TOOT_CHARS=500
SINGLE_USER_MODE=false
AUTHORIZED_FETCH=false
LIMITED_FEDERATION_MODE=false
EOF
```

### Database Setup

```bash
# Switch to mastodon user and setup database
sudo -u mastodon bash << 'EOF'
cd /home/mastodon/live
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"

# Create database
RAILS_ENV=production bundle exec rails db:create
RAILS_ENV=production bundle exec rails db:schema:load
RAILS_ENV=production bundle exec rails db:seed

# Compile assets
RAILS_ENV=production bundle exec rails assets:precompile
EOF
```

## Service Management

### Systemd Service Files (RHEL/Debian/Arch/SUSE)

Create the web service (`/etc/systemd/system/mastodon-web.service`):

```ini
[Unit]
Description=mastodon-web
After=network.target

[Service]
Type=simple
User=mastodon
WorkingDirectory=/home/mastodon/live
Environment="RAILS_ENV=production"
Environment="PORT=3000"
ExecStart=/home/mastodon/.rbenv/shims/bundle exec puma -C config/puma.rb
ExecReload=/bin/kill -SIGUSR1 $MAINPID
TimeoutSec=15
Restart=always
RestartSec=10
SyslogIdentifier=mastodon-web

[Install]
WantedBy=multi-user.target
```

Create the background jobs service (`/etc/systemd/system/mastodon-sidekiq.service`):

```ini
[Unit]
Description=mastodon-sidekiq
After=network.target

[Service]
Type=simple
User=mastodon
WorkingDirectory=/home/mastodon/live
Environment="RAILS_ENV=production"
Environment="DB_POOL=25"
ExecStart=/home/mastodon/.rbenv/shims/bundle exec sidekiq -c 25
TimeoutSec=15
Restart=always
RestartSec=10
SyslogIdentifier=mastodon-sidekiq

[Install]
WantedBy=multi-user.target
```

Create the streaming service (`/etc/systemd/system/mastodon-streaming.service`):

```ini
[Unit]
Description=mastodon-streaming
After=network.target

[Service]
Type=simple
User=mastodon
WorkingDirectory=/home/mastodon/live
Environment="NODE_ENV=production"
Environment="PORT=4000"
ExecStart=/usr/bin/node streaming
TimeoutSec=15
Restart=always
RestartSec=10
SyslogIdentifier=mastodon-streaming

[Install]
WantedBy=multi-user.target
```

### Enable and Start Services

```bash
# Reload systemd and enable services
sudo systemctl daemon-reload
sudo systemctl enable mastodon-web mastodon-sidekiq mastodon-streaming
sudo systemctl start mastodon-web mastodon-sidekiq mastodon-streaming

# Check status
sudo systemctl status mastodon-web mastodon-sidekiq mastodon-streaming
```

### OpenRC Services (Alpine Linux)

Create `/etc/init.d/mastodon-web`:

```bash
#!/sbin/openrc-run

name="mastodon-web"
description="Mastodon web service"

user="mastodon"
group="mastodon"
directory="/home/mastodon/live"

command="/home/mastodon/.rbenv/shims/bundle"
command_args="exec puma -C config/puma.rb"
command_background="yes"

pidfile="/run/${name}.pid"
output_log="/var/log/${name}.log"
error_log="/var/log/${name}.error.log"

depend() {
    need net postgresql redis
}

start_pre() {
    export RAILS_ENV=production
    export PORT=3000
}
```

Create similar files for `mastodon-sidekiq` and `mastodon-streaming`, then:

```bash
# Make executable
sudo chmod +x /etc/init.d/mastodon-*

# Enable and start
sudo rc-update add mastodon-web
sudo rc-update add mastodon-sidekiq  
sudo rc-update add mastodon-streaming
sudo rc-service mastodon-web start
sudo rc-service mastodon-sidekiq start
sudo rc-service mastodon-streaming start
```

## Troubleshooting

### Common Issues

**Ruby/Bundle Issues:**
```bash
# Ensure proper rbenv setup
sudo -u mastodon bash << 'EOF'
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
which ruby
ruby --version
EOF

# Reinstall gems if needed
sudo -u mastodon bash << 'EOF'
cd /home/mastodon/live
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
bundle install --deployment --without development test
EOF
```

**Database Connection Issues:**
```bash
# Test PostgreSQL connection
sudo -u mastodon psql -h localhost -U mastodon mastodon_production

# Check PostgreSQL logs
sudo journalctl -u postgresql-15 -f

# Verify database configuration
sudo -u mastodon bash << 'EOF'
cd /home/mastodon/live
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
RAILS_ENV=production bundle exec rails console
# In console: ActiveRecord::Base.connection
EOF
```

**Asset Compilation Issues:**
```bash
# Clear and recompile assets
sudo -u mastodon bash << 'EOF'
cd /home/mastodon/live
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
RAILS_ENV=production bundle exec rails assets:clobber
RAILS_ENV=production bundle exec rails assets:precompile
EOF
```

**Service Startup Issues:**
```bash
# Check service logs
sudo journalctl -u mastodon-web -f
sudo journalctl -u mastodon-sidekiq -f
sudo journalctl -u mastodon-streaming -f

# Check for port conflicts
sudo netstat -tlnp | grep :3000
sudo netstat -tlnp | grep :4000

# Verify file permissions
sudo ls -la /home/mastodon/live/
sudo -u mastodon test -r /home/mastodon/live/.env.production && echo "Can read env file"
```

## Security Considerations

### Firewall Configuration

**firewalld (RHEL/CentOS/Fedora):**
```bash
# Open required ports
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --permanent --add-port=4000/tcp
sudo firewall-cmd --reload
```

**ufw (Ubuntu/Debian):**
```bash
# Configure firewall
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 3000/tcp
sudo ufw allow 4000/tcp
sudo ufw enable
```

**iptables (Generic):**
```bash
# Basic firewall rules
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 3000 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 4000 -j ACCEPT
```

### SSL/TLS Configuration with nginx

Create `/etc/nginx/sites-available/mastodon` (Debian/Ubuntu) or `/etc/nginx/conf.d/mastodon.conf` (RHEL):

```nginx
map $http_upgrade $connection_upgrade {
  default upgrade;
  ''      close;
}

upstream backend {
    server 127.0.0.1:3000 fail_timeout=0;
}

upstream streaming {
    server 127.0.0.1:4000 fail_timeout=0;
}

proxy_cache_path /var/cache/nginx/mastodon keys_zone=CACHE:10m levels=1:2 inactive=7d max_size=1g;

server {
    listen 80;
    listen [::]:80;
    server_name mastodon.example.com;
    root /home/mastodon/live/public;
    location /.well-known/acme-challenge/ { allow all; }
    location / { return 301 https://$host$request_uri; }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name mastodon.example.com;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    ssl_certificate     /etc/letsencrypt/live/mastodon.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mastodon.example.com/privkey.pem;

    keepalive_timeout    70;
    sendfile             on;
    client_max_body_size 80m;

    root /home/mastodon/live/public;

    gzip on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    add_header Strict-Transport-Security "max-age=31536000" always;

    location / {
        try_files $uri @proxy;
    }

    location ~ ^/(emoji|packs|system/accounts/avatars|system/media_attachments/files) {
        add_header Cache-Control "public, max-age=31536000, immutable";
        add_header Strict-Transport-Security "max-age=31536000" always;
        try_files $uri @proxy;
    }
    
    location /sw.js {
        add_header Cache-Control "public, max-age=604800, must-revalidate";
        add_header Strict-Transport-Security "max-age=31536000" always;
        try_files $uri @proxy;
    }

    location @proxy {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Proxy "";
        proxy_pass_header Server;

        proxy_pass http://backend;
        proxy_buffering on;
        proxy_redirect off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_cache CACHE;
        proxy_cache_valid 200 7d;
        proxy_cache_valid 410 24h;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_lock on;

        tcp_nodelay on;
    }

    location /api/v1/streaming {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Proxy "";

        proxy_pass http://streaming;
        proxy_buffering off;
        proxy_redirect off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        tcp_nodelay on;
    }

    error_page 500 501 502 503 504 /500.html;
}
```

### SSL Certificate Setup

```bash
# Install Certbot
# RHEL/CentOS/Fedora
sudo dnf install -y certbot python3-certbot-nginx

# Debian/Ubuntu  
sudo apt install -y certbot python3-certbot-nginx

# Arch Linux
sudo pacman -S certbot certbot-nginx

# Obtain certificate
sudo certbot --nginx -d mastodon.example.com

# Auto-renewal
echo "0 12 * * * /usr/bin/certbot renew --quiet" | sudo crontab -
```

### SELinux Configuration (RHEL-based systems)

```bash
# Allow nginx to connect to Mastodon services
sudo setsebool -P httpd_can_network_connect 1

# Set SELinux contexts for Mastodon files
sudo semanage fcontext -a -t httpd_exec_t "/home/mastodon/live/public(/.*)?"
sudo restorecon -R /home/mastodon/live/public/

# Allow Mastodon to bind to required ports
sudo semanage port -a -t http_port_t -p tcp 3000
sudo semanage port -a -t http_port_t -p tcp 4000
```

## Performance Tuning

### PostgreSQL Optimization

Edit `/var/lib/pgsql/15/data/postgresql.conf`:

```ini
# Memory settings (adjust based on available RAM)
shared_buffers = 256MB                 # 1/4 of RAM
effective_cache_size = 1GB            # 3/4 of RAM
work_mem = 4MB                        # RAM/max_connections/4
maintenance_work_mem = 64MB           # RAM/16

# Connection settings
max_connections = 100
superuser_reserved_connections = 3

# WAL settings for better performance
wal_buffers = 16MB
checkpoint_completion_target = 0.7
checkpoint_timeout = 5min
max_wal_size = 1GB
min_wal_size = 80MB

# Query optimization
random_page_cost = 1.1                # For SSD storage
effective_io_concurrency = 200        # For SSD storage
```

### Redis Optimization

Edit `/etc/redis/redis.conf` or `/etc/redis.conf`:

```ini
# Memory optimization
maxmemory 256mb
maxmemory-policy allkeys-lru

# Persistence (adjust based on needs)
save 900 1
save 300 10
save 60 10000

# Network optimization
tcp-keepalive 300
timeout 0
```

### Ruby/Rails Optimization

Edit `/home/mastodon/live/.env.production`:

```bash
# Database connection pooling
DB_POOL=25

# Sidekiq concurrency
SIDEKIQ_CONCURRENCY=25

# Puma workers (number of CPU cores)
WEB_CONCURRENCY=4
MAX_THREADS=5

# Streaming cluster
STREAMING_CLUSTER_NUM=4
```

### System-level Optimization

```bash
# Increase file descriptors limit
echo "mastodon soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "mastodon hard nofile 65536" | sudo tee -a /etc/security/limits.conf

# Optimize kernel parameters
sudo tee -a /etc/sysctl.conf << 'EOF'
# Network optimization
net.core.somaxconn = 1024
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_keepalive_time = 120
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3

# Memory optimization
vm.swappiness = 10
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
EOF

sudo sysctl -p
```

## Backup and Restore

### Database Backup

```bash
# Create backup script
sudo tee /usr/local/bin/mastodon-backup.sh << 'EOF'
#!/bin/bash

BACKUP_DIR="/backup/mastodon"
DATE=$(date +%Y%m%d_%H%M%S)
DB_NAME="mastodon_production"
MEDIA_DIR="/home/mastodon/live/public/system"

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Database backup
sudo -u postgres pg_dump "$DB_NAME" | gzip > "$BACKUP_DIR/db_${DATE}.sql.gz"

# Media backup (if stored locally)
if [ -d "$MEDIA_DIR" ]; then
    tar -czf "$BACKUP_DIR/media_${DATE}.tar.gz" -C "$(dirname "$MEDIA_DIR")" "$(basename "$MEDIA_DIR")"
fi

# Configuration backup
tar -czf "$BACKUP_DIR/config_${DATE}.tar.gz" /home/mastodon/live/.env.production

# Remove backups older than 30 days
find "$BACKUP_DIR" -name "*.gz" -mtime +30 -delete

echo "Backup completed: $DATE"
EOF

sudo chmod +x /usr/local/bin/mastodon-backup.sh
```

### Automated Backups

```bash
# Add to crontab
echo "0 2 * * * /usr/local/bin/mastodon-backup.sh" | sudo crontab -
```

### Restore Procedure

```bash
# Stop Mastodon services
sudo systemctl stop mastodon-web mastodon-sidekiq mastodon-streaming

# Restore database
sudo -u postgres dropdb mastodon_production
sudo -u postgres createdb mastodon_production
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE mastodon_production TO mastodon;"
gunzip -c /backup/mastodon/db_YYYYMMDD_HHMMSS.sql.gz | sudo -u postgres psql mastodon_production

# Restore media files (if needed)
sudo rm -rf /home/mastodon/live/public/system
sudo tar -xzf /backup/mastodon/media_YYYYMMDD_HHMMSS.tar.gz -C /home/mastodon/live/public/
sudo chown -R mastodon:mastodon /home/mastodon/live/public/system

# Restore configuration
sudo tar -xzf /backup/mastodon/config_YYYYMMDD_HHMMSS.tar.gz -C /

# Start services
sudo systemctl start mastodon-web mastodon-sidekiq mastodon-streaming
```

## System Requirements

### Minimum Requirements
- **CPU**: 2 cores, 2.0 GHz
- **RAM**: 4GB
- **Storage**: 20GB SSD
- **Network**: 100 Mbps
- **OS**: Linux (64-bit)

### Recommended Production Requirements
- **CPU**: 4+ cores, 2.5+ GHz
- **RAM**: 8GB+ (16GB for large instances)
- **Storage**: 100GB+ SSD with good IOPS
- **Network**: 1 Gbps with low latency
- **OS**: Recent Linux distribution with long-term support

### Scaling Considerations
- **Small Instance**: <1,000 users - 2 cores, 4GB RAM, 50GB storage
- **Medium Instance**: 1,000-10,000 users - 4 cores, 8GB RAM, 200GB storage
- **Large Instance**: 10,000+ users - 8+ cores, 16GB+ RAM, 500GB+ storage

## Support

### Official Resources
- **Official Website**: [https://joinmastodon.org](https://joinmastodon.org)
- **Documentation**: [https://docs.joinmastodon.org](https://docs.joinmastodon.org)
- **GitHub Repository**: [https://github.com/mastodon/mastodon](https://github.com/mastodon/mastodon)
- **Admin Documentation**: [https://docs.joinmastodon.org/admin/](https://docs.joinmastodon.org/admin/)

### Community Support
- **Mastodon Discord**: [https://discord.gg/mastodon](https://discord.gg/mastodon)
- **GitHub Issues**: [https://github.com/mastodon/mastodon/issues](https://github.com/mastodon/mastodon/issues)
- **Mastodon Community**: Follow @Mastodon@mastodon.social
- **Admin Forums**: Various Mastodon admin communities on the fediverse

### Commercial Support
- Available from various consulting companies
- Managed hosting providers offer Mastodon services
- Professional services for large deployments

## Contributing

### How to Contribute
1. **Report Bugs**: Submit issues to [GitHub](https://github.com/mastodon/mastodon/issues)
2. **Translate**: Help with internationalization at [Crowdin](https://crowdin.com/project/mastodon)
3. **Code Contributions**: Submit pull requests for bug fixes and features
4. **Documentation**: Improve documentation and guides
5. **Testing**: Test release candidates and provide feedback

### Development Setup
```bash
# Clone repository
git clone https://github.com/mastodon/mastodon.git
cd mastodon

# Install dependencies
bundle install
yarn install

# Setup development database
rails db:setup

# Run development server
foreman start
```

### Contribution Guidelines
- Follow Ruby and JavaScript style guides
- Include tests for new features
- Update documentation for changes
- Use conventional commit messages
- Sign commits with GPG key

## License

Mastodon is licensed under the **GNU Affero General Public License v3.0 (AGPL-3.0)**.

### Key License Points
- **Free to Use**: Can be used for any purpose
- **Source Available**: Source code must be provided to users
- **Copyleft**: Derivative works must use same license
- **Network Use**: AGPL applies to network/web services
- **No Warranty**: Software provided "as is"

### License Compliance
- Keep license notices intact
- Provide source code to users
- Document any modifications
- Use same license for derivatives
- Consider legal requirements for your jurisdiction

Full license text: [https://github.com/mastodon/mastodon/blob/main/LICENSE](https://github.com/mastodon/mastodon/blob/main/LICENSE)

## Acknowledgments

### Core Team
- **Eugen Rochko** - Creator and lead developer
- **Claire** - Core maintainer
- **Renaud Chaput** - Core maintainer
- **ThibG** - Significant contributor
- And many other contributors

### Technologies Used
- **Ruby on Rails** - Web application framework
- **PostgreSQL** - Primary database
- **Redis** - Caching and job queue
- **React.js** - Frontend user interface
- **ActivityPub** - Federation protocol
- **Sidekiq** - Background job processing

### Community
- **Translators** - Making Mastodon available in many languages
- **Instance Administrators** - Running the fediverse
- **Users** - Creating the vibrant community
- **Contributors** - Improving the software

## Version History

### Major Releases
- **v4.2** (September 2023) - Enhanced moderation tools, quote posts
- **v4.1** (February 2023) - Improved federation, performance optimizations
- **v4.0** (October 2022) - Major UI overhaul, new features
- **v3.5** (May 2022) - Content warnings improvements, admin features
- **v3.4** (October 2021) - Server rules, report improvements
- **v3.3** (February 2021) - Profile directory, admin announcements
- **v3.2** (July 2020) - Audio uploads, admin interface improvements
- **v3.1** (February 2020) - Polls, improved federation
- **v3.0** (October 2019) - Single column mode, admin dashboard

### Current Stable Version
- **v4.2.x** - Latest stable release
- Check [GitHub releases](https://github.com/mastodon/mastodon/releases) for current version

### Upgrade Path
- Always backup before upgrading
- Follow official upgrade guides
- Test in staging environment first
- Monitor for federation issues after upgrade

## Appendices

### Appendix A: Port Reference

| Port | Service | Protocol | Description |
|------|---------|----------|-------------|
| 80 | HTTP | TCP | Web traffic (redirects to HTTPS) |
| 443 | HTTPS | TCP | Secure web traffic |
| 3000 | Mastodon Web | TCP | Internal web service |
| 4000 | Mastodon Streaming | TCP | Real-time updates |
| 5432 | PostgreSQL | TCP | Database (internal) |
| 6379 | Redis | TCP | Cache and queues (internal) |
| 25/587 | SMTP | TCP | Email delivery (outbound) |

### Appendix B: File Locations

| Path | Description |
|------|-------------|
| `/home/mastodon/live/` | Mastodon application directory |
| `/home/mastodon/live/.env.production` | Main configuration file |
| `/home/mastodon/live/public/system/` | Media files storage |
| `/etc/systemd/system/mastodon-*.service` | Service definitions |
| `/etc/nginx/sites-available/mastodon` | nginx configuration |
| `/var/lib/postgresql/*/data/` | PostgreSQL data directory |
| `/etc/redis/redis.conf` | Redis configuration |
| `/var/log/nginx/` | nginx logs |

### Appendix C: Common Commands

```bash
# Service management
sudo systemctl status mastodon-web mastodon-sidekiq mastodon-streaming
sudo systemctl restart mastodon-web mastodon-sidekiq mastodon-streaming

# Mastodon maintenance
sudo -u mastodon bash -c 'cd /home/mastodon/live && RAILS_ENV=production bundle exec rails mastodon:maintenance'

# Database management
sudo -u postgres psql mastodon_production

# Log monitoring
sudo journalctl -u mastodon-web -f
sudo tail -f /var/log/nginx/access.log
```

### Appendix D: Environment Variables Reference

| Variable | Description | Required |
|----------|-------------|----------|
| `LOCAL_DOMAIN` | Your Mastodon domain | Yes |
| `SECRET_KEY_BASE` | Rails secret key | Yes |
| `OTP_SECRET` | Two-factor authentication secret | Yes |
| `VAPID_PRIVATE_KEY` | Push notification private key | Yes |
| `VAPID_PUBLIC_KEY` | Push notification public key | Yes |
| `DB_HOST` | Database hostname | Yes |
| `DB_USER` | Database username | Yes |
| `DB_PASS` | Database password | Yes |
| `DB_NAME` | Database name | Yes |
| `REDIS_HOST` | Redis hostname | Yes |
| `SMTP_SERVER` | SMTP server hostname | Recommended |
| `SMTP_FROM_ADDRESS` | From email address | Recommended |
| `S3_ENABLED` | Enable S3 storage | No |
| `CDN_HOST` | CDN hostname for assets | No |

---

**Note:** This guide is part of the [HowToMgr](https://howtomgr.github.io) collection. Always refer to official Mastodon documentation for the most up-to-date information.