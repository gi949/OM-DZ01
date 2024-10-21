# OM-DZ01

На виртуальной машине установлена CMS wordpress, которая включает в себя следующие компоненты: 

nginx, php-fpm, database (MySQL)

На CMS развернут тестовый домен http://www.example.com/



Установка и настройка node_exporter

Скачиваем и распаковываем Node Exporter

wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz

tar xzvf node_exporter-1.8.2.linux-amd64.tar.gz


Создаем пользователя, перемещаем бинарник в /usr/local/bin

useradd -rs /bin/false nodeusr

mv node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/


Создаем файл для автозапуска экспортера:

vim /etc/systemd/system/node_exporter.service

[Unit]

Description=Node Exporter

After=network.target

[Service]

User=nodeusr

Group=nodeusr

Type=simple

ExecStart=/usr/local/bin/node_exporter

[Install]

WantedBy=multi-user.target


Запускаем сервис

systemctl daemon-reload

systemctl start node_exporter

systemctl enable node_exporter


Также можно получить набор метрик:

curl http://localhost:9100/metrics





Установка nginx_exporter

Подключаем репозиторий

yum install epel-release

Snap можно установить следующим образом:

yum install snapd

После установки необходимо включить модуль systemd , который управляет основным сокетом связи Snap:

systemctl enable --now snapd.socket

ln -s /var/lib/snapd/snap /snap

Установить экспортер NGINX Prometheus

snap install nginx-prometheus-exporter

cp /var/lib/snapd/snap/nginx-prometheus-exporter/120/nginx-prometheus-exporter /usr/local/bin/


Создаем файл для автозапуска экспортера:

vim /etc/systemd/system/nginx_exporter.service

[Unit]

Description=Node Exporter Service

After=network.target

[Service]

User=nodeusr

Group=nodeusr

Type=simple

ExecStart=/usr/local/bin/nginx-prometheus-exporter -nginx.scrape-uri=http://127.0.0.1:8080/server-status

ExecReload=/bin/kill -HUP $MAINPID

Restart=on-failure

[Install]

WantedBy=multi-user.target


Разрешаем и стартуем сервис nginx_exporter:

systemctl enable nginx_exporter

systemctl start nginx_exporter


Также можно получить набор метрик:

curl 127.0.0.1:9113/metrics



Установка mysql_exporter

Создаем пользователя:

groupadd --system prometheus

useradd -s /sbin/nologin --system -g prometheus prometheus


Скачиваем и распаковываем mysqld_exporter:

wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz

tar xvf mysqld_exporter-0.15.1.linux-amd64.tar.gz

mv  mysqld_exporter-*.linux-amd64/mysqld_exporter /usr/local/bin/

chmod +x /usr/local/bin/mysqld_exporter

mysqld_exporter  --version


Создаем пользователя в БД mysql:

CREATE USER 'mysqld_exporter'@'localhost' IDENTIFIED BY 'StrongPass1';

GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysqld_exporter'@'localhost';

FLUSH PRIVILEGES;


Настраиваем доступ к БД для mysqld_exporter: 

vim /etc/.mysqld_exporter.cnf

[client]

user=mysqld_exporter

password=StrongPass1


Задаем права:

chown root:prometheus /etc/.mysqld_exporter.cnf


Создаем файл для автозапуска экспортера:

vi /etc/systemd/system/mysql_exporter.service

[Unit]

Description=Prometheus MySQL Exporter

After=network.target

User=prometheus

Group=prometheus


[Service]

Type=simple

Restart=always

ExecStart=/usr/local/bin/mysqld_exporter \
--config.my-cnf /etc/.mysqld_exporter.cnf \
--collect.global_status \
--collect.info_schema.innodb_metrics \
--collect.auto_increment.columns \
--collect.info_schema.processlist \
--collect.binlog_size \
--collect.info_schema.tablestats \
--collect.global_variables \
--collect.info_schema.query_response_time \
--collect.info_schema.userstats \
--collect.info_schema.tables \
--collect.perf_schema.tablelocks \
--collect.perf_schema.file_events \
--collect.perf_schema.eventswaits \
--collect.perf_schema.indexiowaits \
--collect.perf_schema.tableiowaits \
--collect.slave_status \
--web.listen-address=0.0.0.0:9104


