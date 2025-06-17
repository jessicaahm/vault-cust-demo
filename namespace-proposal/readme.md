Reference:

Policy Templating: https://developer.hashicorp.com/vault/tutorials/policies/policy-templating

```sh
#search docker
docker search hashicorp/vault-enterprise
```

```sh
#pull docker image enterprise latest version #1.19.5
docker pull hashicorp/vault-enterprise:latest
```

```sh
# Startup Vault
docker run --cap-add=IPC_LOCK -d --rm --name vault-enterprise -e "VAULT_DEV_ROOT_TOKEN_ID=root" -e "VAULT_DEV_LISTEN_ADDRESS=:8200" -e "VAULT_LICENSE=${VAULT_LICENSE}" -e "VAULT_ADDR=http://0.0.0.0:8200" -p 8200:8200 hashicorp/vault-enterprise:latest
```

```sh
# Create Namespace
vault login root

# Create /root/admin namespace
vault namespace create admin

# Create /root/admin/tenant1 namespace
VAULT_NAMESPACE=/admin vault namespace create tenant1
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
# Read-only permission on secrets stored at 'tenant1-sharedsecrets/data/retail-banking'
path "tenant1-secrets/data/retail-banking" {
  capabilities = [ "read" ]
}
EOF
```

```sh
VAULT_NAMESPACE=/admin/tenant1 vault policy list
```

```sh
# Create a named role
VAULT_NAMESPACE=/admin/tenant1 vault write auth/approle/role/tenant1-sharedsecrets \
    token_type=batch \
    secret_id_ttl=10m \
    token_ttl=20m \
    token_max_ttl=30m \
    token_policies="tenant1-readonly" \
    secret_id_num_uses=40

```

```sh
VAULT_NAMESPACE=/admin/tenant1 vault read auth/approle/role/tenant1-sharedsecrets
```

```sh
# Get role-id
VAULT_NAMESPACE=/admin/tenant1 vault read auth/approle/role/tenant1-sharedsecrets/role-id
# Get secret-id
VAULT_NAMESPACE=/admin/tenant1 vault write -force auth/approle/role/tenant1-sharedsecrets/secret-id
```

```sh
# Login
VAULT_NAMESPACE=/admin/tenant1 vault write auth/approle/login role_id="2ddc845b-cf46-2449-6f23-9c0322112696" \
    secret_id="baeec36e-ad67-9fa1-9fbb-c8592e5de7dd"

```

```sh
VAULT_NAMESPACE=/admin/tenant1 vault login hvb.AAAAAQKLOPePo8Z-A06W2Bd07S68GpzFaGud_SfPbA8r9xPNLDkt4uDn7aOy5AD_WKhLhNXXEWHv1jfubiZRhCftrBcxIWhLaQVIDDTxlq27AtZ_QsNmep8S1lq9nt0K_95HoXoQTsGVHunrslAvuhX8U8gbkkP-ehm7g0NpwXWk8Ht0pSYjw3F39YF_gHp14TBr0TiZnvPyDg6QxPf9-WEFf-W7YKV-4nYEqTf6GFwkeL98NBwDSpBcWlKveYZkEusDxrFdEA3pIliJDBA.f5Vrs
VAULT_NAMESPACE=/admin/tenant1 vault kv get -mount=tenant1-secrets retail-banking
```

```sh
# Enable  secrets in /tenant1 namespace
#VAULT_NAMESPACE=/admin/tenant1 vault secrets enable -path tenant1-secrets -version=2 kv

# Write secrets for tenant1
VAULT_NAMESPACE=/admin/tenant1 vault kv put tenant1-secrets/retail-banking project="retail-banking" team="teamname1"
```

```sh
vault login root
# Create an entity in the /admin namespace
VAULT_NAMESPACE=/admin vault auth enable -path="userpass-tenant1" userpass
```

```sh
#Policy tenant1 readonly
VAULT_NAMESPACE=/admin vault policy write tenant1-readonly -<<EOF
# Read-only permission on secrets stored at 'shared-secrets/data/teamname1'
path "shared-secrets/data/teamname1" {
  capabilities = [ "read" ]
}
EOF
```

```sh
# configure the userpass-tenant1
VAULT_NAMESPACE=/admin vault write auth/userpass-tenant1/users/celine password="celinepwd" policies="tenant1-readonly"
```

```sh
#List Auth method enabled
VAULT_NAMESPACE=/admin vault auth list -detailed
```

```sh
# Create entity > Store Accessor
#VAULT_NAMESPACE=/admin vault auth list -format=json | jq -r '.["userpass-tenant1/"].accessor' > accessor_tenant1.txt

# Create entity called celine-entity
# VAULT_NAMESPACE=/admin vault write -format=json identity/entity name="celine-entity" policies="tenant1-readonly" \
#      metadata=project="retail-banking" \
#      metadata=team="teamname1" \
#      | jq -r ".data.id" > entity_id_tenant1.txt

# Create entity-alias
VAULT_NAMESPACE=/admin vault write identity/entity-alias name="celine" \
     canonical_id=$(cat entity_id_tenant1.txt) \
     mount_accessor=$(cat accessor_tenant1.txt) \
     custom_metadata=account="shared-secrets-tenant1"
```

```sh
VAULT_NAMESPACE=/admin vault read -format=json identity/entity/id/$(cat entity_id_tenant1.txt) | jq -r ".data"
```

```sh
#CLEAN-UP
#VAULT_NAMESPACE=/admin vault read identity/entity-alias/id/badf77f7-7b14-f370-7487-865e495f784a

VAULT_NAMESPACE=/admin vault delete identity/entity-alias/id/badf77f7-7b14-f370-7487-865e495f784a
```

```sh
VAULT_NAMESPACE=/admin vault login -format=json -method=userpass -path=userpass-tenant1 \
    username=celine | jq -r ".auth.client_token" > celine_token.txt

```