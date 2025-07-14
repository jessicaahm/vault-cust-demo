# File Encryption 

```sh
# Using HVD

export VAULT_ADDR=https://vault-cluster-public-vault-a71d51f3.a6c54b84.z1.hashicorp.cloud:8200
export VAULT_NAMESPACE=admin
```

```sh
vault login
```

```sh
# Enable Transit Secrets Engine
vault secrets enable transit
```

```sh
# Create a file encryption key
vault write -f transit/keys/file-key
```

```sh
# Check file
du -sh sample.pdf
```

```sh
# Covert PDF to base64
ls
base64 -i sample.pdf > sample.txt.b64
```

```sh
vault write transit/encrypt/my-key plaintext=$(echo "my secret data" | base64)
```

```sh
# Encrypt a file using base64
vault write -format=json transit/encrypt/my-key plaintext=@sample.txt.b64 | jq -r .data.ciphertext > ciphertext.txt
```

```sh
# Decrypt file
vault write -format=json transit/decrypt/my-key ciphertext=@ciphertext.txt | jq -r .data.plaintext > plaintext.txt
```

```sh
# Decode the plaintext
base64 --decode -i plaintext.txt > plaintext.pdf
```