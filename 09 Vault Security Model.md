<!-- omit from toc -->
# Vault Security Model

- [Secure Introduction of Vault Clients](#secure-introduction-of-vault-clients)
  - [What is Secret Zero](#what-is-secret-zero)
  - [Secure Introduction Goals](#secure-introduction-goals)
  - [Methods of Providing Authentication to Vault](#methods-of-providing-authentication-to-vault)
- [Security Implications of Running Vault in Kubernetes](#security-implications-of-running-vault-in-kubernetes)
  - [TLS End to End Encryption](#tls-end-to-end-encryption)
  - [Disable Core Dumps](#disable-core-dumps)
  - [Ensure `mlock` is Enabled](#ensure-mlock-is-enabled)
  - [Container Supervisor](#container-supervisor)
  - [Don't Run Vault as Root](#dont-run-vault-as-root)
  - [Running Vault on Kubernetes Links](#running-vault-on-kubernetes-links)

## Secure Introduction of Vault Clients

### What is Secret Zero

- Secret zero is essentially the "first secret" needed to obtain other secrets
- Example: 1Password or LastPass
- In Vault, this is either the authentication credentials or a Vault token
- Once we have secret zero, we can potentially obtain other credentials. Unfortunately, it also allows for an unauthorized user to elevate privileges in the organization
- The goal is to introduce secret zero in the most secure fashion but only when it's needed for the application to use it

### Secure Introduction Goals

1. Use unique credentials for each application instance provisioned
1. Limit your exposure if a credential is compromised
1. Stop hardcoding credentials within the application codebase
1. Reduce the TTL of the credentials used by applications and reduce long-lived creds
1. Distribute credentials securely and only at runtime
1. Use a trusted platform to verify the identities of clients
1. Employ a trusted orchestrator that is already authenticated to Vault to inject secrets

### Methods of Providing Authentication to Vault

- Secure Platform: [slide 4](operations-training/03-Employ-the-Vault-Security-Model.pdf))
- Secure Orchestrator (CI/CD): [slide 5](operations-training/03-Employ-the-Vault-Security-Model.pdf))
- Secure Orchestrator (Terraform): [slide 6](operations-training/03-Employ-the-Vault-Security-Model.pdf))
- Vault Agent - Auto Auth: [slide 7](operations-training/03-Employ-the-Vault-Security-Model.pdf))

[Slides 1-7](operations-training/03-Employ-the-Vault-Security-Model.pdf)

## Security Implications of Running Vault in Kubernetes

### TLS End to End Encryption

- Don't offload TLS at the load balancer
- Ensures end-to-end encryption from the client to the Vault node
- Use TLS certificates signed by a trusted Certificate Authority (CA)
- Require TLS 1.2+

### Disable Core Dumps

- Most commonly, Vault pods are scheduled to run on a separate cluster to reduce/eliminate shared resources
- Core dump files may include Vault's encryption keys
- Ensure `RLIMIT_CORE` is set to 0 or use the `ulimit` command with the core flag  
  `ulimit -c 0`  
  inside the container to ensure your container processes can't core dump.

### Ensure `mlock` is Enabled

- Memory lock ensures memory from a process on a Linux system isn't swapped to disk. Additional configurations are needed for containerized deployments
- The process that starts the container that runs the mlock call must have `IPC_LOCK` capabilities

```yaml
securityContext: 
runAsNonRoot: true 
runAsUser: 1000 
capabilities: 
    add: ["IPC_LOCK"]
```

### Container Supervisor

- If your container starts as root, the processes that might escape that container will also have root on the node
- Mitigations can be used to prevent starting your container as root
  - SecurityContext --> `runAsNonRoot`
  - PodSecurityContext --> `runAsNonRoot`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-world
spec:
  containers:
  # specification of the pod’s containers
  # ...
  securityContext:
    readOnlyRootFilesystem: true
    runAsNonRoot: true
```

### Don't Run Vault as Root

- Vault is designed to run as an unprivileged user – regardless of the platform
- Elevated privileges can potentially expose the Vault process memory and allow access to Vault encryption keys

### Running Vault on Kubernetes Links

- [Learn How to Run Vault on Kubernetes](https://www.hashicorp.com/blog/learn-how-to-run-vault-on-kubernetes)
- [Run Vault on Kubernetes Documentation](https://developer.hashicorp.com/vault/docs/platform/k8s/helm/run)
- [Vault on Kubernetes Deployment Guide](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide)
- [Running Vault with Kubernetes](https://www.hashicorp.com/products/vault/kubernetes)
- [Vault Agent with Kubernetes](https://developer.hashicorp.com/vault/tutorials/kubernetes/agent-kubernetes)
- [Slides 8-14](operations-training/03-Employ-the-Vault-Security-Model.pdf)
