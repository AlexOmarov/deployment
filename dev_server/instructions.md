
# Get initial certificates (inside certbot container)
certbot certonly --webroot --webroot-path=/var/www/certbot --email omarov.dev@yandex.ru --agree-tos --no-eff-email -d berte-edu.ru -d gitlab.berte-edu.ru -d sonar.berte-edu.ru -d rancher.berte-edu.ru

# Change rights for files to admin user
sudo chown -R admin:admin /home/admin
chmod -R 777 /home/admin/dev_server

ранчер - в config/certbot/conf/live/berte-edu.ru/ дублировал с именем key

0. обновить убунту, поставить файервол - инкаминг только из впн, аут - отовсюду, установить докер
   добавить юзера добавить его в судо и докер группу
1. перенести файлы в хом админа, запустить нгинкс с акме, запустить сертбот и выписать сертификат, далее потушить контейнеры и запустить полный композ

sudo apt-get install libreswan xl2tpd net-tools
sudo nano /etc/ipsec.conf



ПРАВА НА ФАЙЛЫ:

docker compose up --user 1000:1000



docker stop dev_server-gitlab-1







Гитлаб бэкап
docker exec dev_server-gitlab-1 gitlab-backup create
TODO: Взять получившийся tar, сохранить вместе с конфигами и перекинуть в облако



Гитлаб рестор

удалить все кроме конфигов и бэкапа

взвести заново контейнер

sudo gitlab-ctl stop puma
sudo gitlab-ctl stop sidekiq
sudo gitlab-ctl status
sudo gitlab-backup restore BACKUP=<TIMESTAMP>

gitlab-ctl reconfigure
gitlab-ctl start
gitlab-ctl status