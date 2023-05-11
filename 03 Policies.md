<!-- cSpell:ignore -->
# [Policies](https://developer.hashicorp.com/vault/docs/concepts/policies)

Everything in Vault is path-based, and policies are no exception. Policies provide a declarative way to grant or forbid access to certain paths and operations in Vault. This section discusses policy workflows and syntaxes.

Policies are deny by default, so an empty policy grants no permission in the system.

Users are granted policies at several different levels including group, entity and alias.

```bash
vault policy list
# default
# root

vault policy read default
# Allow tokens to look up their own properties
# path "auth/token/lookup-self" {
#    capabilities = ["read"]
# }
# . . . . 
# NOTE: Cannot read root policy

# Create a new policy using a pre-defined hcl file
vault policy write admin-policy /tmp/admin.hcl

# Using the API
curl \
  --header "X-Vault-Token: s.bckf4kdkds9dftk43ld9" \
  --request PUT \
  --data @payload.json \
  http://127.0.0.1:8200/v1/sys/policy/<policy-name>

# A policy that would give user the ability to read, update and delete kv secrets for jenkins
path "kv\data\apps\jenkins" {
    capabilities = ["read","update","delete"]
}

# List of Capabilities
["create","read","update","delete","list","sudo","deny"]
```

