<!-- cSpell:ignore -->
# Monitoring a Vault Environment

## [Vault Telemetry](https://developer.hashicorp.com/vault/docs/internals/telemetry)

- The collection of various runtime metrics about the performance of different components of the Vault environment
- Can be used for debugging but it can also be used for performance monitoring and trending
- Metrics are aggregated every 10 seconds and retained for one minute
- The telemetry information is sent to a local or remote agent which generally aggregates this information to an aggregation solution, such as DataDog or Prometheus, for example
- Supports the following providers:
  - statsite
  - statsd
  - circonus
  - dogstatsd
  - [prometheus](https://developer.hashicorp.com/vault/tutorials/monitoring/monitor-telemetry-grafana-prometheus)
  - stackdriver

### Telemetry Links

- [Telemetry Internals](https://developer.hashicorp.com/vault/docs/internals/telemetry)
- [telemetry Stanza](https://developer.hashicorp.com/vault/docs/configuration/telemetry)
- [Telemetry Metrics Reference](https://developer.hashicorp.com/vault/tutorials/monitoring/telemetry-metrics-reference)
- [Monitor Telemetry with Prometheus & Grafana](https://developer.hashicorp.com/vault/tutorials/monitoring/monitor-telemetry-grafana-prometheus)
- See [slides 1-7](operations-training/02-Monitor-a-Vault-Environment.pdf) for more information.

```bash
# Set in vault config file
telemetry {
  disable_hostname = true
  prometheus_retention_time = "12h"
}
```

## [Audit Logs](https://developer.hashicorp.com/vault/docs/audit)

- Keep a detailed log of all authenticated requests and responses to Vault
- Audit log is formatted using JSON
- Sensitive information is hashed with a salt using HMAC-SHA256 to ensure secrets and tokens are never in plain text
- Log files should be protected as a user with permission can still check the value of those secrets via the /sts/audit-hash API and compare to the log file

### What Audit Devices Does Vault Support

- File
  - Most used
  - Writes to a file – appends logs to the file
  - Does not assist with log rotation
  - Use fluentd or similar tool to send to collector
- Syslog
  - Writes audit logs to a syslog
  - Sends to a local agent only
- Socket
  - Writes to a tcp, udp, or unix socket
  - TCP should be used where strong guarantees are required

### Important Info about Audit Devices

> NOTE: No audit devices are enabled by default

- Can and should have more than one audit device enabled
- If there are any audit devices enabled, Vault requires that it can write to the log before completing the client request.
- Prioritizes safety over availability
- If Vault cannot write to a persistent log, it will stop responding to client requests – which means Vault is down!

> IMPORTANT: If an audit devices is enabled, Vault requires at least one audit device to write the log before completing the Vault request.  A full volume will cause vault to stop responding.

### Audit Links

- [Audit Devices](https://developer.hashicorp.com/vault/docs/audit)
- [Vault Audit Log Details](https://support.hashicorp.com/hc/en-us/articles/360000995548-Audit-and-Operational-Log-Details)
- See [slides 8-16](operations-training/02-Monitor-a-Vault-Environment.pdf) for more information.

```bash
# Enable file audit device at default path
❯ vault audit enable file file_path="/var/log/vault_audit.log"
Success! Enabled the file audit device at: file/

# Enable file audit device at custom path of "logs"
# Enable at a custom path inside of vault
# Add -local to disable replication
❯ vault audit enable -path=logs file \
  file_path="/var/log/audit.log"
Success! Enabled the file audit device at: logs/

# View audit devices enabled on the cluster
❯ vault audit list
Path    Type    Description
---- ---- -----------
file/   file    n/a
syslog/ syslog  n/a

❯ vault audit list --detailed
Path       Type      Description    Replication    Options
----       ----      -----------    -----------    -------
syslog/    syslog    n/a            replicated     n/a

# Disable an Audit Device
❯ vault audit disable syslog/
Success! Disabled audit device (if it was enabled) at: syslog/

> cat /var/log/audit.log | jq

{
  "time": "2023-05-08T18:57:41.561900902Z",
  "type": "request",
  "auth": {
    "client_token": "hmac-sha256:e7bd02262daf2763188bd3dc6c8be322c8a62f39ae2462bd0b80ba1efab2761f",
    "accessor": "hmac-sha256:4ff8d448f14c5bf379b4ce81b7bc48aedaadc2ee929272c66ef43fb92cca1195",
    "display_name": "root",
    "policies": [
      "root"
    ],
    "token_policies": [
      "root"
      ...

# Required Permissions for interacting with the file audit device at the default path of file/
path "sys/audit/file" {
  capabilities = ["read","create","list","update","delete","sudo"]
}
```

## Operation Logs

- During startup, Vault will log configuration information to the log, such as listeners & ports, logging level, storage backend, Vault version, and much more....
- Once started, Vault will continue to log entries which are invaluable for troubleshooting
- The log level can be configured in multiple places in Vault, and include levels 
  - err
  - warn
  - info (default)
  - debug
  - trace

### Specifying the Log Level

1. Use the CLI flag -log_levelwhen starting the Vault service
  `vault server –config=/opt/vault/vault.hcl –log-level=debug`
2. Set the environment variable `VAULT_LOG_LEVEL`  
  Change takes effect after Vault server is restarted  
  `export VAULT_LOG_LEVEL=trace`
3. Set the `log_levelconfiguration` parameter in the Vault configuration file  
  Change takes effect after Vault server is restarted  
  `log_level=warn`

### Logging Links

- See [slides 17-23](operations-training/02-Monitor-a-Vault-Environment.pdf) for more information.

```bash
# Can view logs using journalctl
> journalctl –b -–no-pager –u vault
# shift+g to go to bottom of logs

```