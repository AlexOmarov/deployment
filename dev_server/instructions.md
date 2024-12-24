1. Создать admin sudo пользователя
   sudo useradd -m -s /bin/bash admin
   sudo passwd admin
   sudo usermod -aG sudo admin
2. Обновить и поставить vpn либы
   sudo add-apt-repository universe
   sudo apt-get update && sudo apt-get upgrade
   sudo apt-get install strongswan-starter libcharon-extra-plugins libcharon-extauth-plugins libstrongswan-extra-plugins libtss2-tcti-tabrmd0
3. Настроить vpn client connect
   sudo chmod -R 777 /etc/ipsec.d
   Положить сертфикат vpn-сервера (назвать vpn.crt) в /etc/ipsec.d/certs и /etc/ipsec.d/cacerts
   Положить kaz.conf и kaz.secrets в /etc/ipsec.d
   sudo nano /etc/ipsec.conf (добавить include /etc/ipsec.d/*.conf в конец файла без отступов)
   sudo nano /etc/ipsec.secrets (добавить include /etc/ipsec.d/*.secrets в конец файла без отступов)
   sudo systemctl restart strongswan-starter
   sudo ip addr show должен в eth0 быть доп inet c виртуальным ip
   sudo ipsec statusall - должен быть 1 секурный коннект
4. Настроить файервол
   Переподключиться к впн-ке и зайти в терминал заново
   Сделать файервол deny in out
   Сделать файервол allow in out для ip vpn server, ip текущего сервера, на всякий случай виртуального шлюза
   На этом моменте сервер должен быть доступен только через впн и ходить только через впн сам
   sudo ip route show table 220 - один дефолт default via 193.124.112.1 dev eth0 proto static src 10.123.0.1
   route -n - всего два роута:
   0.0.0.0         193.124.112.1   0.0.0.0         UG    0      0        0 eth0
   193.124.112.0   0.0.0.0         255.255.252.0   U     0      0        0 eth0
5. Установка docker
   sudo apt-get install ca-certificates curl
   sudo install -m 0755 -d /etc/apt/keyrings
   sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
   sudo chmod a+r /etc/apt/keyrings/docker.asc

   echo \
   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
   $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   
   sudo apt-get update
   sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   sudo usermod -aG docker admin
   Релогин
   На этом этапе через впн установлен докер, docker отдает пустой список
   ip addr show - добавился docker0
   route -n - добавился 172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
6. Подготовка к установке дев-сервера
   Делаем в хоме админа папку dev_server, в нее кладем docker-compose
   docker network create -d bridge -o 'com.docker.network.bridge.name'='vpn_docker' --subnet=172.21.0.0/16 vpn_docker
   Переносим конфиг в папку config (структура как в compose в volume)
   Переносим данные в папку data (структура как в compose в volume)
   sudo ip route add 172.21.0.0/16 dev vpn_docker scope link src 193.124.113.173 table 220
   sudo ip route add 172.17.0.0/16 dev docker0 scope link src 193.124.113.173 table 220 (для тестконтейнеров, ryuk и остальные запускаются на дефолтном бридже 172.17.0.0/16 при socket mounting)
   если с тестконтейнерами не получается - возможно нужные forward/input правила на accept траффика из 172.17 и 172.21, но глубоко не копал
   ЕСЛИ ЧТО НЕ ТАК - sudo ip route del 172.17.0.0/16 dev docker0 scope link src 193.124.113.173 table 220



   iptables -t nat -I POSTROUTING 1 -s 172.17.0.0/16 -d 172.21.0.0/16 -j RETURN
   iptables -t nat -I POSTROUTING 1 -s 172.21.0.0/16 -d 172.17.0.0/16 -j RETURN
   iptables -A FORWARD -s 172.17.0.0/16 -d 172.21.0.0/16 -j ACCEPT
   iptables -A FORWARD -s 172.21.0.0/16 -d 172.17.0.0/16 -j ACCEPT
   iptables -A FORWARD -s 172.17.0.0/16 -d 172.21.0.0/16 -j ACCEPT
   iptables -A FORWARD -s 172.21.0.0/16 -d 172.17.0.0/16 -j ACCEPT
   Chain DOCKER-USER (0 references)
   target     prot opt source               destination
   ACCEPT     all  --  0.0.0.0/16           0.0.0.0/16
   ACCEPT     all  --  172.21.0.0/16        172.17.0.0/16
   ACCEPT     all  --  172.17.0.0/16        172.21.0.0/16

   sudo iptables -j SNAT -t nat -I POSTROUTING 1 -d 0.0.0.0/0 -s 172.21.0.0/16 --to-source 10.123.0.1 (если что - sudo iptables -t nat -D POSTROUTING 1)
   если что - рестарт докера
7. Установка dev_server
   cd dev_server
   убрать из конфига нгинкса все редиректы кроме ackme
   Запускаем sudo DOCKER_SOCKET=/var/run/docker.sock docker compose up -d
   Проверяем что есть доступ в инет docker exec -it dev_server-certbot-1 ping 8.8.8.8 (если что - docker stop dev_server-nginx-1)
   docker exec -it dev_server-certbot-1 curl -v https://google.com 
   docker exec -it dev_server-certbot-1 curl -v https://gitlab.berte-edu.ru
   убираем файервол временно
   (Не нужно - делаем общий серт) Запускаем docker exec --user root -it dev_server-certbot-1 certbot certonly --webroot --webroot-path=/var/www/certbot --email omarov.dev@yandex.ru --agree-tos --no-eff-email -d berte-edu.ru -d gitlab.berte-edu.ru -d sonar.berte-edu.ru -d rancher.berte-edu.ru
   Запускаем docker exec --user root -it dev_server-certbot-1 certbot certonly --manual --preferred-challenges=dns --webroot-path=/var/www/certbot --email omarov.dev@yandex.ru --agree-tos --no-eff-email -d "*.berte-edu.ru" -d "berte-edu.ru"
   в хостинге где привязывали dns имя (не где покупали, а там где серверы) добавляем txt _acme-challenge.berte-edu.ru. со значением которое выдаст команда и ждем 3-5 минут
   продолжаем выполнение команды
   восстанавливаем конфиг нгинкса все редиректы
8. Настройка гитлаба
   docker exec -it dev_server-gitlab-1 bash
   gitlab-rails console -e production (wait for 15 min)
   user = User.where(id: 1).first
   user.password = 'secret_pass'
   user.password_confirmation = 'secret_pass'
   user.save!
   Зайти в интерфейс добавить новых пользователей и ключи
   Добавить группу и пустой проект
   Добавить новый origin в свой локальный проект по http
   Запушить
   Выписать токен в UI для раннера
   записать его в конфиг раннера
   Перезапустить докер контейнер с раннером
   включить в общих настройках проекта package registry   
   выписать себе персональный токен и положить в gradle.properties

# Change rights for files to admin user
sudo chown -R admin:admin /home/admin
chmod -R 644 (или 777) /home/admin/dev_server

ранчер - в config/certbot/conf/live/berte-edu.ru/ дублировал с именем key


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
