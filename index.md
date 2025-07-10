<!-- cSpell:ignore -->
# Notes

These docs are a combination of my notes from the HashiCorp Certified: Vault Associate training [Kodekloud](https://kodekloud.com/courses/hashicorp-certified-vault-associate-certification/), [Udemy](https://www.udemy.com/course/hashicorp-vault/) and HashiCorp Certified: Vault Operations Professional 2022 training [KodeKloud](https://kodekloud.com/courses/hashicorp-certified-vault-operations-professional-2022/), [Udemy](https://www.udemy.com/course/hashicorp-certified-vault-operations-professional/)

The slides for these two trainings can be seen in the [associate-training](associate-training/) and [operations-training](operations-training/) folders.

## Table of Contents

1. [Vault Install and Setup](01%20Vault%20Install%20and%20Setup)
2. [Fault-Tolerant Vault Environments](02%20Fault-Tolerant%20Vault%20Environments)
3. [Monitoring a Vault Environment](03%20Monitoring%20a%20Vault%20Environment)
4. [Production Hardening](04%20Production%20Hardening)
5. [Auth Methods](05%20Auth%20Methods)
6. [Access Control and Policies](06%20Access%20Control%20and%20Policies)
7. [Maintenance](07%20Maintenance)
8. [HSM Integration (Enterprise)](08%20HSM%20Integration%20(Enterprise))
9. [Vault Security Model](09%20Vault%20Security%20Model)
10. [Secrets Engines](10%20Secrets%20Engines)
11. [Vault Agent](11%20Vault%20Agent)
12. [Scale Vault For Performance](12%20Scale%20Vault%20For%20Performance)

## General Notes

```bash
# Start vault in dev mode
vault server -dev
```

- Master key has been renamed to root key

## Token Types and Their Prefix

- Service tokens --> `hvs.*`
- Batch tokens --> `hvb.*`
- Recovery tokens --> `hvr.*`
