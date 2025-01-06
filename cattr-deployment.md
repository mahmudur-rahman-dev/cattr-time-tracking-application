# Cattr Deployment Sequence

## 1. Setup EC2
```bash
# Launch t2.small with Amazon Linux 2023
# Security Group: 22, 80, 443
ssh -i key.pem ec2-user@your-ip
```

## 2. Install Dependencies
```bash
sudo yum update -y
sudo yum install -y docker git
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
sudo curl -L "https://github.com/docker/compose/releases/download/v2.21.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
exit
```

## 3. Clone and Configure
```bash
# SSH back in
ssh -i key.pem ec2-user@your-ip

# Clone repository
git clone https://github.com/cattr-app/server-application.git cattr
cd cattr

# Create docker-compose.override.yml
cat > docker-compose.override.yml << 'EOL'
version: '3.9'

services:
  app:
    restart: unless-stopped
    ports:
      - "80:80"

  db:
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=password
      - MYSQL_DATABASE=cattr
    volumes:
      - type: volume
        source: mysql-data
        target: /var/lib/mysql

volumes:
  mysql-data:
EOL

# Create .env file
cat > .env << 'EOL'
APP_DEBUG=false
APP_URL=http://YOUR_IP_HERE
FRONTEND_APP_URL=http://YOUR_IP_HERE

DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=cattr
DB_USERNAME=root
DB_PASSWORD=password

REQUEST_SIGNATURE=demo2024secure

SCREENSHOTS_STATE=optional
RECAPTCHA_ENABLED=false
RATE_LIMITER_ENABLED=false

REVERB_APP_KEY=cattr
REVERB_APP_SECRET=secret

VUE_APP_STORAGE_SPACE_MAX_USED=75
EOL

# Replace IP in .env
sed -i "s/YOUR_IP_HERE/$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)/g" .env
```

## 4. Deploy
```bash
# Start containers
docker-compose up -d

# Wait for DB
sleep 30

# Generate key
docker-compose exec app sh -c "cd /app && echo \"APP_KEY=base64:$(openssl rand -base64 32)\" >> .env"

# Setup database
docker-compose exec db mysql -uroot -ppassword -e "DROP DATABASE IF EXISTS cattr; CREATE DATABASE cattr;"
docker-compose exec app sh -c "cd /app && php82 artisan migrate --force"

# Create admin
docker-compose exec app sh -c "cd /app && php82 artisan cattr:make:admin"
```

## 5. Access
- Open http://YOUR_IP in browser
- Login with admin credentials created

## Maintenance Commands
```bash
# View logs
docker-compose logs

# Restart services
docker-compose restart

# Update
docker-compose pull
docker-compose down
docker-compose up -d

# Backup database
docker-compose exec db mysqldump -u root -ppassword cattr > backup.sql
```