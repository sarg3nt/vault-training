<!-- omit from toc -->
# Build Fault-Tolerant Vault Environments

- [Configure a Highly Available Cluster](#configure-a-highly-available-cluster)
  - [What Should a Cluster Look Like?](#what-should-a-cluster-look-like)
- [Integrated Storage](#integrated-storage)
- [Enable and Configure Disaster Recovery Replication (Enterprise)](#enable-and-configure-disaster-recovery-replication-enterprise)
- [Promote a Secondary Cluster (Enterprise)](#promote-a-secondary-cluster-enterprise)

## Configure a Highly Available Cluster

### What Should a Cluster Look Like?

- Ideally, we want something that provides redundancy, failure tolerance, scalability, and a fully replicated architecture
- For Vault Enterprise, you are limited to either Integrated Storage or Consul storage backends
- HashiCorp (and consultants like me) are moving away from Consul as the primary storage backend and using Integrated Storage for everything
- The Vault Operations Professional exam will NOT feature Consul as a configuration or deployment option

## Integrated Storage

The content that was in this section was integrated into the install content here: [01 Vault Install and Setup](01%20Vault%20Install%20and%20Setup.md#integrated-storage)

## Enable and Configure Disaster Recovery Replication (Enterprise)

- This feature is only available in Vault Enterprise
- [Slides 14-41](operations-training/04-Build-Fault-Tolerant-Vault-Environments.pdf))

## Promote a Secondary Cluster (Enterprise)

- This feature is only available in Vault Enterprise
- [Slides 42-52](operations-training/04-Build-Fault-Tolerant-Vault-Environments.pdf))