[Install]

WantedBy=multi-user.target


Разрешаем и стартуем сервис 

systemctl daemon-reload

systemctl enable mysql_exporter

systemctl start mysql_exporter


Также можно получить набор метрик: 

curl http://localhost:9104/metrics



На второй ВМ устанавливаем и настраиваем Blackbox Exporter

Добавляем пользователя и создаем каталог для файла конфигурации Blackbox Exporter:

useradd -M -s /bin/false blackbox

mkdir /opt/blackbox

Скачиваем финальную версию, распаковываем её и копируем в каталог:

wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.23.0/blackbox_exporter-0.23.0.linux-amd64.tar.gz

tar xzf blackbox_exporter-0.23.0.linux-amd64.tar.gz

cd blackbox_exporter-0.23.0.linux-amd64

cp blackbox_exporter /usr/local/bin/



Настраиваем конфигурацию Blackbox Exporter:

nano /opt/blackbox/blackbox.yml


modules:

  http_2xx:
  
    prober: http
    
    timeout: 5s
    
    http:
    
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200]
      
      no_follow_redirects: false
      
      preferred_ip_protocol: "ip4"


Создаем системный юнит

sudo nano /etc/systemd/system/blackbox_exporter.service

[Unit]

Description=Blackbox Exporter Service

Wants=network-online.target

After=network-online.target


[Service]

User=blackbox

Group=blackbox

ExecStart=/usr/local/bin/blackbox_exporter --config.file=/opt/blackbox/blackbox.yml

Restart=always


[Install]

WantedBy=multi-user.target


Разрешаем и стартуем сервис:

systemctl daemon-reload

systemctl enable --now blackbox_exporter

systemctl status blackbox_exporter






На этой же ВМ запускаем prometheus server с помощью docker-compose

Настраиваем конфигурацию docker-compose:

vi docker-compose.yml

version: '3'


services:

  prometheus:

    image: prom/prometheus:latest
    volumes:
    
    - ./prometheus.yml/:/etc/prometheus/prometheus.yml
    
    command:
    
    - '--config.file=/etc/prometheus/prometheus.yml'
    
    ports:
    
    - 9090:9090
	
	
Настраиваем конфигурацию сервера prometheus
prometheus.yml

global:

  scrape_interval: 5s
  
  scrape_timeout: 3s
  
  evaluation_interval: 5s
  

scrape_configs:

  - job_name: node

    static_configs:

      - targets:

	- 192.168.56.103:9100

  - job_name: nginx

    static_configs:

      - targets:

	- 192.168.56.103:9113


  - job_name: mysql

    static_configs:

      - targets:

	- 192.168.56.103:9104
		
  - job_name: blackbox

    metrics_path: /probe

    params:

      module: [http_2xx]

    static_configs:

      - targets:

	- http://www.example.com

	- http://www.example.com/?page_id=2

	- www.example.com/?page_id=7

    relabel_configs:

      - source_labels: [__address__]

	target_label: __param_target

      - source_labels: [__param_target]

	target_label: instance

      - target_label: __address__

	replacement: 192.168.56.101:9115



В разделе blackbox в пункте targets указываем URL-адреса для проверки доступности CMS

Запускаем сервер prometheus
docker-compose up -d

Подключаемся к веб-интерфейсу сервера prometheus по http://localhost:9090

Проверяем доступность CMS:

![s1](https://github.com/user-attachments/assets/ae07f9d4-2b14-472c-9a93-173fb296b65b)

и Exporter-ов

![s2](https://github.com/user-attachments/assets/888b9aac-0de2-4945-8811-db82cca1eecb)




