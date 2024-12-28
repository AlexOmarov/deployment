1. Создать admin sudo пользователя
   sudo useradd -m -s /bin/bash admin
   sudo passwd admin
   sudo usermod -aG sudo admin
2. Обновить и поставить vpn либы
   sudo apt install --reinstall software-properties-common
   sudo add-apt-repository universe
   sudo apt-get update && sudo apt-get upgrade
   sudo apt-get install strongswan-starter libcharon-extra-plugins libcharon-extauth-plugins libstrongswan-extra-plugins libtss2-tcti-tabrmd0
