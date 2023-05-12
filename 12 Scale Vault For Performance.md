<!-- cSpell:ignore -->
# Scale Vault For Performance

## Use Batch Tokens

- Batch tokens are encrypted binary large objects (blobs)
- Designed to be lightweight & scalable
- They are NOT persisted to storage, but they are not fully-featured
- Ideal for high-volume operations
- Can be used for DR Replication cluster promotion as well
- Includes information such as policy, TTL, and other attributes
- Batch tokens are encrypted using the barrier key, which is why they can be used across all clusters within the replica set

### Compare Batch Tokens vs. Service Tokens

> NOTE: Know these differences very well for the exam

|                                                     | Service       | Batch                        |
| --------------------------------------------------- | ------------- | ---------------------------- |
| Can be root tokens                                  | Yes           | No                           |
| Can create child tokens                             | Yes           | No                           |
| Renewable                                           | Yes           | No                           |
| Listable                                            | Yes           | No                           |
| Manually Revocable                                  | Yes           | No                           |
| Can be periodic                                     | Yes           | No                           |
| Can have explicit Max TTL                           | Yes           | No (always uses a fixed TTL) |
| Has accessors                                       | Yes           | No                           |
| Has Cubbyhole                                       | Yes           | No                           |
| Revoked with parent (if not orphan)                 | Yes           | Stops Working                |
| Dynamic secrets lease assignment                    | Self          | Parent (if not orphan)       |
| Can be used across Performance Replication clusters | No            | Yes (if orphan)              |
| Creation scales with performance standby node count | No            | Yes                          |
| Cost                                                | Heavyweight * | Lightweight **               |

\* Multiple storage writes per token creation  
\** No storage cost for token creation, in memory only

> NOTE: Batch tokens must have `-orphan=true` set to be replicated to performance clusters

```bash
# Create a performance replicable batch token
vault token create -type=batch -orphan=true -policy=my-policy

# Create an approle batch token
vault write auth/approle/role/hcvop policies=devops \
  token_type="batch" \
  token_ttl="60s"

```

See [slides 1-14](operations-training/06-Scale-Vault-for-Performance.pdf) for more information

## Use Cases of Performance Standby Nodes

- Vault Open Source clusters are scale up.  Only the master node can perform reads and writes, the standby nodes must forward all traffic to the master.
- Vault Enterprise clusters are scale out.  The standby nodes can perform reads but must send writes to the master.  These are called Performance Standby nodes.
- This feature is only available in Vault Enterprise
- See [slides 15-23](operations-training/06-Scale-Vault-for-Performance.pdf) for more information

## Enable and Configure Performance Replication

- This feature is only available in Vault Enterprise
- See [slides 24-35](operations-training/06-Scale-Vault-for-Performance.pdf) for more information

## Creating a Paths Filter

Creates an allow or deny list to determine what paths get replicated to other clusters.  
This is useful for the cases where we have clusters in different regions and there are local laws that restrict the export of user information.  For example, we cannot export user PII from Europe to other regions so we could restrict the replication of the secrets engine that has that data.

- An allow list would **only** allow those paths listed
- A deny list would replicate all paths not listed in the deny list
- We can also restrict a mount with the `-local` flag so it is not replicated.  For example if we wanted a `kv-v2` secrets engine called `apac` on the secondary cluster to not be replicated back to the primary we could run  
`vault secrets enable -path=apac -local kv-v2`
- Enable a paths filter on a secondary cluster with  
`vault write sys/replication/performance/primary/paths-filter/<cluster-id> mode=allow paths=aws/,hcvop/,customers/`
- This feature is only available in Vault Enterprise
- See [slides 36-45](operations-training/06-Scale-Vault-for-Performance.pdf) for more information
