
# Get initial certificates (inside certbot container)
certbot certonly --webroot --webroot-path=/var/www/certbot --email omarov.dev@yandex.ru --agree-tos --no-eff-email -d berte-edu.ru -d gitlab.berte-edu.ru -d sonar.berte-edu.ru -d rancher.berte-edu.ru

# Change rights for files to admin user
sudo chown -R admin:admin /home/admin
chmod -R 777 /home/admin/dev_server

ранчер - в config/certbot/conf/live/berte-edu.ru/ дублировал с именем key