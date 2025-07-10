<!-- omit from toc -->
# Access Control

- [Identity: Entities and Groups](#identity-entities-and-groups)
  - [Entities](#entities)
  - [Groups](#groups)
- [Vault Policies](#vault-policies)
  - [Out of the Box Policies](#out-of-the-box-policies)
- [Vault Polices - Capabilities](#vault-polices---capabilities)
  - [Most Used](#most-used)
  - [Special](#special)
  - [Examples](#examples)
- [Customizing the Path](#customizing-the-path)
  - [Using the `*` to Customize the Path](#using-the--to-customize-the-path)
  - [Using the `+` to Customize the Path](#using-the--to-customize-the-path-1)
  - [ACL Templating](#acl-templating)
    - [Template Parameters](#template-parameters)
- [Working with Policies](#working-with-policies)
  - [Testing Policies](#testing-policies)
  - [Administrative Policies](#administrative-policies)
- [Sentinel Policies (Enterprise)](#sentinel-policies-enterprise)
- [Control Groups (Enterprise)](#control-groups-enterprise)
- [Multi-Tenancy with Namespaces (Enterprise)](#multi-tenancy-with-namespaces-enterprise)

## [Identity: Entities and Groups](https://developer.hashicorp.com/vault/tutorials/auth-methods/identity)

Vault supports multiple authentication methods and also allows enabling the same type of authentication method on different mount paths. Each Vault client may have multiple accounts with various identity providers that are enabled on the Vault server.

Vault clients can be mapped as entities and their corresponding accounts with authentication providers can be mapped as aliases. In essence, each entity is made up of zero or more aliases. Identity secrets engine internally maintains the clients who are recognized by Vault.

There are two types of groups `internal` and `external`.
Internal groups are manually created in vault while external groups are automatically created when inferred from an auth method.  

> **NOTE:** OIDC groups are not automatically created.

### Entities

- Vault creates an entity and attaches an alias to it if a corresponding entity doesn't already exist.
- This is done using the Identity secrets engine, which manages internal identities that are recognized by Vault
- An `entity` is a representation of a single person or system used to log into Vault. Each has a unique value. Each entity is made up of zero or more aliases
- An `alias` is a combination of the auth method plus some identification. It is a mapping between an entity and auth method(s)
- An entity can be manually created to map multiple entities for a single user to provide more efficient authorization management
- Any tokens that are created for the entity inherit the capabilities that are granted by alias(es)
- Not everyone creates manual entities as most of the time there's just one auth method enabled. Can be useful if you have multiple accounts that you want to tie together to inherit privileges.
- [Slides 1-14](https://github.com/sarg3nt/vault-training/blob/main/operations-training/07-Configure-Access-Control.pdf)

```bash
# Create a new entity that we will add aliases to
vault write identity/entity name="Julie Smith" \
    policies="it-management" \
    metadata="organization"="HCVOP, Inc" \
    metadata="team"="management"
Key        Value
---        -----
aliases    <nil>
id         d1b44e30-10c7-8c32-9851-3207c0eb8895 # This is the canonical_id we'd use when creating an alias below
name       Julie Smith

# Add GitHub auth as an alias
vault write identity/entity-alias \
    name="jsmith22" \
    canonical_id=<entity_id> \
    mount_accessor=<github_auth_accessor>

# Add LDAP auth as an alias
vault write identity/entity-alias \
    name="jsmith@hcvop.com" \
    canonical_id=<entity_id> \
    mount_accessor=<ldap_auth_accessor>

# Find accessors for existing auths
vault auth list
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
  - Note that the group name must match the group name in your identity provider

## Vault Policies

[Slides 16-40](https://github.com/sarg3nt/vault-training/blob/main/operations-training/07-Configure-Access-Control.pdf))

- Also called ACL Policies or just ACLs
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
vault policy list
default
root

# Look at the default policy
vault policy read default
# Allow tokens to look up their own properties
path "auth/token/lookup-self" {
    capabilities = ["read"]
}

# Allow tokens to renew themselves
path "auth/token/renew-self" {
    capabilities = ["update"]
    . . .

# NOTE: Cannot read root policy even though it exists
vault policy read root
No policy named: root

# Create or update a policy
vault policy write admin-policy /tmp/admin.hcl
Success! Uploaded policy: admin-policy

vault policy write webapp -<< EOF
path "kv/data/apps/*" {
  capabilities = ["read","create","update","delete"]
}
path "kv/metadata/*" {
  capabilities = ["read","create","update","list"]
}
EOF

# Using the API
curl \
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

[Root-Protected Paths](https://learn.hashicorp.com/tutorials/vault/policies#root-protected-api-endpoints)

- Root-Protected Paths
  - Many paths in Vault require a root token or `sudo` capability to use
  - These paths focus on important/critical paths for Vault or plugins
- Examples of root-protected paths:
  - `auth/token/create-orphan` (create an orphan token)
  - `pki/root/sign-self-issued` (sign a self-issued certificate)
  - `sys/rotate` (rotate the encryption key)
  - `sys/seal` (manually seal Vault)
  - `sys/step-down` (force the leader to give up active status)

## Vault Polices - Capabilities

[Slides 41-47](https://github.com/sarg3nt/vault-training/blob/main/operations-training/07-Configure-Access-Control.pdf)

### Most Used

- `create` – create a new entry
- `read` – read credentials, configurations, etc
- `update` – overwrite the existing value of a secret or configuration
- `delete` – delete something
- `list` – view what's there, doesn't allow you to read, good for admins to see a list of what is there but not the secret data itself

> **NOTE:** Write is not a valid capability

### Special

- `sudo` – used for root-protected paths
- `deny` – deny access – always takes precedence over any other capability

### Examples

Requirement:

- Access to generate database credentials at `database/creds/db01`
- Create, Update, Read, and Delete secrets stored at `kv/apps/dev-app01`

Solution:

```bash
path "database/creds/dev-db01" {
  capabilities = ["read"]
}
path "kv/apps/dev-app01" {
  capabilities = ["create", "read", "update", "delete"]
}
```

Requirement:

- Access to read credentials after the path `kv/apps/webapp`
- Deny access to `kv/apps/webapp/super-secret`

Solution:

```bash
path "kv/apps/webapp/*" {
  capabilities = ["read"]
}
path "kv/apps/webapp/super_secret" {
  capabilities = ["deny"]
}
```

> **NOTE:** The above policy would not allow a user to browse `kv/apps/webapp` in the Web UI. We would need to grant `list` to those paths.

## Customizing the Path

[Slides 48-59](https://github.com/sarg3nt/vault-training/blob/main/operations-training/07-Configure-Access-Control.pdf)

### Using the `*` to Customize the Path

- The glob `*` is a wildcard and can only be used at the **end** of a path
- Can be used to signify anything **after** a path or as part of a pattern
- Examples:
  - `secret/apps/application1/*` - allows any path after application1
  - `kv/platform/db-*`  - would match `kv/platform/db-2` but not `kv/platform/db2`

If we wanted to ALSO read from `secret/apps/application1`, the policy would look like this:

```bash
path "secret/apps/application1/*" {
  capabilities = ["read"]
}
path "secret/apps/application1" {
  capabilities = ["read"]
}
```

### Using the `+` to Customize the Path

- The plus `+` supports wildcard matching for a single directory in the path
- Can be used in multiple path segments, i.e., `secret/+/+/db`
- Examples:
  - `secret/+/db` - matches `secret/db2/db` or `secret/app/db`
  - `kv/data/apps/+/webapp` – matches the following:
    - `kv/data/apps/dev/webapp`
    - `kv/data/apps/qa/webapp`
    - `kv/data/apps/prod/webapp`

Example:

```bash
path "secret/+/+/webapp" {
 capabilities = ["read", "list"]
}
path "secret/apps/+/team-*" {
 capabilities = ["create", "read"]
}
```

### ACL Templating

- Use variable replacement in some policy strings with values available to the token
- Define policy paths containing double curly braces: `{{<parameter>}}`

Example:  Creates a section of the key/value v2 secret engine to a specific user

```bash
path "secret/data/{{identity.entity.id}}/*" {
  capabilities = ["create", "update", "read", "delete"]
}
path "secret/metadata/{{identity.entity.id}}/*" {
  capabilities = ["list"]
}
```

If my `entity.id` was `123456` then vault would create a policy for me that allows me to store secrets at `secret/data/123456/*`

#### Template Parameters

| Parameter                                                              | Description                                                             |
| ---------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `identity.entity.id`                                                   | The entity's ID                                                         |
| `identity.entity.name`                                                 | The entity's name                                                       |
| `identity.entity.metadata.<<metadata key>>`                            | Metadata associated with the entity for the given key                   |
| `identity.entity.aliases.<<mount accessor>>.id`                        | Entity alias ID for the given mount                                     |
| `identity.entity.aliases.<<mount accessor>>.name`                      | Entity alias name for the given mount                                   |
| `identity.entity.aliases.<<mount accessor>>.metadata.<<metadata key>>` | Metadata associated with the alias for the given mount and metadata key |
| `identity.groups.ids.<<group id>>.name`                                | The group name for the given group ID                                   |
| `identity.groups.names.<<group name>>.id`                              | The group ID for the given group name                                   |
| `identity.groups.names.<<group id>>.metadata.<<metadata key>>`         | Metadata associated with the group for the given key                    |
| `identity.groups.names.<<group name>>.metadata.<<metadata key>>`       | Metadata associated with the group for the given key                    |

## Working with Policies

[Slides 60-64](https://github.com/sarg3nt/vault-training/blob/main/operations-training/07-Configure-Access-Control.pdf)

### Testing Policies

Test to make sure the policy fulfills the requirements

Example Requirements:

- Clients must be able to request AWS credential granting read access to a S3 bucket
- Read secrets from `secret/apikey/Google`

```bash
# Create a token using the new policy
vault token create -policy="web-app"
# Authenticate with the newly generated token
vault login <token>
# Make sure that the token can read
vault read secret/apikey/Google
# This should fail
vault write secret/apikey/Google key="ABCDE12345"
# Request a new AWS credentials 
vault read aws/creds/s3-readonly 
```

### Administrative Policies

- Permissions for Vault backend functions live at the sys/ path
- Users/admins will need policies that define what they can do within Vault to administer Vault itself
  - Unsealing
  - Changing policies
  - Adding secret backends
  - Configuring database configurations

Example Administrative Policy

```bash
# Configure License
path "sys/license" {
  capabilities = ["read", "list", "create", "update", "delete"]
}
# Initialize Vault
path "sys/init" {
  capabilities = ["read", "update", "create"]
}
# Configure UI in Vault
path "sys/config/ui" {
  capabilities = ["read", "list", "update", "delete", "sudo"]
}
# Allow rekey of unseal keys for Vault
path "sys/rekey/*" {
  capabilities = ["read", "list", "update", "delete"]
}
# Allows rotation of master key
path "sys/rotate" {
  capabilities = ["update", "sudo"]
}
# Allows Vault seal
path "sys/seal" {
  capabilities = ["sudo"]
}
```

> See instructors [github](https://github.com/btkrausen/hashicorp/tree/master/vault/policies) for example policies

## Sentinel Policies (Enterprise)

[Slides 66-80](https://github.com/sarg3nt/vault-training/blob/main/operations-training/07-Configure-Access-Control.pdf)

## Control Groups (Enterprise)

[Slides 82-92](https://github.com/sarg3nt/vault-training/blob/main/operations-training/07-Configure-Access-Control.pdf)

## [Multi-Tenancy with Namespaces (Enterprise)](https://developer.hashicorp.com/vault/docs/commands/namespace)

[Slides 93-109](https://github.com/sarg3nt/vault-training/blob/main/operations-training/07-Configure-Access-Control.pdf)
