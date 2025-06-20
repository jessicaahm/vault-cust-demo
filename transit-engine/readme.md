# Transit Engine - Allowing use of legacy key

Using HVAC Blog: https://medium.com/hashicorp-engineering/encryption-with-transit-data-keys-bfe5241ae194

```sh
export VAULT_ADDR=http://127.0.0.1:8200
```

```sh
# Create a policies for transit to manage transit engine
vault policy write transit-policy - <<EOD
# Enable transit secrets engine
path "sys/mounts/transit" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}

# To read enabled secrets engines
path "sys/mounts" {
  capabilities = [ "read" ]
}

# Manage the transit secrets engine
path "transit/*" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}
EOD
```

```sh
# Create a policies for transit to use transit (application)
vault policy write app-network -<<EOF
path "transit/encrypt/network" {
   capabilities = [ "update" ]
}
path "transit/decrypt/network" {
   capabilities = [ "update" ]
}
EOF

```

```sh
vault token create -policy=transit-policy
```

```sh
# Enable transit engine
vault login hvs.CAESIIqlpq9XhJ7vAVeWgp7y6t4i2gaakfNK6Yeu1RWwkxrnGh4KHGh2cy5qZlNUemxGVWNITU1hWTJrczNYWnFZRXY
vault secrets enable transit
```

```sh
# Create an encryption key named "network"
vault write -f transit/keys/network
```

```sh
export APP_ORDER_TOKEN=$(vault token create \
    -policy=app-network \
    -format=json | jq -r '.auth | .client_token')

```

```sh
export CIPHERTEXT=$(VAULT_TOKEN=$APP_ORDER_TOKEN vault write transit/encrypt/network \
    plaintext=$(base64 <<< "4111 1111 1111 1111")\
    -format=json | jq -r '.data | .ciphertext')
```

```sh
VAULT_TOKEN=$APP_ORDER_TOKEN vault write \
    transit/decrypt/network ciphertext=$CIPHERTEXT
```

```sh
base64 --decode <<< "NDExMSAxMTExIDExMTEgMTExMQo="
```

```sh
# Rotate encyrption key
vault write -f transit/keys/network/rotate
```

```sh
vault write transit/encrypt/network plaintext=$(base64 <<< "4111 1111 1111 1111")
```

```sh
vault write transit/rewrap/network \
    ciphertext=$CIPHERTEXT
```

```sh
vault read transit/keys/network
```

```sh
vault write transit/keys/network/config auto_rotate_period=24h
```

```sh
vault write transit/keys/network/config min_decryption_version=5
```

```sh
# Getting the data key
# Getting the data key
# Store the plaintext key in a variable
PLAINTEXT_KEY=$(vault write -f transit/datakey/plaintext/network -format=json | jq -r '.data.plaintext')

```

```sh
# Install hvac - python client for hashicorp vault: https://github.com/hvac/hvac
pip3 install hvac

```

```python
import hvac
import base64
from Crypto.Cipher import AES
from Crypto.Random import get_random_bytes
from Crypto.Util.Padding import pad, unpad

# Set up the client
client = hvac.Client(url='http://127.0.0.1:8200')
gen_key_response = client.secrets.transit.generate_data_key(
    name='network',
    key_type='plaintext',
)
ciphertext = gen_key_response['data']['ciphertext']
plaintext = gen_key_response['data']['plaintext']
print('Generated data key ciphertext is: {cipher}'.format(cipher=ciphertext)) # use this for decryption
print('Generated data key plaintext is: {cipher}'.format(cipher=plaintext)) # Use this for encryption

# Getting the data_key from plaintext to encrypt the blob
data_key = base64.b64decode(str(plaintext).encode('utf-8'))
print('Decoded data key plaintext is: {cipher}'.format(cipher=data_key))

iv = get_random_bytes(16)

cipher = AES.new(data_key, AES.MODE_CBC, iv)

# Encrypt the data
data=b'This is secret'
padded_data = pad(data, AES.block_size)
encrypted_data = cipher.encrypt(padded_data)
print(encrypted_data)

# Decrypt the data
# <there should be an extra step to convert ciphertext to plaintext here....>

cipher_dec = AES.new(data_key, AES.MODE_CBC, iv)
decrypted_padded = cipher_dec.decrypt(encrypted_data)
print(decrypted_padded)

```

### BYOK 
Import your target key

