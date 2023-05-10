# [Identity: Entities and Groups](https://developer.hashicorp.com/vault/tutorials/auth-methods/identity)

Vault supports multiple authentication methods and also allows enabling the same type of authentication method on different mount paths. Each Vault client may have multiple accounts with various identity providers that are enabled on the Vault server.

Vault clients can be mapped as entities and their corresponding accounts with authentication providers can be mapped as aliases. In essence, each entity is made up of zero or more aliases. Identity secrets engine internally maintains the clients who are recognized by Vault.

There are two types of groups `internal` and `external`.  
Internal groups are manually created in vault while external groups are automatically created when inferred from an auth method.  

NOTE: OIDC groups are not automatically created.
