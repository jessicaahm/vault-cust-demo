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
CREATE USER 'user1'@'host' IDENTIFIED BY 'password' REQUIRE X509;
CREATE USER 'user2'@'host' REQUIRE X509;
```