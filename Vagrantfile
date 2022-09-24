# -*- mode: ruby -*-
# vi: set ft=ruby :

# переменные
LOCAL_HOST_PORT = "45678"
                  # создаем переменную для проброса порта 
VM_NAME = "web - Wordpress+Drupal"
                  # создаем переменную для имени машины
DB_WORDPRESS="WORDPRESS"
DB_DRUPAL="DRUPAL"
DB_WPUSER="wpuser1"
PASS_WPUSER="samplepassword"
DB_DPUSER="wpuser2"
PASS_DPUSER="samplepassword"

# stage 0
# настраиваем виртуальную машину

Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/focal64"
  config.vm.hostname = "web"
  config.vm.network "forwarded_port", guest: 80, host: LOCAL_HOST_PORT
  config.vm.network "public_network"
                  # добавляем сетевой интерфейс, который смотрит в локальную сеть
  config.vm.synced_folder "./sync-config-folder", "/sync-config-folder", id: "sync-config-folder", automount: true
  config.vm.provision "shell", inline: "usermod -a -G vboxsf vagrant"
                  # добавляем пользователя vagrant в группу vboxsf для доступа к папке
  config.vm.boot_timeout = 1800
                  # время в секундах, в течение которого Vagrant будет ждать, пока машина загрузится и станет доступной. По умолчанию это 300 секунд

  config.vm.provider "virtualbox" do |vb|
   vb.name = VM_NAME
                  # понятное имя ВМ в virtualbox
   vb.memory = 4096
   vb.cpus = 2   
                  # сначала надо создать пустой dvd'rom и только потом в него можно смонитовать образ
   vb.customize ["storageattach", :id, "--storagectl", "IDE", "--port", "1", "--device", "1",
                "--type", "dvddrive", "--mtype", "readonly", "--medium", "emptydrive"]

                  # смонтируем диск VBoxLinuxAdditions.iso для последующей его установки
   vb.customize ["storageattach", :id, "--storagectl", "IDE", "--port", "1", "--device", "1",
                "--type", "dvddrive", "--mtype", "readonly", "--medium", "additions", "--forceunmount"]
  end


# stage 1
# устанавливаем обновления и чистим систему
  
  config.vm.provision :shell, inline: <<-SHELL
      echo "Stage 1"
      sudo apt-get update && apt-get upgrade -y
      sudo timedatectl set-timezone Europe/Moscow

      # ставим VBoxLinuxAdditions
      sudo mkdir -p /mnt/cdrom
      sudo mount /dev/cdrom /mnt/cdrom
      cd /mnt/cdrom
      echo y | sudo sh ./VBoxLinuxAdditions.run

      sudo apt autoclean
                     # удалить неиспользуемые пакеты из кэша
      sudo apt clean
                     # очитска кэша
      sudo apt autoremove
                     # удаление ненужных зависимостей
      sudo apt autoremove --purge

    SHELL

# перезагружаем виртуальную машину
  config.vm.provision :shell do |shell|

    shell.privileged = true
    shell.reboot = true
  end

# stage 2
# линкуем шарную папку, даже после перезагрузки vm
  config.vm.provision :shell, inline: <<-SHELL
      echo "Stage 2"
      sudo rm -rf /sync-config-folder
                        # линкуем папку для доступа к ней после перезагрузки
      sudo ln -sfT /media/sf_sync-config-folder /sync-config-folder

    SHELL

