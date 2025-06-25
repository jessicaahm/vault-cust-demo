Reference:

Policy Templating: https://developer.hashicorp.com/vault/tutorials/policies/policy-templating

Managing secrets across namespace:  https://developer.hashicorp.com/vault/tutorials/enterprise/namespaces-secrets-sharing

Set up vault

```sh
#search docker
docker search hashicorp/vault-enterprise
```

```sh
#pull docker image enterprise latest version #1.19.5
docker pull hashicorp/vault-enterprise:latest
```

```sh
./load_vault_license.sh
```

```sh
# Startup Vault
docker run -d --rm --name vault-enterprise -e "VAULT_DEV_ROOT_TOKEN_ID=root" -e "VAULT_DEV_LISTEN_ADDRESS=:8200" -e "VAULT_LICENSE=${VAULT_LICENSE}" -e "VAULT_ADDR=http://0.0.0.0:8200" -p 8200:8200 hashicorp/vault-enterprise:latest
```

```sh
# Create Namespace
vault login root

# Create /root/admin namespace
vault namespace create admin

# Create /root/admin/tenant1 namespace
VAULT_NAMESPACE=/admin vault namespace create tenant1

# Create /root/admin/tenant1 namespace
VAULT_NAMESPACE=/admin vault namespace create tenant2

# Create /root/admin/tenant1 namespace
VAULT_NAMESPACE=/admin vault namespace create tenant-sharedsecret
```

```sh
# Enable shared secrets in /admin namespace
VAULT_NAMESPACE=/admin vault secrets enable -path shared-secrets -version=2 kv
```

```sh
# Write secrets for teamname1
VAULT_NAMESPACE=/admin vault kv put shared-secrets/teamname1 project="retail-banking" team="teamname1"
```

```sh
# Write secrets for teamname2
VAULT_NAMESPACE=/admin vault kv put shared-secrets/teamname2 project="personal-banking" team="teamname2"
```

```sh
VAULT_NAMESPACE=/admin vault kv list shared-secrets
```

```sh
# enable approle in tenant1 == teamname1
VAULT_NAMESPACE=/admin/tenant1 vault auth enable approle
```

```sh
vault login root
#Policy tenant1 readonly
VAULT_NAMESPACE=/admin/tenant1 vault policy write tenant1-readonly -<<EOF
# Read-only permission on secrets stored at 'tenant1-secrets/data/retail-banking'
path "tenant1-secrets/data/retail-banking" {
  capabilities = [ "read" ]
}

EOF
```

```sh
vault login root
#Policy /admin/tenant-sharedsecret readonly
VAULT_NAMESPACE=/admin/tenant-sharedsecret vault policy write tenant-sharedsecret-readonly -<<EOF
path "*" {
capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
EOF
# Read-only permission on secrets stored at 'shared-secrets/data/retail-banking'
# path "shared-secrets/data/*" {
#   capabilities = [ "read" ]
# }

# path "shared-secrets/metadata/*" {
#   capabilities = [ "read" ]
# }

# path "sys/internal/ui/*" {
#   capabilities = [ "read" ]
# }
```

```sh
VAULT_NAMESPACE=/admin/tenant1 vault policy list
```

```sh
#Policy /admin readonly
VAULT_NAMESPACE=/admin vault policy write tenant1-readonly -<<EOF
# Read-only permission on secrets stored at 'shared-secrets/data/teamname1'
path "shared-secrets/data/teamname1" {
  capabilities = [ "read" ]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOF
```

```sh
# Enable  secrets in /tenant1 namespace
VAULT_NAMESPACE=/admin/tenant1 vault secrets enable -path tenant1-secrets -version=2 kv
# Write secrets for tenant1
VAULT_NAMESPACE=/admin/tenant1 vault kv put tenant1-secrets/retail-banking project="retail-banking" team="teamname1"

```

```sh
# Enable  secrets in /enant-sharedsecret namespace
VAULT_NAMESPACE=/admin/tenant-sharedsecret vault secrets enable -path shared-secrets -version=2 kv
# Write secrets for tenant1
VAULT_NAMESPACE=/admin/tenant-sharedsecret vault kv put shared-secrets/retail-banking project="retail-banking" team="teamname1"

```

```sh
# read all secrets
VAULT_NAMESPACE=/admin/tenant1 vault kv get -mount=tenant1-secrets retail-banking
VAULT_NAMESPACE=/admin/tenant1 vault kv get -mount=tenant1-secrets retail-banking
VAULT_NAMESPACE=/admin vault kv get -mount=shared-secrets teamname1
```

Create entity and groups

```sh
# create group in /admin
#VAULT_NAMESPACE=/admin vault write -format=json identity/group name="teamname1" policies="tenant1-readonly" member_entity_ids='["26744adf-4ee7-43a0-680e-1560533bd32c","88e63956-bd99-d5a4-8edd-205e1448d1e5"]'

VAULT_NAMESPACE=/admin vault write -format=json identity/group \
  name="teamname1" \
  policies="tenant1-readonly" \
  member_entity_ids=[26744adf-4ee7-43a0-680e-1560533bd32c,88e63956-bd99-d5a4-8edd-205e1448d1e5]
```

