# Build Fault-Tolerant Vault Environments

## Configure a Highly Available Cluster

### What Should a Cluster Look Like?

- Ideally, we want something that provides redundancy, failure tolerance, scalability, and a fully replicated architecture
- For Vault Enterprise, you are limited to either Integrated Storage or Consul storage backends
- HashiCorp (and consultants like me) are moving away from Consul as the primary storage backend and using Integrated Storage for everything
- The Vault Operations Professional exam will NOT feature Consul as a configuration or deployment option

### Multi-Node Cluster using Integrated Storage

- Integrated Storage (aka Raft) allows Vault nodes to provide its own replicated storage across the Vault nodes within a cluster
- Define a local path to store replicated data
- All data is replicated among all nodes in the cluster

### How Do I Configure Integrated Storage?

- Initial configuration of Integrated Storage is done in the Vault configuration file
- Multiple ways to join nodes to create a Vault cluster in the configuration file....or you do it manually
- Use `retry_join` stanza to automate the creation of the cluster from participating Vault nodes
- path = the filesystem path where all the Vault data will be stored
- node_id = the identifier for the node in the cluster – cannot be duplicated within a cluster
- retry_join [optional] – automatically join the listed nodes to create a cluster

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

**Manually Configure `retry_join`**  
Each retry_join stanza can include DNS names or IP addresses and the port

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

**Using `auto_join` to discover other Vault nodes using tags**

> NOTE: We should be able to use `auto_join` with vSphere, see the Links section below for docs.

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

**`auto_join` Links:**

- See the [go-discover](https://github.com/hashicorp/go-discover) docs for a list of providers supported by `auto_join` and their uses.
- [vSphere Config Options](https://github.com/hashicorp/go-discover/blob/8b3ddf4/provider/vsphere/vsphere_discover.go#L145-L157)

**Manually join standby nodes to the cluster using the CLI:**

```bash
❯ vault operator raft join https://active_node.example.com:820
```

**Commands:**

```bash
# Show our peers, must be authenticated.
❯ vault operator raft list-peers

Node       Address             State       Voter
----       -------             -----       -----
node-a     10.0.101.22:8201    leader      true
node-b     10.0.101.23:8201    follower    true
node-c     10.0.101.24:8201    follower    true
node-d     10.0.101.25:8201    follower    true
node-e     10.0.101.26:8201    follower    true

# Remove a peer:  Use the name of the node as shown under Node above.
# Make sure to run this command before shutting down a node / deleting it
❯ vault operator raft remove-peer node-e
Peer removed successfully!

# Have a node step down as leader
❯ vault operator step-down
```

See [slides 1-13](operations-training/04-Build-Fault-Tolerant-Vault-Environments.pdf) for more information

## Enable and Configure Disaster Recovery Replication (Enterprise)

- This feature is only available in Vault Enterprise
- See [slides 14-41](operations-training/04-Build-Fault-Tolerant-Vault-Environments.pdf) for more information

## Promote a Secondary Cluster (Enterprise)

- This feature is only available in Vault Enterprise
- See [slides 42-52](operations-training/04-Build-Fault-Tolerant-Vault-Environments.pdf) for more information
