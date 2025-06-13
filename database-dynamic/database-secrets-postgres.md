Dynamic secrets for database credential management tutorial
[link](https://developer.hashicorp.com/vault/tutorials/db-credentials/database-secrets)

```sh
docker pull postgres:latest
```

```sh
docker run \
    --detach \
    --name learn-postgres \
    -e POSTGRES_USER=root \
    -e POSTGRES_PASSWORD=rootpassword \
    -p 5432:5432 \
    --rm \
    postgres
```

```sh
docker ps -f name=learn-postgres --format "table {{.Names}}\t{{.Status}}"
```

```sh
# Create a role in postgres which can read all table
docker exec -i \
    learn-postgres \
    psql -U root -c "CREATE ROLE \"ro\" NOINHERIT;"
```

```sh
docker exec -i \
    learn-postgres \
    psql -U root -c "GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"ro\";"
```

```sh
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=root
export POSTGRES_URL=127.0.0.1:5432
```

```sh
vault server -dev -dev-root-token-id root
```

```sh
vault secrets enable database
```

```sh
vault secrets list --detailed
```

```sh
vault write database/config/postgres \
     plugin_name=postgresql-database-plugin \
     connection_url="postgresql://{{username}}:{{password}}@$POSTGRES_URL/postgres?sslmode=disable" \
     allowed_roles=readonly \
     allowed_roles=writeonly \
     username="root" \
     password="rootpassword"

```

```sh
tee readonly.sql <<EOF
CREATE ROLE "{{name}}" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}' INHERIT;
GRANT ro TO "{{name}}";
EOF

```

```sh
vault write database/roles/readonly \
      db_name=postgres \
      creation_statements=@readonly.sql \
      default_ttl=1h \
      max_ttl=24h

```

```sh
vault read database/creds/readonly
```

```sh
docker exec -i \
    learn-postgres \
    psql -U root -c "SELECT usename, valuntil FROM pg_user;"
```