References:

[How to setup mariaDB and secure connections from clients](https://www.cyberciti.biz/faq/how-to-setup-mariadb-ssl-and-secure-connections-from-clients/)

[Vault dynamic database credentials](https://developer.hashicorp.com/vault/tutorials/db-credentials/database-secrets)

[Good Videos on why vault](https://www.youtube.com/watch?v=shLoMeYoUoU)

[TLS certificates auth methods](https://developer.hashicorp.com/vault/docs/auth/cert)

[Installing and Using MariaDB via Docker](https://mariadb.com/kb/en/installing-and-using-mariadb-via-docker/)

[How to setup mariadb](https://hub.docker.com/_/mariadb)

```sh
# Clean-up
docker stop $(docker ps -q)
docker rm $(docker ps -aq)
docker system prune -a --volumes
```

### Pre-requisite: Set up Environment

1. Set up Vault
2. Set up MariaDB

```sh
# Set up Vault with license key (Dev Mode)

vault server -dev -dev-root-token-id root -dev-tls
```

```sh
docker pull mariadb
docker pull ubuntu
```

```sh
#docker network ls

kubectl 
```

```sh
# Create a network for the pods to reside in
docker network create database
```

```sh
# Run mariadb 
docker-compose up

#docker-compose down
```

```sh
# Login via host.docker.internal:3306
# mysql -h host.docker.internal -P 3306 -u root -p

docker exec -it mariadb mariadb -u root -p

docker exec -it mariadb bash

```

```sh
# Generate ca cert (can be done in vault)
openssl genrsa 2048 > ca-key.pem
openssl req -new -x509 -nodes -days 365000 -key ca-key.pem -out ca-cert.pem

# create db cert req: :MariaDBAdmin
openssl req -newkey rsa:2048 -days 365000 -nodes -keyout server-key.pem -out server-req.pem
openssl rsa -in server-key.pem -out server-key.pem

# Sign the cert
openssl x509 -req -in server-req.pem -days 365000 -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 -out server-cert.pem

# Create client cert : MariaDBUser
openssl req -newkey rsa:2048 -days 365000 -nodes -keyout client-key.pem -out client-req.pem
openssl rsa -in client-key.pem -out client-key.pem

# Sign the cert
openssl x509 -req -in client-req.pem -days 365000 -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 -out client-cert.pem

# Verify cert
openssl verify -CAfile ca-cert.pem server-cert.pem client-cert.pem
```

```sh
# validate root ca
openssl s_client -connect 0.0.0.0:3306 -CAfile /usr/local/share/ca-certificates/ca-cert.crt
```

```sh
# Ensure SSL is added to server
mariadb -u root -p

SHOW VARIABLES LIKE '%ssl%';

```

```shell
MariaDB [(none)]> SHOW VARIABLES LIKE '%ssl%';

+---------------------+----------------------------------------------+
| Variable_name       | Value                                        |
+---------------------+----------------------------------------------+
| have_openssl        | YES                                          |
| have_ssl            | YES                                          |
| ssl_ca              | /usr/local/share/ca-certificates/ca-cert.crt |
| ssl_capath          |                                              |
| ssl_cert            | /etc/mysql/ssl/server-cert.pem               |
| ssl_cipher          |                                              |
| ssl_crl             |                                              |
| ssl_crlpath         |                                              |
| ssl_key             | /etc/mysql/ssl/server-key.pem                |
| version_ssl_library | OpenSSL 3.0.13 30 Jan 2024                   |
+---------------------+----------------------------------------------+

10 rows in set (0.001 sec)

```sh
#create a user that required certificate to login
CREATE USER 'clientuser'@'%' REQUIRE X509;
GRANT ALL PRIVILEGES ON *.* TO 'clientuser'@'%';
FLUSH PRIVILEGES;
```

```sh
mariadb -u clientuser -h 127.0.0.1 --ssl-ca=/etc/mysql/ssl/ca-cert.pem --ssl-cert=/etc/mysql/ssl/client-cert.pem --ssl-key=/etc/mysql/ssl/client-key.pem
```