[What are the BYOK type of key supported](https://developer.hashicorp.com/vault/api-docs/secret/transit)

```sh
# Read the wrapping key so you can wrap the key for import
vault read -field=public_key transit/wrapping_key
```

```python {"terminalRows":"14"}
# Generate an AES key
from Crypto.Hash import CMAC
from Crypto.Cipher import AES
from Crypto.Random import get_random_bytes

from cryptography.hazmat.primitives.serialization import load_pem_public_key
from cryptography.hazmat.primitives.keywrap import aes_key_wrap_with_padding
from cryptography.hazmat.primitives.asymmetric import padding
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives import serialization, hashes


import hvac
import requests
import base64

client = hvac.Client(url='http://127.0.0.1:8200')

# store the target key = targetkey_128
targetkey_128 = get_random_bytes(16)

print("AES 128-bit key:", targetkey_128.hex())

# get the wrapping key

VAULT_ADDR = 'http://127.0.0.1:8200'
VAULT_TOKEN = 'root'

headers = {
    'X-Vault-Token': VAULT_TOKEN,
}

resp = requests.get(f'{VAULT_ADDR}/v1/transit/wrapping_key', headers=headers)
resp.raise_for_status()
data = resp.json()

public_key_pem = data['data']['public_key']
print(public_key_pem.encode('utf-8'))

# Wrap the wrapping key and Load the RSA public key from PEM
# wrapping_key = load_pem_public_key(public_key_pem.encode())
wrapping_key = serialization.load_pem_public_key(
    public_key_pem.encode('utf-8'),
    backend=default_backend()
)

# generate an ephemeral AES key for wrapping the target key #need 256
ephemeralAESKey = get_random_bytes(32)
print("AES 128-bit key:", ephemeralAESKey.hex())

# wrapped target key
wrapped_target_key = aes_key_wrap_with_padding(ephemeralAESKey, targetkey_128, default_backend())

# Key wrap ephemeralAESKey with SHA-256 (Also SHA1, SHA384, SHA512)
wrapped_aeskey = wrapping_key.encrypt(
    ephemeralAESKey,
    padding.OAEP(
        mgf=padding.MGF1(algorithm=hashes.SHA256()),
        algorithm=hashes.SHA256(),
        label=None
    )
)

combined_ciphertext = wrapped_aeskey + wrapped_target_key
base64_ciphertext = base64.b64encode(combined_ciphertext).decode('utf-8')
print("base64Ciphertext:", base64_ciphertext)
```

```sh
ciphertext="WH3bMt7HAcEBxnWk+3u2i22UUWI3hH5maTrdoHAASIvwU5l201+Spq3fCKptjKdi0kglPUFO1jaYyibVDx/WD22obkDcyeVi+Ns6xyKibcvmq3Mi2xsrj0SrK3OUds+AIvdDBQCrDZxKxZAHA2jx71IuxwU6ct64wku5yQk9BINpXngc9Vfk3GL0Ek3DN+1YK4z56Ypt55BrvvPGOwuhRa3dEvj1M2fN13gZSGx5f3CSQYITp81ojIkXucdsTgdk0kxEXddlV5nOL//KTBTuETQ08Br9uH6D0nHZ0eH2fe1E3pxdmJ2mIvqKSA/RH08zFN8RhfwujkhvndnuLp8KR1hYxVoXW72L+ig50Kzl2xi3oi+65B/sjLKCLTH70x4VwP+eH3+Hbj73D+0ZMgI1LNJQIob6NxrbiB+A/NoA1+5QyApmZ3xyagrfaJbSn2/7K7HHw/1WhfP48aSRfbZ4y9ffq7FB6SFfiS/e/Duhmc6p1nsxcqXCUYK2v3WZ7wbuYAce+gXlujBYN0OPC8PuF0qGyjQPximzUFjcwuVr9kMyVPkTxWTswp4LowJveSe0FzQDv8f3h39wuNFyBwSNO3rpgFUmwbdA8PFPKcpqy48laqXGLG5va3c1gTsb4vIm9o3j5mzAEAHY6IX5g/8d0Nkq9PKwCmLE7h8XM4Al/RUwewqCvuGO6DTntCWWnUBZJ5ABx1e9nmw="
# vault write transit/keys/test-key/config deletion_allowed=true
# vault delete transit/keys/test-key
vault write transit/keys/test-key/import ciphertext=$ciphertext hash_function=SHA256 type=aes128-gcm96


```

```python
import requests
import base64
from Crypto.Random import get_random_bytes
from cryptography.hazmat.primitives.serialization import (
    Encoding, PrivateFormat, NoEncryption, load_pem_public_key
)
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.keywrap import aes_key_wrap_with_padding
from cryptography.hazmat.backends import default_backend

# Vault config
VAULT_ADDR = 'http://127.0.0.1:8200'
VAULT_TOKEN = 'root'
KEY_NAME = 'imported-rsa-1024'

# Step 1: Generate RSA 1024-bit private key and export to DER (PKCS#8)
private_key = rsa.generate_private_key(
    public_exponent=65537,
    key_size=2048,
    backend=default_backend()
)

private_key_der = private_key.private_bytes(
    encoding=Encoding.DER,
    format=PrivateFormat.PKCS8,
    encryption_algorithm=NoEncryption()
)

# Step 2: Generate an ephemeral AES-256 key and wrap the DER with AES-KWP
ephemeral_aes_key = get_random_bytes(32)
wrapped_private_key = aes_key_wrap_with_padding(
    ephemeral_aes_key,
    private_key_der,
    backend=default_backend()
)

# Step 3: Get Vault's wrapping RSA public key
resp = requests.get(
    f"{VAULT_ADDR}/v1/transit/wrapping_key",
    headers={"X-Vault-Token": VAULT_TOKEN}
)
resp.raise_for_status()
vault_public_key_pem = resp.json()['data']['public_key']

vault_wrapping_key = load_pem_public_key(
    vault_public_key_pem.encode('utf-8'),
    backend=default_backend()
)

# Step 4: Wrap the ephemeral AES key using RSA-OAEP + SHA-256
wrapped_aes_key = vault_wrapping_key.encrypt(
    ephemeral_aes_key,
    padding.OAEP(
        mgf=padding.MGF1(algorithm=hashes.SHA256()),  # Explicit MGF1 with SHA-256
        algorithm=hashes.SHA256(),                    # Explicit OAEP with SHA-256
        label=None
    )
)

# Step 5: Combine RSA-wrapped AES key + AES-wrapped target key, base64-encode
combined_ciphertext = wrapped_aes_key + wrapped_private_key
b64_ciphertext = base64.b64encode(combined_ciphertext).decode('utf-8')

# Step 6: Create payload and import key into Vault Transit
import_payload = {
    "type": "rsa-2048",               # Key type in Vault
    "ciphertext": b64_ciphertext,     # Wrapped key material
    "hash_function": "SHA256"       # REQUIRED for RSA-OAEP wrapped keys
}

headers = {
    "X-Vault-Token": VAULT_TOKEN,
    "Content-Type": "application/json"
}

# Step 7: Import the key
import_resp = requests.post(
    f"{VAULT_ADDR}/v1/transit/keys/{KEY_NAME}/import",
    json=import_payload,
    headers=headers
)

# Step 8: Check result
try:
    import_resp.raise_for_status()
    print("✅ RSA private key imported into Vault Transit successfully.")
except requests.exceptions.HTTPError as err:
    print(f"❌ Import failed: {import_resp.status_code} - {import_resp.text}")

```

```sh
# Encrypt with the new Key
vault write transit/encrypt/imported-rsa-1024 \
    plaintext=$(base64 <<< "4111 1111 1111 1111")\
    -format=json | jq -r '.data | .ciphertext'


```

```sh
#Decrypt
vault write \
    transit/decrypt/imported-rsa-1024 ciphertext="vault:v1:Fgv5SaCAJPSuhUKsaLdACBXaxKB1YKcIITHKkQqQtTufw+pfGKUfH4HQ8jUiF/YLc5lYPSZLytqjnsVxasqOYr51hGidt9ZXhesJ6MyQXIKqWkfP9aLOfYPd8a1HuTDx/pvYV7bl9rnVsEBn+OeXIFL4FDgtugivUb1YEIqdyLdzuTA0y2pqlwOzxP7vhVa5Rxg9ImcEyljJmHELlR7vJLVO5gmR5UHginoAWo5bEWiaE9NzmuFJjyFGL7oj+K+OBH6Q9R1zc6XNHv3KMvJuaQOsCJPDEH59mko2zg0cxitfsrb6oa7D8DNSb6hM/VZyP9YUfWKqaqwRFtvHXuKxxw=="
```

```sh
base64 --decode <<< "NDExMSAxMTExIDExMTEgMTExMQo="
```