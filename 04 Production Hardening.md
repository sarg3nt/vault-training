<!-- omit from toc -->
# [Production Hardening](https://developer.hashicorp.com/vault/tutorials/operations/production-hardening)

[Slides 144-182](https://github.com/sarg3nt/vault-training/blob/main/operations-training/01-Create-a-working-Vault-server-configuration-given-a-scenario.pdf)

For using Raft/Consul Autopilot, see: [Consul Autopilot](https://developer.hashicorp.com/consul/tutorials/datacenter-operations/autopilot-datacenter-operations) OR [Vault Integrated Storage Autopilot](https://developer.hashicorp.com/vault/tutorials/raft/raft-autopilot)

There are many best practices for a production hardened deployment of Vault.

Practice defense in depth and follow the Vault security model

> **NOTE:** There are specific recommendations for running Vault in Kubernetes that are not included on this page.  
See [09 Vault Security Model](./09%20Vault%20Security%20Model.md)

- [General Recommendations](#general-recommendations)
  - [Deployment Model](#deployment-model)
  - [Limit Access to Vault Nodes](#limit-access-to-vault-nodes)
  - [Limit Services Running on Vault Nodes](#limit-services-running-on-vault-nodes)
  - [Permit Only Required Ports on Firewall](#permit-only-required-ports-on-firewall)
  - [Immutable Upgrades](#immutable-upgrades)
- [Operating System](#operating-system)
  - [Run Vault as an Unprivileged User](#run-vault-as-an-unprivileged-user)
  - [Secure Files and Directories](#secure-files-and-directories)
  - [Protect the Storage Backend](#protect-the-storage-backend)
  - [Disable Shell History](#disable-shell-history)
  - [Configure SELinux/AppArmor](#configure-selinuxapparmor)
  - [Turn Off Core Dumps](#turn-off-core-dumps)
  - [Protect and Audit the vault.service File](#protect-and-audit-the-vaultservice-file)
  - [Patch the Operating System Frequently](#patch-the-operating-system-frequently)
  - [Disable Swap](#disable-swap)
- [Vault-Specific Configurations](#vault-specific-configurations)
  - [Secure Vault with TLS](#secure-vault-with-tls)
  - [Secure Consul](#secure-consul)
  - [Enable Auditing](#enable-auditing)
  - [Say No to Cleartext Credentials](#say-no-to-cleartext-credentials)
  - [Upgrade Vault Frequently](#upgrade-vault-frequently)
  - [Stop Using Root Tokens, Seriously](#stop-using-root-tokens-seriously)
  - [Verify the Integrity of the Vault Binary](#verify-the-integrity-of-the-vault-binary)
  - [Disable the UI – if Not in Use](#disable-the-ui--if-not-in-use)
  - [Encrypt the Gossip Protocol (Consul)](#encrypt-the-gossip-protocol-consul)
  - [Secure the Unseal/Recovery Keys](#secure-the-unsealrecovery-keys)
  - [Minimize the TTLs for Leases and Tokens](#minimize-the-ttls-for-leases-and-tokens)
  - [Follow the Principle of Least Privilege](#follow-the-principle-of-least-privilege)
  - [Perform Regular Backups](#perform-regular-backups)
  - [Integrate with Existing Identity Providers](#integrate-with-existing-identity-providers)


## General Recommendations

### Deployment Model

- The fewer shared resources, the better
- Think "single tenancy" where possible
- Secure deployments: Hardware > VMs > Containers
- Ultimately comes down to protecting memory contents
- Many customers will still use virtualization (VMware/Cloud) or containerization (Docker/K8s) but will deploy to dedicated clusters

### Limit Access to Vault Nodes

- Reduce or eliminate access to Vault nodes
- Includes SSH/RDP and through platform-based access (i.e., AWS SSM, kubectl, etc)
- Instead, access Vault via API or CLI from your workstation or jump box
- If you REALLY need access, use HashiCorp Boundary to limit/control access

### Limit Services Running on Vault Nodes

- Vault nodes should be dedicated to Vault services
- You should not have other services contending for resources
- More services = more firewall requirements
- Don't forget: Encryption keys are stored in memory
- Exception to this rule may include:
- Telemetry agent
- Log file agent (Splunk, SumoLogic, DataDog)

### Permit Only Required Ports on Firewall

- Vault and Consul use dedicated ports for communication
- Permit only the required ports to reduce attack surface
- Many enterprise Vault deployments don’t even allow SSH or UI ports
- Default ports include:
  - Vault: 8200, 8201
  - Consul: 8500, 8201, 8301
- More ports will be needed for our implementation

### Immutable Upgrades

- Immutable upgrades guarantee a known state because you know the result of your automation configurations
- Easy to bring new nodes online, destroy the old nodes
- Consul and Raft can use AutoPilot to assist with upgrades
- Care must be taken when using Raft because you need to ensure replication has been completed to newly added nodes

## Operating System

### Run Vault as an Unprivileged User

- Never run Vault as Root
- Running as root can expose Vault’s sensitive data
- Limit access to configuration files and folders to the Vault user
- Normally, I create a user named “vault”
- Don’t forget that the Vault user will need access to write local files, such as the database, audit logs, and snapshots
  `chmod –R vault:vault /opt/vault/data`

### Secure Files and Directories

- Protect and audit critical Vault directories and files, including directory for snapshots
- Ensure unauthorized changes can’t be made
- Includes binaries, config files, plugins files and directory, service configurations, audit device files and directory, etc.

``` bash
# Set permissions on Vault folder
chmod 740 –R /etc/vault.d
```

### Protect the Storage Backend

- Vault writes all configuration and data to the storage backend
- No storage backend = No Vault!
- Consul Storage Backend:
  - Use Consul ACLs when running on Consul
  - Limit access to any Consul node
  - Enable verify_server_hostname in config file

### Disable Shell History

- Disabling history prevents retrieval of commands
- Possible to discover credentials/tokens in history
- Can also disable just 'vault' command in history

```bash
history
1365  vault login hvs.RTMd9YZ5Np9WGjvTfARaqffQ
1366  vault policy list
1367  vault list sys/policies/acl
1368  vault secrets list
1369  vault list pki/roles

# Disable command history system wide
echo 'set +o history' >> /etc/profile
```

### Configure SELinux/AppArmor

- Don’t disable to make install/management easier
- Provides additional layers of protection for the OS
- Adhere to CIS or DISA to improve posture of the host OS

[Hardening HashiCorp Vault with SELinux](https://hashicorp.com/blog/hardening-hashicorp-vault-with-selinux)

### Turn Off Core Dumps

- Core dumps could reveal encryption keys
- Different process to disable depending on the OS

### Protect and Audit the vault.service File

- Make sure you know if this file is modified or replaced
- An attacker could point to compromised binaries to leak data
- This assumes you are running systemd, but any "service" file should be monitored and secured

### Patch the Operating System Frequently

- Make sure to patch the OS frequently
- Follow your standards or be more stringent for Vault
- Options include Satellite, SpaceWalk, or other solution
- If you use an immutable architecture, then replace the nodes often with known good "patched" images. Use Packer to help simplify this workflow.

### Disable Swap

- Vault stores sensitive data in-memory, unencrypted
- That data should never be written to disk
- Disabling swap provides an extra layer of protection
- Different process for different operating systems but again, you don't have to do this on the exam
- Example is enabling `mlock` to prevent memory swap

## Vault-Specific Configurations

### Secure Vault with TLS

- Vault contains sensitive data
- Communications should never occur without TLS in place
- Load Balancers used with Vault can terminate TLS or instead use pass through to the Vault nodes.
- Verify tls_disable configuration does not equal true or 1
- Default is false (not disabled)

### Secure Consul

- Consul contains your sensitive data
- Communications should never occur without TLS in place
- Issue a certificate from a trusted CA
- Enable Consul ACLs
- Configure gossip encryption (use `consul keygen`)
- In short, follow the Consul Security Model

### Enable Auditing

- Use multiple Audit Devices to log all interactions
- Send that data to a collection server
- Archive log data based upon security policies
- Create alerts based on certain actions

### Say No to Cleartext Credentials

- Don’t put credentials in configuration files
- Use Environment Variables, where supported
- Use cloud-integrated services, such as AWS IAM Roles or Azure Managed Service Identities

### Upgrade Vault Frequently

- New updates frequently include security fixes
- New cipher suites can be enabled or supported
- New functionality enabled

### Stop Using Root Tokens, Seriously

- Root tokens have unrestricted access to Vault with no TTL
- Not bound by ACL policies, ERPs, or RGPs
- Get rid of the initial root token after initial setup
  `vault token revoke hvs.xxxxxxxxx`

### Verify the Integrity of the Vault Binary

- Always get Vault binaries directly from HashiCorp
- Use the HashiCorp checksum to validate
- Modified version of Vault binary could leak data

### Disable the UI – if Not in Use

- Vault UI is disabled by default
- Configured in the Vault configuration file
- Do the same for Consul UI, as well

### Encrypt the Gossip Protocol (Consul)

- TLS only secures the interfaces, not Consul gossip traffic
- Use the –encrypt flag in the Consul configuration file
- Uses a 32-byte key – can use `consul keygen` to generate

### Secure the Unseal/Recovery Keys

- Initialize Vault using PGP keys, such as keybase.io
- Distribute to multiple team members, no single person should have all the keys
- Don’t store the keys in Vault itself

### Minimize the TTLs for Leases and Tokens

- Use the smallest TTL possible for tokens
- Define Max TTLs to prevent renewals beyond reasonable time frame
- Minimizing TTL also helps reduce burden on the storage backend

### Follow the Principle of Least Privilege

- Only give tokens access to paths required for business function
- Separate policies for applications and users`
- Limit use of * and + in policies, where possible
- Templated policies can help with policy creation and maintenance 

### Perform Regular Backups

- Backup configuration files and directories
- Automate Vault backup using snapshots or equivalent depending on the storage backend
- Regularly test backups to ensure functionality
  `vault operator raft snapshot save monday.snap`
  `vault write sys/storage/raft/snapshot-auto/config/daily` (Enterprise)

### Integrate with Existing Identity Providers

- Use your existing IdP to provide access to users
- If/when users leave, they immediately lose access to Vault
- The fewer places a user has credentials, the better
- Using locally defined credentials is an administrative burden

