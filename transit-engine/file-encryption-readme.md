# File Encryption

```sh
# Using HVD

export VAULT_ADDR=${VAULT_ADDR}
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

Large File to use Data Key [Optional]

```sh
# Create the data key
export DATA_KEY=$(vault write -format=json -f transit/datakey/plaintext/file-key)
```

```sh
echo $DATA_KEY
```

```sh
# Decode the plaintext datakey and use it to encrypt a local file
echo $DATA_KEY | jq -r .data.plaintext | base64 --decode > key.bin
```

```sh
openssl enc -aes-256-cbc -salt -in sample.pdf -out sample.pdf.enc -pass file:./key.bin
```

```sh
# Store the ciphertext data key
echo $DATA_KEY | jq -r .data.ciphertext > ciphertext_data.key
```

```sh
# Decrypt the data key
vault write transit/decrypt/file-key ciphertext=$(cat ciphertext_data.key)

```

```sh
# Use the decrypted data key to store the file
echo "C0qxobhuqM4fl2sLOrtbkIdL1JFkR03Zgpu9I4rL+Oc=" | base64 --decode > key.bin
openssl enc -d -aes-256-cbc -in sample.pdf.enc -out sample_plaintext.pdf -pass file:./key.bin

```