```sh
VAULT_NAMESPACE=/admin/tenant1 vault list identity/entity/id
```

```sh
# create group in /admin/tenant-sharedsecret
VAULT_NAMESPACE=/admin/tenant-sharedsecret vault write -format=json identity/group name="teamname1" policies="tenant-sharedsecret-readonly" member_entity_ids=$(cat entity_id.txt)


```

```sh
VAULT_NAMESPACE=/admin vault read identity/group/name/teamname1
```

```sh
# Create entity (approle) in /admin/tenant1 namespace
#VAULT_NAMESPACE=/admin/tenant1 vault auth list -format=json | jq -r '.["approle/"].accessor' > accessor.txt
#VAULT_NAMESPACE=/admin/tenant1 vault write -format=json identity/entity name="teamname1" | jq -r ".data.id" > entity_id.txt
VAULT_NAMESPACE=/admin/tenant1 vault write identity/entity-alias name="approle1" canonical_id=$(cat entity_id.txt) mount_accessor=$(cat accessor.txt)

```

```sh
VAULT_NAMESPACE=/admin/tenant1 vault read identity/entity/id/$(cat entity_id.txt)
```

```sh
VAULT_NAMESPACE=/admin/tenant1 vault read identity/entity-alias/id/81d2618b-f6ae-ed73-6c97-9ee54d626260
```

Approle create 

```sh
# Create a named approle
VAULT_NAMESPACE=/admin/tenant1 vault write auth/approle/role/approle1 \
    token_type=batch \
    secret_id_ttl=10m \
    token_ttl=20m \
    token_max_ttl=30m \
    token_policies="tenant1-readonly" \
    secret_id_num_uses=40

```

```sh
VAULT_NAMESPACE=/admin/tenant1 vault read auth/approle/role/approle1
```

```sh
# Get role-id
VAULT_NAMESPACE=/admin/tenant1 vault read auth/approle/role/approle1/role-id > role_id.txt
# Get secret-id
VAULT_NAMESPACE=/admin/tenant1  vault write -format=json -force auth/approle/role/approle1/secret-id > secret_id.txt
```

```sh
# Login
VAULT_NAMESPACE=/admin/tenant1 vault write -format=json auth/approle/login role_id='0691d495-ff8b-72d0-1bee-ca0446ad54d6' \
    secret_id=$(cat secret_id.txt | jq -r '.data.secret_id')  > approle_token.txt

```

```sh
# read all secrets
VAULT_TOKEN=$(cat approle_token.txt | jq -r '.auth.client_token')
VAULT_TOKEN=$VAULT_TOKEN VAULT_NAMESPACE=/admin/tenant-sharedsecret vault token lookup 

# VAULT_TOKEN=$VAULT_TOKEN VAULT_NAMESPACE=/admin/tenant1 vault kv get -mount=tenant1-secrets retail-banking
#VAULT_TOKEN=$VAULT_TOKEN VAULT_NAMESPACE=/admin/tenant1 vault kv get -mount=tenant1-secrets retail-banking
#VAULT_TOKEN=$VAULT_TOKEN VAULT_NAMESPACE=/admin vault kv get -mount=shared-secrets teamname1
#VAULT_TOKEN=$VAULT_TOKEN VAULT_NAMESPACE=/admin/tenant-sharedsecret  vault kv get -mount=shared-secrets retail-banking
VAULT_TOKEN=$VAULT_TOKEN VAULT_NAMESPACE=/admin/tenant-sharedsecret  vault kv get -mount=shared-secrets teamname1


```

```sh
VAULT_NAMESPACE=/admin/tenant1 vault read identity/entity/id/88e63956-bd99-d5a4-8edd-205e1448d1e5
```

```sh

```

```sh
vault login root
VAULT_TOKEN=$VAULT_TOKEN VAULT_NAMESPACE=/admin/tenant-sharedsecret  vault  kv get -mount=shared-secrets retail-banking

```

```sh
#CLEAN-UP

#Delete entity-alias created
VAULT_NAMESPACE=/admin vault delete identity/entity-alias/id/badf77f7-7b14-f370-7487-865e495f784a


```

```sh
VAULT_NAMESPACE=/admin vault login -format=json -method=userpass -path=userpass-tenant1 \
    username=celine | jq -r ".auth.client_token" > celine_token.txt

```

Set GPO to any

```sh
vault read sys/config/group-policy-application
```

```sh
vault write sys/config/group-policy-application \
   group_policy_application_mode="any"
```

Enable Audit Logs

```sh
#vault status
vault login root
VAULT_NAMESPACE=/admin vault audit enable file file_path="/var/logs/vault/audit.log"
```