# stage 3
# устанавливаем необходимые пакеты и производим их настройку
# передаем переменные "LOCAL_HOST_PORT", "DB_WORDPRESS", "DB_DRUPAL", "DB_USER", "PASS_USER", "PASS_ROOT" в shell

  config.vm.provision :shell, env: {"LOCAL_HOST_PORT" => LOCAL_HOST_PORT, "DB_WORDPRESS" => DB_WORDPRESS, "DB_DRUPAL" => DB_DRUPAL, "DB_WPUSER" => DB_WPUSER, "PASS_WPUSER" => PASS_WPUSER, "DB_DPUSER" => DB_DPUSER, "PASS_DPUSER" => PASS_DPUSER}, inline: <<-SHELL
      echo "Stage 3"
      sudo apt install -y apache2
                     # установим apache2
      sudo apt install -y php libapache2-mod-php php-mysql
                     # установим php. В дополнение к php пакету также потребуется libapache2-mod-php и php-mysql
      sudo apt install -y php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip php-sqlite3 php-cli
                     # установим некоторые из самых популярных расширений PHP
      sudo apt install -y mysql-server
                     # установим СУБД mysql
      sudo a2enmod rewrite
                     # включим модуль rewrite для apache

  # настройка wordpress 
      wget -O /var/www/wordpress-latest.tar.gz https://wordpress.org/latest.tar.gz
                     # скачиваем архив wordpress с именем wordpress-latest.tar.gz в директорию /var/www/
      tar -xzf /var/www/wordpress-latest.tar.gz -C /var/www/
                     # распаковываем архив wordpress в директорию /var/www/
      sudo chown -R www-data:www-data /var/www/wordpress/
                     # дадим права пользователю www-data рекурсивно на папку wordpress
      sudo chmod -R 755 /var/www/wordpress/
      sudo rm -rf /var/www/wordpress-latest.tar.gz
                     # удаляем скаченный архив
      sudo cp /sync-config-folder/apache/wordpress.conf /etc/apache2/sites-available/wordpress.conf
                     # скопируем подготовленный конфиг для wordpress из общей папки
      sudo chmod -X /etc/apache2/sites-available/wordpress.conf
                     # убираем атрибут исполняемого файла т.к. все файлы в Windows исполняемые
      sudo ln -s /etc/apache2/sites-available/wordpress.conf /etc/apache2/sites-enabled/wordpress.conf
                     # включаем конфиг wordpress в apache путем создания симлинка
      sudo cp -p /var/www/wordpress/wp-config-sample.php  /var/www/wordpress/wp-config.php
                     # копируем wordpress config
      sudo chmod a=rw /var/www/wordpress/wp-config.php
      
      # заменим переменные подключения к базе данных на наши значения
      sed -i 's/database_name_here/'$DB_WORDPRESS'/g' /var/www/wordpress/wp-config.php
      sed -i 's/username_here/'$DB_WPUSER'/g' /var/www/wordpress/wp-config.php
      sed -i 's/password_here/'$PASS_WPUSER'/g' /var/www/wordpress/wp-config.php

      # Генерируем соли и записываем их в файл
      SALT=$(curl -L https://api.wordpress.org/secret-key/1.1/salt/)
      STRING='put your unique phrase here'
      printf '%s\n' "g/$STRING/d" a "$SALT" . w | ed -s /var/www/wordpress/wp-config.php
      
      
    # создаем базу данных mysql
      mysql -u root -e "CREATE DATABASE $DB_WORDPRESS DEFAULT CHARACTER SET utf8;"
      mysql -u root -e "CREATE USER '$DB_WPUSER'@'localhost' IDENTIFIED BY '$PASS_WPUSER';"
      mysql -u root -e "GRANT ALL PRIVILEGES ON $DB_WORDPRESS. * TO '$DB_WPUSER'@'localhost';"
      mysql -u root -e "FLUSH PRIVILEGES;"

      sudo systemctl restart apache2
      sudo systemctl restart mysql

 # настройка drupal
      wget -O /var/www/drupal-latest.tar.gz https://www.drupal.org/download-latest/tar.gz
                     # скачиваем архив drupal с именем drupal-latest.tar.gz в директорию /var/www/
      tar -xzf /var/www/drupal-latest.tar.gz -C /var/www/
                     # распаковываем архив drupal в директорию /var/www/
      sudo rm -rf /var/www/drupal-latest.tar.gz
                     # удаляем скаченный архив
      sudo mv /var/www/drupal-*/ /var/www/drupal
                     # переименовываем распакованный архив
      
      sudo chown -R root:www-data /var/www/drupal/
                     # дадим права пользователю www-data рекурсивно на папку drupal
      sudo chmod -R 755 /var/www/drupal/
                     # выставим права на папку drupal
      sudo cp /var/www/drupal/sites/default/default.settings.php /var/www/drupal/sites/default/settings.php
                     # скопируем дефолтный конфиг
      sudo mkdir /var/www/drupal/sites/default/files
                     # создадим необходимые файлы, папки и дадим им права
      sudo mkdir /var/www/drupal/sites/default/files/translations
      sudo chmod a+w /var/www/drupal/sites/default
      sudo chmod a+w /var/www/drupal/sites/default/settings.php
      sudo chmod a+w /var/www/drupal/sites/default/files
      sudo chmod a+w /var/www/drupal/sites/default/files/translations

      sudo cp /sync-config-folder/apache/drupal.conf /etc/apache2/sites-available/drupal.conf
                     # скопируем подготовленный конфиг для drupal из общей папки
      sudo chmod -X /etc/apache2/sites-available/drupal.conf
                     # убираем атрибут исполняемого файла т.к. все файлы в Windows исполняемые
      sudo ln -s /etc/apache2/sites-available/drupal.conf /etc/apache2/sites-enabled/drupal.conf
                     # включаем конфиг drupal в apache путем создания симлинка

    # создаем базу данных mysql
      mysql -u root -e "CREATE DATABASE $DB_DRUPAL DEFAULT CHARACTER SET utf8;"
      mysql -u root -e "CREATE USER '$DB_DPUSER'@'localhost' IDENTIFIED BY '$PASS_DPUSER';"
      mysql -u root -e "GRANT ALL PRIVILEGES ON  $DB_DRUPAL. * TO '$DB_DPUSER'@'localhost';"
      mysql -u root -e "FLUSH PRIVILEGES;"
      
 # рестартим apache и mysql
      sudo systemctl restart apache2
      sudo systemctl restart mysql

      SHELL

      # перезагружаем виртуальную машину
  config.vm.provision :shell do |shell|

    shell.privileged = true
    shell.reboot = true
  end
end



