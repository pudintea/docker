# MARIADB DOCKER LTS
Struktur Folder
```
.
├── docker-compose.yaml
├── data/
├── backup/
└── config/
    └── mariadb.cnf
```
Kode di docker-compose.yml
```
services:
  mariadb:
    image: mariadb:11.8
    container_name: mariadb

    restart: unless-stopped

    environment:
      TZ: Asia/Jakarta
      MARIADB_ROOT_PASSWORD: StrongRootPassword
      MARIADB_DATABASE: appdb
      MARIADB_USER: appuser
      MARIADB_PASSWORD: StrongUserPassword

    ports:
      - "3306:3306"

    volumes:
      - mariadb_data:/var/lib/mysql
      - ./conf.d:/etc/mysql/conf.d:ro
      - ./backup:/backup

    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 10

    networks:
      - db-network

volumes:
  mariadb_data:

networks:
  db-network:
    external: true
```
Kode untuk mariadb.cnf
```
[mysqld]

# Charset
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# Connection
max_connections = 500
connect_timeout = 30
wait_timeout = 28800
interactive_timeout = 28800

# InnoDB
innodb_buffer_pool_size = 1G
innodb_log_file_size = 256M
innodb_flush_method = O_DIRECT

# Query cache (MariaDB masih mendukung)
query_cache_type = 1
query_cache_size = 64M

# Packet
max_allowed_packet = 256M

# Logging
slow_query_log = 1
slow_query_log_file = /var/lib/mysql/slow.log
long_query_time = 2

# Timezone
default_time_zone = '+07:00'

[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4
```

## Cara Pengecekan 
Masuk ke container:
```
docker exec -it mariadb bash
```
Cek file konfigurasi:
```
mysql --help | grep my.cnf
```
Atau:
```
mariadb -uroot -p -e "SHOW VARIABLES LIKE 'max_connections';"
```
Hasilnya harus sesuai dengan nilai pada custom.cnf.

## Cara Penggunaan database:
```
DB_CONNECTION=mysql
DB_HOST=mariadb
DB_PORT=3306
DB_DATABASE=appdb
DB_USERNAME=appuser
DB_PASSWORD=StrongRootPassword
```
## Pudin Saepudin
