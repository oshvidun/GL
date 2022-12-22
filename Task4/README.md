В рамках данной задачи будет развертывание Wordress, с использованием Terraform в Google Cloud.


![GC](https://user-images.githubusercontent.com/85619391/209127572-6b34c0fc-38ac-4273-8a9b-ef7088987276.png)

Для того чтобы развернуть проект нам потребуется:
установить google cloud CLI, создать проект в котром будут развораичаться Wordpress, включить все необходимые googleapis, создать сервис аккаунт иключ для подключения.

Мы создадим три файла
1. my.json - необходим для авторизации сервис аккаунта
2. main.tf - инструкции и настройки нашего проекта
3. init.sh - скрипт который устанавливает веб сервер и загружает пакет WordPress после запуска ОС

main.tf
Настройка провайдера:

'''provider "google" {
# connect the file that contains the data for authorization
  credentials = file("my.json") 
#
  project = "gl-task4-wordpress-372318"
  region = "europe-west1"
} '''

Создаем сеть и подсеть в которую в дальнейшем будут добавлены наши ресурсы
''' #Create network 
resource "google_compute_network" "this" {
  auto_create_subnetworks = false
  name                    = "network-wordpress"
  routing_mode            = "REGIONAL"
}
#Create subnet
resource "google_compute_subnetwork" "this" {
  name          = "network-wordpress"
  ip_cidr_range = "10.20.0.0/28"
  region        = "europe-west1"
  network       = google_compute_network.this.id
} '''

Создадим правило для базы данных

''' #Create databases private IP address
resource "google_compute_global_address" "this" {

  name          = "private-ip-db-address"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.this.id
}
resource "google_service_networking_connection" "this" {

  network                 = google_compute_network.this.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.this.name]
} '''

Настроим Firewall
''' #Fierwall rule
resource "google_compute_firewall" "wordpress_ingress" {
  name    = "allow-http"
  network = google_compute_network.this.id

  allow {
    protocol = "icmp"
  }

  allow {
    protocol = "tcp"
    ports    = ["80"]
  }

  source_ranges = ["0.0.0.0/0"]
}

resource "google_compute_firewall" "wordpress_ingress_ssh" {
  name    = "allow-ssh"
  network = google_compute_network.this.id

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["Enter_your IP/32"]
} '''

В качестве базы данных мы не будем создавать отдельный инстанс, а восольземся mySQL PaaS
''' #Create DB
resource "google_sql_database_instance" "this" {
  database_version = "MYSQL_5_7"
  name             = "mydb-wordpress"
  region           = "europe-west1"

  depends_on = [
  google_service_networking_connection.this]

  settings {
    availability_type = "REGIONAL"
    disk_autoresize   = false
    disk_size         = 50
    disk_type         = "PD_HDD"
    tier              = "db-g1-small"
    backup_configuration {
      enabled            = true
      start_time         = "04:00"
      binary_log_enabled = true
    }

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.this.id
    }
    database_flags {
      name  = "max_connections"
      value = 500
    }
  }
}
resource "google_sql_database" "this" {
  name      = "wordpress"
  instance  = google_sql_database_instance.this.name
  charset   = "utf8"
  collation = "utf8_general_ci"
}
resource "random_string" "this" {
 length    = 24
 special   = false
 min_upper = 5
 min_lower = 5
}

resource "random_password" "this" {
 length    = 24
 special   = false
 min_upper = 5
 min_lower = 5
}
resource "google_sql_user" "this" {
 name     = random_string.this.result
 password = random_password.this.result
 instance = google_sql_database_instance.this.name
} '''
Теперь создадим инстанс на котором будет развлернут наш WordPress
''' #Create instance with wordpress
resource "google_compute_instance" "this" {
 name                    = "inst-wordpress"
 machine_type            = "e2-standard-2"
 zone                    = "europe-west1-b"
  metadata_startup_script = templatefile("${path.module}/init.sh", {
    DB_USERNAME = random_string.this.result
    DB_PASSWORD = random_password.this.result
    DB_HOST     = google_sql_database_instance.this.private_ip_address
  })

 boot_disk {
   initialize_params {
     image = "debian-cloud/debian-10"
     size  = 50
   }
 }

 network_interface {
   subnetwork = google_compute_subnetwork.this.id

   access_config {
     network_tier = "STANDARD"
   }
 }

 service_account {
   scopes = ["userinfo-email", "compute-ro", "storage-ro"]
 }
} '''

init.sh

''' #!/usr/bin/env bash

apt update
apt install -y apache2 \
            ghostscript \
            libapache2-mod-php \
            php \
            php-bcmath \
            php-curl \
            php-imagick \
            php-intl \
            php-json \
            php-mbstring \
            php-mysql \
            php-xml \
            php-zip

mkdir -p /srv/www
sudo chown www-data: /srv/www
curl --silent https://wordpress.org/latest.tar.gz | sudo -u www-data tar zx -C /srv/www
find /srv/www/ -type d -exec chmod 755 {} \;
find /srv/www/ -type f -exec chmod 644 {} \;
cat << EOF > /etc/apache2/sites-available/000-default.conf
<VirtualHost *:80>
  DocumentRoot /srv/www/wordpress
  <Directory /srv/www/wordpress>
      Options FollowSymLinks
      Require all granted
      DirectoryIndex index.php
      Order allow,deny
      Allow from all
  </Directory>
  <Directory /srv/www/wordpress/wp-content>
      Options FollowSymLinks
      Order allow,deny
      Allow from all
  </Directory>
</VirtualHost>
EOF
a2enmod rewrite
systemctl reload apache2
sudo -u www-data cp /srv/www/wordpress/wp-config-sample.php /srv/www/wordpress/wp-config.php

sudo -u www-data sed -i 's/database_name_here/wordpress/' /srv/www/wordpress/wp-config.php
sudo -u www-data sed -i 's/username_here/${DB_USERNAME}/' /srv/www/wordpress/wp-config.php
sudo -u www-data sed -i 's/password_here/${DB_PASSWORD}/' /srv/www/wordpress/wp-config.php
sudo -u www-data sed -i 's/localhost/${DB_HOST}/' /srv/www/wordpress/wp-config.php

systemctl restart apache2 '''


