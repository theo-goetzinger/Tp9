## Théo Goetzinger

## TP9

## prometheus.yml 

```

# my global config
global:
    scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
    evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
    # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
    alertmanagers:
    - static_configs:
      - targets:
        # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
    # - "first_rules.yml"
    # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
    # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
    - job_name: 'prometheus'

      # metrics_path defaults to '/metrics'
      # scheme defaults to 'http'.

      static_configs:
        - targets: ['localhost:9090']

    - job_name: 'mysql'
        # If prometheus-node-exporter is installed, grab stats about the local
        # machine by default.
      static_configs:
        - targets: ['exporter:9104']


```

## docker-compose.yml : 

```
version: '3.7'

services:
  maria:
    image: mariadb
    restart: on-failure
    environment:
      MYSQL_ROOT_PASSWORD: password
    volumes:
        - ./scripts:/docker-entrypoint-initdb.d
        - ./mysql:/var/lib/mysql
        - ./sql/user.sql:/docker-entrypoint-initdb.d/user.sql
        - ./sql/sylius_dev.sql:/docker-entrypoint-initdb.d/sylius_dev.sql

    ports:
      - 3306
    networks:
     - bdd
     - exporter
     - prometheus


  exporter:
    image: prom/mysqld-exporter
    restart: on-failure
    environment:
      DATA_SOURCE_NAME: "exporter:password@(db:3306)/"
    ports:
      - 9104
    networks:
      - bdd
      - exporter
    depends_on:
      - maria

  prometheus:
    image: prom/prometheus
    restart: on-failure
    volumes:
      - ./configs/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    networks:
      - bdd
      - exporter
      - prometheus
    depends_on:
      - exporter
      - maria

  grafana:
  image: grafana/grafana
    restart: on-failure
    volumes:
      - grafana_data_tp9:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_USERS_ALLOW_SIGN_UP: "false"
    ports:
      - "3000:3000"
    networks:
      - prometheus
      - bdd
    depends_on:
      - prometheus
      - maria

volumes:
  mariadb_tp9:
  grafana_data_tp9:

networks:
  bdd:
  exporter:
  prometheus:


##  Script user.sql 

```

CREATE USER 'exporter'@'%' IDENTIFIED BY 'password' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';

CREATE USER 'grafana'@'%' IDENTIFIED BY 'grafana';
GRANT SELECT ON sylius_dev.* TO 'grafana'@'%';
GRANT SELECT ON mysql.* TO 'grafana'@'%';


```

## sylius_dev.sql qui permet d'initialiser la bdd pour le tp (je le met pas ici car trop de ligne)

## datasource.yml

```
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    orgId: 1
    url: http://prometheus:9090
    basicAuth: false
    isDefault: true
    editable: true

  - name: MySQL
    type: mysql
    url: maria:3306
    database: sylius_dev
    user: grafana
    password: grafana
    jsonData:
      maxOpenConns: 0
      maxIdleConns: 2
      connMaxLifetime: 14400


```
## lancement du docker-compose

```
sudo docker-compose up -d

```
##connexion à grafana via l'interface web 

```
127.0.0.1:3000

```
## Soucis

```
j'ai accès au grafana, mais il est impossible de récupérer les données de la bdd donc de créer un dashboard, voici le message d'erreur que nous avions vu ensemble je n'ai malheuresement pas trouvé de fix pour le problème depuis mais mon code est bon odnc je ne voit pas d'ou viens le soucis 

grafana_1     | t=2020-11-30T10:54:23+0000 lvl=info msg="Request Completed" logger=context userId=1 orgId=1 uname=grafana method=POST path=/api/tsdb/query status=400 remote_addr=172.29.0.1 time_ms=270 size=245 referer=http://127.0.0.1:3000/datasources/edit/2/


```