<!-- omit from toc -->
# Vault Install, Setup and Unseal

- [Links](#links)
- [Install](#install)
  - [Install Ubuntu via Apt](#install-ubuntu-via-apt)
  - [Install Linux Manually](#install-linux-manually)
  - [Load Balancer](#load-balancer)
- [Configuration](#configuration)
- [Integrated Storage](#integrated-storage)
  - [Multi-Node Cluster using Integrated Storage](#multi-node-cluster-using-integrated-storage)
  - [Configure Integrated Storage](#configure-integrated-storage)
    - [Manually Configure `retry_join`](#manually-configure-retry_join)
    - [Using `auto_join` to discover other Vault nodes using tags](#using-auto_join-to-discover-other-vault-nodes-using-tags)
    - [`auto_join` Links](#auto_join-links)
    - [Manually join standby nodes to the cluster using the CLI](#manually-join-standby-nodes-to-the-cluster-using-the-cli)
  - [Commands](#commands)
  - [Notes On Automating Snapshots](#notes-on-automating-snapshots)
- [Init \& Unseal](#init--unseal)
  - [Transit Auto Unseal](#transit-auto-unseal)
- [Secure Vault Initialization with PGP](#secure-vault-initialization-with-pgp)

## Links

- [Starting Vault in Dev Mode](https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-dev-server)
- [Deploy Vault](https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-deploy) tutorial (single node)
- [Vault HA Cluster with Integrated Storage](https://developer.hashicorp.com/vault/tutorials/raft/raft-storage)
- [Vault with Integrated Storage Reference Architecture](https://developer.hashicorp.com/vault/tutorials/day-one-raft/raft-reference-architecture)

```bash
# Start vault in dev mode
vault server -dev
```

## [Install](https://developer.hashicorp.com/vault/downloads)

### Install Ubuntu via Apt

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vault
```

### Install Linux Manually

[Latest releases](https://releases.hashicorp.com/vault/)

```bash
#!/bin/bash
# Following is from https://discuss.hashicorp.com/t/is-there-a-programmatic-way-to-look-up-the-latest-version/15175/8 modified by Dave Sargent
# Get the latest vault version from github
VAULT_VERSION=$(curl -fsS https://api.github.com/repos/hashicorp/vault/tags \
    | jq -re '.[].name' \
    | grep -v 'beta\|rc' \
    | sed 's/^v\(.*\)$/\1/g' \
    | sort -Vr \
    | head -1)

wget -q --show-progress "https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip" -O /tmp/vault.zip
unzip /tmp/vault.zip -d /tmp
# Note, vault is already executable.
sudo mv /tmp/vault /usr/local/bin
rm /tmp/vault.zip
```

Add a service file, use [this](https://github.com/btkrausen/hashicorp/blob/master/vault/config_files/vault.service) as a starting point

Create the service

### Load Balancer

- Vault provides health information via the `/sys/health` endpoint
- Load Balancers can target specific return codes to determine an Active node vs. a Performance Standby node
- The default status codes include:
  - `200` – initialized, unsealed, and active node
  - `429` – unsealed but standby node
  - `472` – DR replication secondary and active node
  - `473` – Performance Standby
  - `501` – Not Initialized
  - `503` – Sealed node

## [Configuration](https://developer.hashicorp.com/vault/docs/configuration#storage)

Add a configuration file to `/etc/vault.d/vault.hcl`  
Use [this](https://github.com/btkrausen/hashicorp/blob/master/vault/config_files/vault.hcl) file as a starting point.

## [Integrated Storage](https://developer.hashicorp.com/vault/docs/concepts/integrated-storage)

- [Slides 213-235](operations-training/01-Create-a-working-Vault-server-configuration-given-a-scenario.pdf))
- [Vault HA Cluster with Integrated Storage](https://developer.hashicorp.com/vault/tutorials/raft/raft-storage)

```bash
# Storage stanza of the vault.hcl config file for using integrated storage in a cluster
storage "raft" {
  path = "/opt/vault/data" 
  node_id = "vault-node-a.hcvop.com"  # Name of the local node
  # Adding the retry_join for each other node in the cluster will cause the cluster to auto build
  retry_join {
    leader_api_addr = https://vault-node-b.hcvop.com:8200
    leader_ca_cert_file = "/opt/vault.d/ca.pem" # cert files should only be necessary if servers don't trust the certs.
    leader_client_cert_file = "/opt/vault.d/cert.pem"
    leader_client_key_file = "/opt/vault.d/pri.key" 
  }
  retry_join {
    leader_api_addr = https://vault-node-c.hcvop.com:8200
    leader_ca_cert_file = "/opt/vault.d/ca.pem"
    leader_client_cert_file = "/opt/vault.d/cert.pem"
    leader_client_key_file = "/opt/vault.d/pri.key"  
  }
  performance_multiplier = 1 # Time it takes vault to detect and fix leader failure and election
}
```

### Multi-Node Cluster using Integrated Storage

- Integrated Storage (aka Raft) allows Vault nodes to provide its own replicated storage across the Vault nodes within a cluster
- Define a local path to store replicated data
- All data is replicated among all nodes in the cluster

### Configure Integrated Storage

- Initial configuration of Integrated Storage is done in the Vault configuration file
- Multiple ways to join nodes to create a Vault cluster in the configuration file....or you do it manually
- Use `retry_join` stanza to automate the creation of the cluster from participating Vault nodes. This should work with VMWare labels
- `path` = the filesystem path where all the Vault data will be stored
- `node_id` = the identifier for the node in the cluster – cannot be duplicated within a cluster
- `retry_join` [optional] – automatically join the listed nodes to create a cluster

```bash
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "node-a.hcvop.com"
  retry_join {
    auto_join = "provider=aws region=us-east-1 tag_key=vault tag_value=east-1"
  }
}
listener "tcp" {
  address = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_disable = 0
}
seal "awskms" {
  region = "us-east-1"
  kms_key_id = "12345678-abcd-1234-abcd-123456789101",
}
api_addr = "https://vault.hcvop.com:8200"
cluster_addr = " https://node-a.hcvop.com:8201"
cluster_name = "vault-prod-us-east-1"
ui = true
log_level = "INFO"
```

#### Manually Configure `retry_join`

Each `retry_join` stanza can include DNS names or IP addresses and the port

```bash
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "node-a.hcvop.com"
  retry_join {
    leader_api_addr = "https://node-b.hcvop.com:8200"
  }
  retry_join {
    leader_api_addr = "https://node-c.hcvop.com:8200"
  }
  retry_join {
    leader_api_addr = "https://node-d.hcvop.com:8200"
  }
  retry_join {
    leader_api_addr = "https://node-e.hcvop.com:8200"
  }
}
```

#### Using `auto_join` to discover other Vault nodes using tags

> **NOTE:** We should be able to use `auto_join` with vSphere, see the Links section below for docs.

```bash
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "node-a.hcvop.com"
  retry_join {
    # NOTE: Below is untested
    auto_join = "provider=vsphere host=lanier.ad.selinc.com tag_name=vault_keymaster_cluster category_name=vault user=vmwareuser password=vmwarepassword"
  }
}
```

#### `auto_join` Links

- See the [go-discover](https://github.com/hashicorp/go-discover) docs for a list of providers supported by `auto_join` and their uses.
- [vSphere Config Options](https://github.com/hashicorp/go-discover/blob/8b3ddf4/provider/vsphere/vsphere_discover.go#L145-L157)

#### Manually join standby nodes to the cluster using the CLI

```bash
vault operator raft join https://active_node.example.com:820
```

### Commands

```bash
# Show our peers, must be authenticated.
vault operator raft list-peers

Node       Address             State       Voter
----       -------             -----       -----
node-a     10.0.101.22:8201    leader      true
node-b     10.0.101.23:8201    follower    true
node-c     10.0.101.24:8201    follower    true
node-d     10.0.101.25:8201    follower    true
node-e     10.0.101.26:8201    follower    true

# Have a node step down as leader
vault operator step-down

# Remove a peer:  Use the name of the node as shown under Node above.
# Make sure to run this command before shutting down a node / deleting it
vault operator raft remove-peer node-e
Peer removed successfully!

# Other Commands
vault operator raft join https://vault-node-b.hcvop.com:8200
vault operator raft leave node-b

vault operator raft snapshot save <file_name>
vault operator raft snapshot restore <file_name>
```

[Slides 1-13](operations-training/04-Build-Fault-Tolerant-Vault-Environments.pdf))

### Notes On Automating Snapshots

A [Github repo](https://github.com/adfinis/vault-raft-backup-agent) that has a decent looking solution for automating snapshots.

- According to [this](https://developer.hashicorp.com/vault/docs/enterprise/automated-integrated-storage-snapshots) only the active node should be doing snapshot so we need to modify the above to make sure the current node is the active node then run the snap
- Script to also manage how many snaps to retain
- Destination could probably be an NFS share or we could use the S3 tool mentioned in the above repo and move snaps to Flashblade. Check with Pickering on what he thinks would be best.
- Script should check local storage for available space and throw alert to alerting system if space is to small.
- Throw alert to alerting system if snapshot fails or there is not auth key
- [Vault Integrated Storage Autopilot](https://developer.hashicorp.com/vault/tutorials/raft/raft-autopilot)

## Init & Unseal

- [Slides 185-198](operations-training/01-Create-a-working-Vault-server-configuration-given-a-scenario.pdf)) for auto unseal setup
- [Slides 199-212](operations-training/01-Create-a-working-Vault-server-configuration-given-a-scenario.pdf)#120) for unseal migrations

```bash
# Init vault
vault operator init

# Unseal vault, will need to do this three times in standard prod mode if auto-unseal is not enabled.
vault operator unseal
```

### [Transit Auto Unseal](https://developer.hashicorp.com/vault/tutorials/auto-unseal/autounseal-transit)

```bash
### On the transit server (keymaster), create the unseal key and policy
# Enable transit secrets engine if it's not already enabled
vault secrets enable transit
# Create the unseal key
vault write -f transit/keys/autounseal

# Create the policy
vault policy write unseal-policy /tmp/unseal_policy.hcl

# unseal_policy.hcl
# NOTE: The tutorial linked above only shows the first two, but the labs from training had the key policies as well.
path "transit/encrypt/autounseal" {
  capabilities = ["update"]
}
path "transit/decrypt/autounseal" {
  capabilities = ["update"]
}
path "transit/keys/autounseal" {
  capabilities = ["update", "create", "read"]
}
path "transit/keys" {
  capabilities = ["list"]
}

# Create a token using the policy
vault token create -policy=unseal-policy -ttl=24h
Key                  Value
---                  -----
token                hvs.CAESILr2t_yhROKFcXkYebQV_qkl61yq77G4hgVqxbqP-4o8Gh4KHGh2cy5LRHRJd254NmFzUkExS3BXQjdxUFUzcFk
token_accessor       8K4FMqpIsrOOr6QtSw18B35I
token_duration       24h
token_renewable      true
token_policies       ["default" "unseal-policy"]
identity_policies    []
policies             ["default" "unseal-policy"]

# Add the seal clause to the vault config of the server you want to auto unseal
seal "transit" {
  address = "http://<transit-server>:8200"
  token = "hvs.CAESILr2t_yhROKFcXkYebQV_qkl61yq77G4hgVqxbqP-4o8Gh4KHGh2cy5LRHRJd254NmFzUkExS3BXQjdxUFUzcFk"
  mount_path = "transit/"
  key_name = "autounseal"
  #tls_skip_verify = "true"
}
# and restart the vault service.

# Initialize the vault instance
vault operator init
# The server should now be initialized and unsealed. Record and distribute the recovery keys
```

## Secure Vault Initialization with PGP

When initializing vault we should use PGP keys provided by the key guardians to encrypt the vault key shards.

```bash
# For a cluster that will not use auto unseal
vault operator init \
  -key-shares=5 \ # default, must match number of keys providing
  -key-threshold=3 \ # default
  -pgp-keys="/opt/bob.pub,/opt/steve.pub,/opt/fred.pub,/opt/katie.pub,/opt/nancy.pub"

# For a cluster that will use auto unseal.
vault operator init \
  -recovery-shares=5 \ # default, must match number of keys providing
  -recovery-threshold=3 \ # default
  -recovery-pgp-keys="/opt/bob.pub,/opt/steve.pub,/opt/fred.pub,/opt/katie.pub,/opt/nancy.pub"
```

We can also protect the root token with a pgp key

```bash
vault operator init \
  -recovery-shares=5 \ # default, must match number of keys providing
  -recovery-threshold=3 \ # default
  -recovery-pgp-keys="/opt/bob.pub,/opt/steve.pub,/opt/fred.pub,/opt/katie.pub,/opt/nancy.pub" \
  -root-token-pgp-key="/opt/jason.pub"
```

To decrypt a key

```bash
echo "<provided key>" | base64 -d | gpg -dq
```

[Slides 289-304](operations-training/01-Create-a-working-Vault-server-configuration-given-a-scenario.pdf))
