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
docker network ls
```

```sh
# Create a network for the pods to reside in
docker network create database
```

```sh
# Run mariadb 
docker-compose up

#docker-compose down

# Login via host.docker.internal:3306
```