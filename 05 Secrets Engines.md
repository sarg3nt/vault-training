<!-- omit from toc -->
# [Secrets Engines](https://developer.hashicorp.com/vault/docs/secrets)

Secrets engines are components which store, generate, or encrypt data. Secrets engines are incredibly flexible, so it is easiest to think about them in terms of their function. Secrets engines are provided some set of data, they take some action on that data, and they return a result.

- [Installing and Managing](#installing-and-managing)
- [Configuring a Secrets Engine](#configuring-a-secrets-engine)
- [KV](#kv)
- [Transit](#transit)
- [Database Secrets Engine](#database-secrets-engine)
- [PKI](#pki)
- [Cubbyhole](#cubbyhole)
  - [Cubbyhole Response Wrapping](#cubbyhole-response-wrapping)
- [Identity Secrets Engine](#identity-secrets-engine)

## Installing and Managing

```bash
vault secrets enable <engine-name>
vault secrets enable -path=<path> -description="<description" <engine-name>
vault secrets enable -path=sel-kv/ -description="My cool secrets engine" kv-v2
vault secrets enable -path=secret kv-v2
# Upgrade kv v1 to kv-v2
vault kv enable-versioning <path>

vault secrets list -detailed
vault secrets disable <path>
vault secrets disable sel-kv/
vault secrets move <oldpath> <newpath>
vault secrets tune -default-lease-ttl=72h pki/
```

## Configuring a Secrets Engine

1. Configure Vault with access to the platform
1. Configure Roles based on permission needed

```bash
# Configuring AWS
vault write aws/config/root \
  access_key=jJKFHK8D9H89DSFFS0F9SDF \
  secret_key=wJadkf4k43K9dfsf \
  region=us-east-1

# Configuring a database secrets engine
vault write database/config/prod-database \
  plugin_name=mysql \
  connection_url="" \
  allowed_roles="role1,role2" \
  username="" \
  password="" 

# The token/user must have access to a policy that has READ access to the path, i.e. database/creds/data-consultant
# Example Policy:
path "database/creds/data-consultant" {
  capabilities = ["read"]
}

# See 'Database Secrets Engine' for examples of generating a credential
```

## [KV](https://developer.hashicorp.com/vault/docs/secrets/kv)

The kv secrets engine is a generic Key-Value store used to store arbitrary secrets within the configured physical storage for Vault. This backend can be run in one of two modes; either it can be configured to store a single value for a key or, versioning can be enabled and a configurable number of versions for each key will be stored.

```bash
### Enable KV secrets engine
# Enable a kv v1 instance at the secret path
vault secrets enable -path=secret kv
# Upgrade the v1 secret kv store to V2
vault kv enable-versioning secret/
# Create a kv-v2 engine at the kvv2 path
vault secrets enable -path=kvv2 -description="My description" kv-v2

### Store Data
# WARNING: vault kv put with a single key will update the entire secret not just the one key, potentially wiping out other keys
vault kv put secret/app1/dbconnection user=fred pass=123
vault kv put secret/app1/dbconnection2 @myfile.json # Read from a json file
# myfile.json
# {
#   "user": "bob",
#   "password": "We are legion!"
# }
vault kv metadata put -custom-metadata=abc=123 secret/app1/dbconnection

### Get Data
# Read a secret
vault kv get secret/app1/dbconnection
# Get as json for machine readable output
vault kv get -format=json secret/app1/dbconnection
# get a previous version of a secret (v2 only)
vault kv get -version=1 secret/app1/dbconnection
# Gets all of the metadata for every version at once
vault kv metadata secret/app1/dbconnection
# List paths and secrets.  If it ends with a / it is a path
vault kv list secret/
vault kv list secret/app1/

### Delete
vault kv delete secret/app1/dbconnection

### kv v2 secrets management
# Undelete 
vault kv undelete -version=2 secret/app1/dbconnection
# Do a rollback, note creates a new version.
vault kv rollback -version=1 secret/app1/dbconnection
# To update just the one key, do a patch
vault kv patch secret/app1/dbconnection pass=456
# Destroy the data epermanantly, requires version(s) only destroys that version
vault kv destroy -versions=1,2 secret/app1/dbconnection
# Delete the metadata and associated data
vault kv metadata delete secret/app1/dbconnection
```

## [Transit](https://developer.hashicorp.com/vault/docs/secrets/transit)

The transit secrets engine handles cryptographic functions on data in-transit. Vault doesn't store the data sent to the secrets engine. It can also be viewed as "cryptography as a service" or "encryption as a service". The transit secrets engine can also sign and verify data; generate hashes and HMACs of data; and act as a source of random bytes.

```bash
# Enable transit, this weill enable at the default path of transit/
vault secrets enable transit

### Keys
# Create an ecryption key
# This creates a key with the default encrypt engine of AES-256
vault write -f transit/keys/<key-name>
vault write -f transit/keys/training
# Specify an encrypt type
vault write -f transit/keys/<key-name> type="rsa-256"
# Rotate the encryption key
vault write -f transit/keys/training/rotate
# List keys
vault list transit/keys
# View key info
vault read transit/keys/training
# Key                       Value
# ---                       -----
# allow_plaintext_backup    false
# auto_rotate_period        0s
# deletion_allowed          false
# derived                   false
# exportable                false
# imported_key              false
# keys                      map[1:1682617139 2:1682618630]
# latest_version            2
# min_available_version     0
# min_decryption_version    1
# min_encryption_version    0
# name                      training
# supports_decryption       true
# supports_derivation       true
# supports_encryption       true
# supports_signing          false
# type                      aes256-gcm96

# Limit minimum version of a key to be used
# This will make the first version of the key not work
# This does not delete the old keys, we can turn them back on by setting this value to an older version (1)
vault write transit/keys/training/config \
  min_decryption_version=2

### Encrypt data
vault write transit/encrypt/training \
  plaintext=$(base64 <<< "Getting started with Hashicorp Vault")  
# Key            Value
# ---            -----
# ciphertext     vault:v1:Vx8+QwqQ6OpgGkaLr4u9yjaTUAQWasemRjpMyHVIyJKEQfP0W1eUhP0mcXA09oJ2QURw013J9yEm2TsLCt0LDu8=
# key_version    1

# Return as json and select the ciphertext
vault write transit/encrypt/training \
  plaintext=$(base64 <<< "Getting started with Hashicorp Vault") \
  -format=json | jq -r .data.ciphertext

### Decrypt data
vault write transit/decrypt/training \
  ciphertext="vault:v1:Vx8+QwqQ6OpgGkaLr4u9yjaTUAQWasemRjpMyHVIyJKEQfP0W1eUhP0mcXA09oJ2QURw013J9yEm2TsLCt0LDu8="
# Key          Value
# ---          -----
# plaintext    R2V0dGluZyBzdGFydGVkIHdpdGggSGFzaGljb3JwIFZhdWx0Cg==

# Decrypt, return as json, select just the text and base64 decode
vault write transit/decrypt/training \
  ciphertext="vault:v1:Vx8+QwqQ6OpgGkaLr4u9yjaTUAQWasemRjpMyHVIyJKEQfP0W1eUhP0mcXA09oJ2QURw013J9yEm2TsLCt0LDu8=" \
  -format=json | jq -r .data.plaintext | base64 -d 

### Rewrap Data
# This decrypts the data with the old key then encrypts with the new one
# This is assuming you have rotated the key, otherwise there is no point
vault write transit/rewrap/training \
  ciphertext="vault:v1:Vx8+QwqQ6OpgGkaLr4u9yjaTUAQWasemRjpMyHVIyJKEQfP0W1eUhP0mcXA09oJ2QURw013J9yEm2TsLCt0LDu8="
# Key            Value
# ---            -----
# ciphertext     vault:v2:cMrNSeVCOQE2S7Li0Nd1t2AcoiZunfQj+et+Jjv9TaF4THxxpTDBU6VG83v7SAAaqDbL1KdltnUWvidgaa7WVb0=
# key_version    2

### Policies
# Permit Encryption Operation
path "transit/encrypt/training" {
  capabilities = ["update"]
}
path "transit/decrypt/training" {
  capabilities = ["update"]
}
```

## [Database Secrets Engine](https://developer.hashicorp.com/vault/docs/secrets/databases)

- Dynamically creates and manages secrets.  
- Generated Credenetials are based on roles that provide a 1 to 1 mapping to a permission set in the database.
- Credentials are tied to a lease.  When the lease expires Vault deletes the creds from the database.
- Plugins are used to connect to different database engines.  If the specific plugin does not exist you can use the `custom` plugin to specify how to connect.

```bash
# Enable secrets engine
vault secrets enable database

# Configure a mysql connection.  Pressing enter will cause vault to validate the connection
vault write database/config/my-mysql-database \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(127.0.0.1:3306)/" \
    allowed_roles="my-first-role, my-second-role" \
    username="vaultuser" \
    password="vaultpass"

# Examine the config
vault read database/config/my-mysql-database
# We can then rotate the password we gave above with
vault write -f database/rotate-root/my-mysql-database 

# Create the my-role role on the `my-mysql-database` database connection we created above
# The creation_statements grants the permissions you want it to grant on the database
vault write database/roles/my-role \
    db_name=my-mysql-database \ 
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON <database>.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"
# Examine the config
vault read database/roles/my-role

# Get dynamic credentials for the database
vault read database/creds/<role-name>
vault read database/creds/my-role
# Key                Value
# ---                -----
# lease_id           database/creds/my-role/2f6a614c-4aa2-7b19-24b9-ad944a8d4de6
# lease_duration     1h
# lease_renewable    true
# password           yY-57n3X5UQhxnmFRP3f
# username           v_vaultuser_my-role_crBWVqVh2Hc1

# We can revoke the lease if we need to 
vault lease revoke database/creds/my-role/2f6a614c-4aa2-7b19-24b9-ad944a8d4de6
# Or we can do it by prefix
vault lease revoke -prefix database/creds/my-role
```

Static roles are a 1-1 mapping to an existing user in the database.  
Vault will not change the username as it does with normal roles, but it will rotate the password.  
This is used in legacy apps that have a static username you must use or in situatios where the roles and security have been set up already.

## [PKI](https://developer.hashicorp.com/vault/docs/secrets/pki)  

The PKI secrets engine generates dynamic X.509 certificates. With this secrets engine, services can get certificates without going through the usual manual process of generating a private key and CSR, submitting to a CA, and waiting for a verification and signing process to complete. Vault's built-in authentication and authorization mechanisms provide the verification functionality.

[Build your own certificate authority (CA)](https://developer.hashicorp.com/vault/tutorials/secrets-management/pki-engine)

```bash
# Enable the engine
vault secrets enable pki
# Tune the engine
vault secrets tune -max-lease-ttl=720h pki
# Generate an intermediate CSR
vault write -format=json \
  pki/intermediate/generate/internal \
  common_name="vault.ad.selinc.com Intermediate" \
  | jq -r .data.csr > pki_intermediate.csr

# Sign the cert request with internal root CA

# Import the signed intermediate cert
vault write pki/intermediate/set-signed \
  certificate=@intermediate.cert.pem

# Configure the CA and CRL urls
vault write pki/config/urls \
    issuing_certificates="https://certmaster.vault.ad.selinc.com:8200/v1/pki/ca" \
    crl_distribution_points="https://certmaster.vault.ad.selinc.com:8200/v1/pki/crl"

### Create Roles
# See https://developer.hashicorp.com/vault/api-docs/secret/pki#create-update-role for more information
vault write pki/roles/k8s_role \
  allowed_domains=apps.ad.selinc.com \
  allowed_subdomains=true \
  allow_bare_domain=false \
  max_ttl=720h \
  allow_localhost=true \
  organization=selinc \
  country=us

### Request a New Certificate
vault write pki/issue/k8s_role \
  common_name=foo.apps.ad.selinc.com \
  alt_names=foo2.aps.ad.selinc.com \
  max_ttl=720h
# This will output the serial_number, cert, private key, issuing_ca etc.  
# This will be the only time you will see the private_key

### Revoke a Cert
vault write pki/revoke serial_number="4d:00:01 . . ."

### Cleanup
# Keep the storage backend clean by periodically removing expired certs.
vault write pki/tidy tidy_cert_store=true tidy_revoked_certs=true

```

See the above linked docs for more information

## [Cubbyhole](https://developer.hashicorp.com/vault/docs/secrets/cubbyhole)

The cubbyhole secrets engine is used to store arbitrary secrets within the configured physical storage for Vault namespaced to a token.
In cubbyhole, paths are scoped per token. No token can access another token's cubbyhole. When the token expires, its cubbyhole is destroyed.

Also unlike the kv secrets engine, because the cubbyhole's lifetime is linked to that of an authentication token, there is no concept of a TTL or refresh interval for values contained in the token's cubbyhole.

Writing to a key in the cubbyhole secrets engine will completely replace the old value.

```bash
# Write data to cubbyhole
vault write cubbyhole/<name>  <key>=<value> <key>=<value>
vault write cubbyhole/training user=fred pass=abc123

# Read data from cubbyhole
vault read cubbyhole/training 
# Key     Value
# ---     -----
# pass    abc123
# user    fred
```

### [Cubbyhole Response Wrapping](https://developer.hashicorp.com/vault/tutorials/secrets-management/cubbyhole-response-wrapping)

Scenario.  You need to get secrets to a user that does not have vault access but do not want to transmit them over third party systems.

Response wrapping creates a single use token with a short TTL to store the secrets in.  The wrapping token can then be sent to the user via Teams, email, etc.  The user can then use that token to get the secret and because it is single use it will then auto destruct.

```bash
# Get a wrapped KV secret
# Only thing we need to add that is different is the -wrap-ttl
vault kv get -wrap-ttl=5m secret/data/app1/user1
# Key                              Value
# ---                              -----
# wrapping_token:                  hvs.CAESIK7wg9P-OdczciFN_Gy_8NYu5CVhFw0g7lutpVyz39WAGh4KHGh2cy5LdUx1UmwzUDIxRXZmOU9VdlRpU29zMm4
# wrapping_accessor:               Xplm8tlhAr3NR8IAcEENYxqc
# wrapping_token_ttl:              5m
# wrapping_token_creation_time:    2023-05-03 13:48:42.478079432 -0700 PDT
# wrapping_token_creation_path:    secret/data/app1/user1

# NOTE: the -wrap-ttl command can be used on most (all?) of the secrets engines.

### Unwrapping by the end user
# End user could then use the vault cli to unwrap the data.  They would need to already have access to vault with at least policy=default
vault unwrap hvs.CAESIK7wg9P-OdczciFN_Gy_8NYu5CVhFw0g7lutpVyz39WAGh4KHGh2cy5LdUx1UmwzUDIxRXZmOU9VdlRpU29zMm4
# Key         Value
# ---         -----
# data        map[pass:123]
# metadata    map[created_time:2023-04-26T21:43:24.692896959Z custom_metadata:<nil> deletion_time: destroyed:false version:1]

# The user could also use the Vault UI but they would need a token to access it.  You can generate a token for them by running:
vault token create -policy=default
# This would generate a token with the default policy which has very little access, but just enough to use unwrapping.  
# Go to 'Tools" --> "Unwrap" and enter the token
```

## [Identity Secrets Engine](https://developer.hashicorp.com/vault/docs/secrets/identity)

The Identity secrets engine is the identity management solution for Vault. It internally maintains the clients who are recognized by Vault. Each client is internally termed as an Entity. An entity can have multiple Aliases. For example, a single user who has accounts in both GitHub and LDAP, can be mapped to a single entity in Vault that has 2 aliases, one of type GitHub and one of type LDAP. When a client authenticates via any of the credential backends (except the Token backend), Vault creates a new entity and attaches a new alias to it, if a corresponding entity doesn't already exist. The entity identifier will be tied to the authenticated token. When such tokens are put to use, their entity identifiers are audit logged, marking a trail of actions performed by specific users.

See [docs](https://developer.hashicorp.com/vault/docs/secrets/identity) for more information.

```bash
# Create a new entity
vault write identity/entity name="Bryan Krausen" policies="manager"
# Key        Value
# ---        -----
# aliases    <nil>
# id         1ec199dd-bf3a-4ee6-c560-c78050b9fbe6
# name       Bryan Krausen

# NOTE: This has no alias yet, so it's not really doing anything

# Create an alias that maps a user `bryan` to the above entity using its id
# name is the name of the username we are creating the alias for, example is this username might be a userpass username
# canonical_id is the id from the entity we created above
# mount_accessor is the Accessor from the auth method we are linking this to, find with `vault auth list`
vault write identity/entity-alias name="bryan" \ 
  canonical_id="1ec199dd-bf3a-4ee6-c560-c78050b9fbe6" \
  mount_accessor="auth_userpass_9d9c583b"

```
