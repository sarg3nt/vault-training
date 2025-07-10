<!-- omit from toc -->
# Maintenance

- [Rekey Vault](#rekey-vault)
  - [What is Rekey](#what-is-rekey)
  - [Why Rekey](#why-rekey)
  - [Impact to Production](#impact-to-production)
  - [Rekey Procedure](#rekey-procedure)
- [Key Rotation](#key-rotation)

## Rekey Vault

### What is Rekey

> **NOTE:** Can not rekey vault without the minimum number of unseal keys

- Creates a new set of Vault recovery/unseal keys
- Allows you to specify the number of keys and threshold during the rekey process. Does not have to match current number of keys and threshold
- Requires a threshold of keys to successfully rekey (similar to an unseal or root token generation)
- Provides a nonce value to be given to key holders

### Why Rekey

- One or more keys are lost
- Employees leave the organization
- Your organization requires that you rekey after a period of time as a security measurement

### Impact to Production

- The Rekey operation is an online operation and can be done anytime
- This means no downtime for performing this operation
- Vault continues to service requests during the rekey operation

[Slides 315-327](operations-training/01-Create-a-working-Vault-server-configuration-given-a-scenario.pdf)

### Rekey Procedure

```bash
### 1. Initialize Rekey process
# Can provide -key-shares, -key-threshold and -pgp-keys
# For Shamir key shard clusters
vault operator rekey -init
# For auto unseal clusters we must tell it we want new recovery keyss
vault operator rekey -init -target=recovery
Key                      Value
---                      -----
Nonce                    847122f2-5537-c676-f9b7-fb545deb4c7a
Started                  true
Rekey Progress           0/3
New Shares               5
New Threshold            3
Verification Required    false

### 2. Key guardians must run below up to the threshold
# If this is an auto unseal instance then -target=recovery must be added
vault operator rekey -target=recovery
Rekey operation nonce: 847122f2-5537-c676-f9b7-fb545deb4c7a
Unseal Key (will be hidden):

Recovery Key 1: ba5npTK+uEku/aE087sAx7n72MHxFnhpcol4NlIdvMUi
Recovery Key 2: iQoUq7jfULeFdDu3h+VgxRaviAXVdzEi8VL4qefWXy3x
Recovery Key 3: d5ejDsXRnKcgRoQXniiPmMY/q3xnPVIjv+lgw7LJlEwU
Recovery Key 4: 2iBPW6iUuFb9XVSMEOQkN/rX/u771vHxmtPSZT+3zNkj
Recovery Key 5: aFWaHOMgGCEYbVmOykjwNQtZxyMueRIkNWRwQIbd2FeD

Operation nonce: 847122f2-5537-c676-f9b7-fb545deb4c7a
```

## Key Rotation

Rekeying vault creates a new master key on the back end (which protects the encryption key). Key rotation creates a new Encryption key.

- Rotates the encryption key that is used to protect data on the storage backend
- Since the encryption key is never available to users or operators, it does NOT require a threshold of key holders to rotate
- New keys are added to the keyring - old values can still be decrypted with the old key

```bash
# Check the key status
vault operator key-status
Key Term            1
Install Time        08 May 23 14:12 UTC
Encryption Count    2

# Rotate the key
vault operator rotate
Success! Rotated key
Key Term            2
Install Time        08 May 23 17:17 UTC
Encryption Count    1
```

Need `sys/rotate` sudo access to rotate the key

```bash
path "sys/rotate" {
  capabilities = ["update","sudo"]
}
# Thi is needed by the CLI to read the status of the key after rotating it, not to rotate it
path "sys/key-status" {
  capabilities = ["read"]
}
```
