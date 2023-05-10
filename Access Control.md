# Access Control

## Vault Identity Entities and Groups

### Entities

- Vault creates an entity and attaches an alias to it if a corresponding entity doesn't already exist.
- This is done using the Identity secrets engine, which manages internal identities that are recognized by Vault
- An entity is a representation of a single person or system used to log into Vault. Each has a unique value. Each entity is made up of zero or more aliases
- Alias is a combination of the auth method plus some identification.  It is a mapping between an entity and auth method(s)
- An entity can be manually created to map multiple entities for a single user to provide more efficient authorization management
- Any tokens that are created for the entity inherit the capabilities that are granted by alias(es)
- Not everyone creates manual entities as most of the time there's just one auth method enabled.  Can be useful if you have multiple accounts that you want to tie together to inherit privileges.
- See [slides 1-14](operations-training/07-Configure-Access-Control.pdf) for more information.

```bash
# Create a new entity that we will add aliases to
❯ vault write identity/entity name="Julie Smith" \
    policies="it-management" \
    metadata="organization"="HCVOP, Inc" \
    metadata="team"="management"
Key        Value
---        -----
aliases    <nil>
id         d1b44e30-10c7-8c32-9851-3207c0eb8895 # This is the canonical_id we'd use when creating an alias below
name       Julie Smith

# Add GitHub auth as an alias
❯ vault write identity/entity-alias \
    name="jsmith22" \
    canonical_id=<entity_id> \
    mount_accessor=<github_auth_accessor>

# Add LDAP auth as an alias
❯ vault write identity/entity-alias \
    name="jsmith@hcvop.com" \
    canonical_id=<entity_id> \
    mount_accessor=<ldap_auth_accessor>

# Find accessors for existing auths
❯ vault auth list
Path      Type     Accessor               Description                Version
----      ----     --------               -----------                -------
token/    token    auth_token_0dc86457    token based credentials    n/a
```

### Groups

- A group can contain multiple entities as its members
- A group can also have subgroups
- Policies can be set on the group and the permissions will be granted to all members of the group
- **Internal groups** are manually created and can be used to easily manage permissions for entities
  - Frequently used when using Vault Namespaces to propagate permissions down to child namespaces
  - Helpful when you don't want to configure an identical auth method on every single namespace
- **External groups** are manually or automatically created and are used to set permissions based on group membership from an external identity provider, such as LDAP, Okta, or OIDC provider.
  - Allows you to set up once in Vault and continue manage permissions in the identity provider.
  - Note that the group name must match the group name in your identity provide

## Vault Policies

- Vault policies provide operators a way to permit or deny access to certain paths or actions within Vault (RBAC)
  - Gives us the ability to provide granular control over who gets access to secrets
- Policies are written in declarative statements and can be written using JSON or HCL
- When writing policies, always follow the principal of least privilege
  - In other words, give users/applications only the permissions they need
- Policies are Deny by Default (implicit deny) - therefore you must explicitly grant to paths and related capabilities to Vault clients  
  > No policy = no authorization
  - Policies support an explicit DENY that takes precedence over any other permission
- Policies are attached to a token. A token can have multiple policies
- Policies are cumulative and capabilities are additive

### Out of the Box Policies

- **root** policy is created by default – superuser with all permissions
  - You cannot change nor delete this policy
  - Attached to all root tokens
- **default** policy is created by default – provides common permissions
  - You can change this policy, but it cannot be deleted
  - Attached to all non-root tokens by default (can be removed if needed)

```bash
# List all policies
❯ vault policy list
default
root

# Look at the default policy
❯ vault policy read default
# Allow tokens to look up their own properties
path "auth/token/lookup-self" {
    capabilities = ["read"]
}

# Allow tokens to renew themselves
path "auth/token/renew-self" {
    capabilities = ["update"]
    . . .

# NOTE: Cannot read root policy even though it exists
❯ vault policy read root
No policy named: root

# Create or update a policy
❯ vault policy write admin-policy /tmp/admin.hcl
Success! Uploaded policy: admin-policy

❯ vault policy write webapp -<< EOF
path "kv/data/apps/*" {
  capabilities = ["read","create","update","delete"]
}
path "kv/metadata/*" {
  capabilities = ["read","create","update","list"]
}
EOF

# Using the API
❯ curl \
  --header "X-Vault-Token: s.bckf4kdkds9dftk43ld9" \
  --request PUT \
  --data @payload.json \
  http://127.0.0.1:8200/v1/sys/policy/<policy-name>

# A policy that would give users the ability to read, update and delete kv secrets for jenkins
path "kv\data\apps\jenkins" {
    capabilities = ["read","update","delete"]
}

# List of Capabilities
["create","read","update","delete","list","sudo","deny"]
```

- See [slides 16-22](operations-training/07-Configure-Access-Control.pdf) for more information.

## Managing Policies 

- See [slides 23-31](operations-training/07-Configure-Access-Control.pdf) for more information.

## Anatomy of a Vault Policy

- See [slides 32-35](operations-training/07-Configure-Access-Control.pdf) for more information.

## Vault Polices - Path

- See [slides 36-40](operations-training/07-Configure-Access-Control.pdf) for more information.

## Vault Polices - Capabilities

- See [slides 41-47](operations-training/07-Configure-Access-Control.pdf) for more information.

## Customizing the Path

- See [slides 48-59](operations-training/07-Configure-Access-Control.pdf) for more information.

## Working with Policies

- See [slides 60-64](operations-training/07-Configure-Access-Control.pdf) for more information.

END OF SECTION

## Understand Sentinel Policies

- See [slides 66-80](operations-training/07-Configure-Access-Control.pdf) for more information.

END OF SECTION

## Define Control Groups and Describe their Basic Workflow

- See [slides 82-92](operations-training/07-Configure-Access-Control.pdf) for more information.

## Multi-Tenancy with Namespaces

- See [slides 93-109](operations-training/07-Configure-Access-Control.pdf) for more information.
