<!-- omit from toc -->
# [Vault Agent](https://developer.hashicorp.com/vault/docs/agent)

- [Securely Configure Auto-Auth and Token Sink](#securely-configure-auto-auth-and-token-sink)
  - [Vault Agent – Auto-Auth](#vault-agent--auto-auth)
  - [Vault Agent Configuration File](#vault-agent-configuration-file)
    - [Starting the Vault Agent](#starting-the-vault-agent)
  - [Vault Agent - Sink](#vault-agent---sink)
  - [Auto-Auth Security Concerns](#auto-auth-security-concerns)
  - [Protecting the Token using Response Wrapping](#protecting-the-token-using-response-wrapping)
  - [Response-Wrapping the Token - Comparison](#response-wrapping-the-token---comparison)
- [Vault Templating](#vault-templating)
  - [Consul Template](#consul-template)
  - [Consul Template - Workflow](#consul-template---workflow)
    - [Create a templated file](#create-a-templated-file)
    - [Destination File that application server will read at runtime](#destination-file-that-application-server-will-read-at-runtime)
  - [Vault Agent Templating](#vault-agent-templating)
    - [Template Configuration](#template-configuration)

The Vault Agent is a client daemon that runs alongside an application to enable legacy applications to interact and consume secrets.

Vault Agent provides several different features:

- Automatic authentication including renewal
- Secure delivery/storage of tokens (response wrapping)
- Local caching of secrets
- Templating - Retrieve secrets from vault and create a file based on a template

## Securely Configure Auto-Auth and Token Sink

[Slides 1-13](operations-training/08-Configure-Vault-Agent.pdf)

### Vault Agent – Auto-Auth

- The Vault Agent uses a pre-defined auth method to authenticate to Vault and obtain a token
- The token is stored in the "sink", which is essentially just a flat file on a local file system that contains the Vault token
- The application can read this token and invoke the Vault API directly
- This strategy allows the Vault Agent to manage the token and guarantee a valid token is always available to the application

Vault Agent supports many types of auth methods to authenticate and obtain a token.

Auth methods are generally the methods you'd associate with "machine-oriented" auth methods:

- AliCloud
- AppRole
- AWS
- Azure
- Certificate
- CloudFoundry
- GCP
- JWT
- Kerberos
- Kubernetes

### Vault Agent Configuration File

```bash
# agent.hcl
auto_auth {
  # Auto-Auth Configuration (AppRole)
  method "approle" {
    mount_path = "auth/approle"
    config = {
      role_id_file_path = "<path-to-file>"
      secret_id_file_path = "<path-to-file>"
      # By default vault will delete the secret_id file above after reading it.
      # Vault assumes you are injecting the secret id file via a deployment process such as Jenkins or via kubernetes
      # Use this parameter to disable this functionality (not recommended)
      # ? how to handle a stand alone server that restarts ?
      remove_secret_id_file_after_reading = false
    }
  }

  # Sink Configuration
  sink "file" {
    config = {
      path = "/etc/vault.d/token.txt"
    }
  }
}

# Vault
vault {
  address = "http://<cluster_IP>:8200"
}
```

#### Starting the Vault Agent

```bash
vault agent -config=/etc/vault.d/agent.hcl
```

### Vault Agent - Sink

As of today, file is the only supported method of storing the auto-auth token

Common configuration parameters include:

- `type` –what type of sink to use (again, only `file` is available)
- `path` –location of the file
- `mode` –change the file permissions of the sink file (default is 640)
- `wrap_ttl` = retrieve the token using response wrapping

### Auto-Auth Security Concerns

- A man in the middle attack between the app and Vault when vault is returning the token
- The app server being compromised and the bad actor gaining access to the sink file.

### Protecting the Token using Response Wrapping

To help secure tokens when using Auto-Auth, you can have Vault response wrap the token when the Vault Agent authenticates

- Response wrapped by the auth method
- Response wrapped by the token sink

The placement of the `wrap_ttl` in the Vault Agent configuration file determines where the response wrapping happens.

### Response-Wrapping the Token - Comparison

- Response Wrapped by the Auth Method
  - Pros:
    - Prevents man-in-the-middle attacks (MITM)
    - More secure
  - Cons:
    - Vault agent cannot renew the token
- Response Wrapped by the Sink
  - Pros:
    - Allows the Vault agent to renew the token and re-authenticate when the token expires
  - Cons:
    - The token gets wrapped after it is retrieved from Vault. Therefore, it is vulnerable to MITM attacks

Response Wrapped by the Auth Method:

```bash
# agent.hcl
pid_file = "/home/vault/pidfile"

auto_auth {
  method "kubernetes" {
    wrap_ttl = "5m" # wrap_ttl goes here for wrapping at the auth method.
    mount_path = "auth/kubernetes"
    config = {
      role = "example"
    }
  }
  sink "file" {
    config = {
      path = "/etc/vault/token"
    }
  }

vault {
  address = "http://<cluster_IP>:8200"
}
```

Response Wrapped by the Sink

```bash
# agent.hcl
pid_file = "/home/vault/pidfile"

auto_auth {
  method "kubernetes" {
    mount_path = "auth/kubernetes"
    config = {
      role = "example"
    }
  }
  sink "file" {
    wrap_ttl = "5m" # wrap_ttl goes here for wrapping at the sink method
    config = {
      path = "/etc/vault/token"
    }
  }
}

vault {
  address = "http://<cluster_IP>:8200"
}
```

## [Vault Templating](https://developer.hashicorp.com/vault/docs/agent/template)

[Slides 15-24](operations-training/08-Configure-Vault-Agent.pdf)

- Templating is typically for legacy applications that cannot reach out to the Vault API and get secrets for itself.
- Templating should not be used for modern / internal applications we have control over.
- There are Vault integrations with most programming languages.
  - [.Net](https://developer.hashicorp.com/vault/tutorials/app-integration/dotnet-httpclient)
  - [Go](https://pkg.go.dev/github.com/hashicorp/vault)

### Consul Template

- A standalone application that renders data from Consul or Vault onto the target file system
  - [Consul Template Github](https://github.com/hashicorp/consul-template)
  - Despite its name, Consul Template does not require a Consul cluster to operate
- Consul Template retrieves secrets from Vault
  - Manages the acquisition and renewal lifecycle
  - Requires a valid Vault token to operate

### Consul Template - Workflow

1. Create a templated file that specifies the path and data you want
2. Execute Consul Template. Consul Template retrieves data and renders to destination file
3. The application reads the new file at runtime that includes secrets retrieved from Vault

#### Create a templated file

```yaml
# /etc/vault/web.tmpl
production:
  adapter: postgresql
  encoding: unicode
  database: orders
  host: postgres.hcvop.com # The path in Vault to read secrets from
  {{ with secret "database/creds/readonly" }}
  username: "{{ .Data.username }}" # The data to be retrieved and consumed from the Vault response
  password: "{{ .Data.password }}"
  {{ end }}
```

#### Destination File that application server will read at runtime

```yaml
# /etc/webapp/config.yml
production:
  adapter: postgresql
  encoding: unicode
  database: orders
  host: postgres.hcvop.com
  username: "v-vault-readonly-fm3dfm20sm2s"
  password: "fjk39fkj49fks02k_3ks02mdz1s1"
```

### Vault Agent Templating

- To further extend the functionality of the Vault Agent, a subset of the Consul-Template functionality is directly embedded into the Vault Agent
  - No need to install the Consul-Template binary on the application server
- Vault secrets can be rendered to destination file(s) using the Consul-Template markup language
  - Uses the client token acquired by the auto-auth configuration

#### Template Configuration

```bash
auto_auth {
  method "approle" {
    mount_path = "auth/approle"
...
sink "file" {
  config = {
    path = "/etc/vault.d/token.txt"
...

# Global Template Configurations (affects all templates)
template_config {
  exit_on_retry_failure = true 
  static_secret_render_interval = "10m"
}

# Template Configuration
template {
  source = "/etc/vault/web.tmpl"
  destination = "/etc/webapp/config.yml"
}
```
