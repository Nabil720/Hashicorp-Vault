# Vault KV Secrets Engine Setup 


## 1. Create KV v2 Secrets Engine

Enable a new KV v2 secrets engine with extended TTL:

```bash
# Enable KV v2 at path 'Demo-name'
vault secrets enable -path=Demo-name -version=2 kv

# Configure extended TTL for this secrets engine
vault secrets tune -default-lease-ttl=876000h -max-lease-ttl=876000h Demo-name/
```
## 2. Create Initial Secret

Store a sample secret in the new engine:

```bash
vault kv put Demo-name/auth-service password=mypass
```
## 3. Create Access Policy

Create a policy file (policy.hcl) with the following content:

```bash
cat <<EOF > policy.hcl
path "Demo-name/data/auth-service" {
  capabilities = ["create", "read", "update", "list"]
}

path "Demo-name/metadata/auth-service" {
  capabilities = ["list"]
}
EOF
```
## Apply the policy:

```bash
vault policy write policy policy.hcl
```


## 4. Adjust System-Wide Token TTL Limits

First, increase the global token TTL limits from default (768h) to 99 years:

```bash
# Check current token authentication settings
vault read sys/auth/token/tune

# Increase max_lease_ttl to 99 years (876,000 hours)
vault write sys/auth/token/tune max_lease_ttl=876000h
```
Note: This requires root/admin privileges as it changes system-wide settings.

## 5. Generate Token

Create a token with the policy attached and TTL:

```bash
vault token create -policy="Demo-name-policy" -ttl=876000h
```

## Example output:

```bash
Key                  Value
---                  -----
token                hvs.CAESIC5oySu03lvAoF8Z-B*****
token_accessor       qHteq8hauqVdW0OIN6xyEQz9
token_duration       876000h
token_renewable      true
token_policies       ["default" "Demo-name-policy"]
```

# Verification On Other VM

## 1. Test Token Authentication

```bash
export VAULT_ADDR="http://192.168.121.131:8200/"
export VAULT_SKIP_VERIFY=true

vault login hvs.CAESIC5oySu03lvAoF8Z-******
```

## Example output:

```bash
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                hvs.CAESIC5oySu03lvAoF8Z****
token_accessor       qHteq8hauqVdW0OIN6xyEQz9
token_duration       875999h20m53s
token_renewable      true
token_policies       ["default" "Demo-name-policy"]
identity_policies    []
policies             ["default" "Demo-name-policy"]
```

## 2. Verify Token Capabilities

```bash
# Check capabilities on secret data
vault token capabilities Demo-name/data/auth-service
# Expected: create, list, read, update

# Check capabilities on metadata
vault token capabilities Demo-name/metadata/auth-service
# Expected: list
```

## 3. Test Secret Operations

```bash
# Read secret
vault kv get Demo-name/auth-service

# Update secret
vault kv put Demo-name/auth-service password=newpassword
```
