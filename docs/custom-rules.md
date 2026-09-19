# Custom Falco Rules

This project contains 10 custom runtime rules.

## 1. Interactive Shell in Container

Detects an interactive shell with a TTY inside a container.

Key shell names:

```text
bash sh zsh ash dash ksh
```

Controlled test:

```bash
kubectl -n dashboard exec -it \
  backend-5b94fccfcc-mzp24 -- sh
```

## 2. Sensitive Credential File Read in Container

Detects suspicious reads of:

- `/etc/shadow`
- SSH private keys
- Kubernetes service-account tokens

The service-account token logic was tuned to shell-like and CLI-style tools because the first broad version created legitimate noise from ArgoCD, Cilium, and CoreDNS.

Controlled test:

```bash
kubectl -n dashboard exec \
  backend-5b94fccfcc-mzp24 -- \
  sh -c 'cat /var/run/secrets/kubernetes.io/serviceaccount/token >/dev/null'
```

## 3. Privilege Changing Tool in Container

Detects tools from Falco's `userexec_binaries` list, including:

```text
sudo su suexec critical-stack dzdo
```

Controlled test:

```bash
kubectl -n dashboard exec \
  backend-5b94fccfcc-mzp24 -- \
  su --version
```

The test prints a version and does not actually switch users.

## 4. Namespace Manipulation Tool in Container

Detects:

```text
nsenter
unshare
```

These can appear during container escape or privileged debugging activity.

## 5. Container Runtime Socket Access

Detects connections to runtime sockets such as:

```text
/var/run/docker.sock
/run/docker.sock
/run/containerd/containerd.sock
/var/run/containerd/containerd.sock
```

Runtime socket access can provide powerful control over containers or the host runtime.

## 6. Suspicious Network Tool in Container

Detects execution of tools such as:

```text
nc ncat netcat socat nmap masscan telnet netstat ss
```

This rule detects suspicious tool execution; it is not a complete reverse-shell detector.

## 7. Download Piped to Shell

Detects command patterns such as:

```text
curl ... | sh
wget ... | bash
```

This can be legitimate, but it is also a common malicious execution pattern.

## 8. SSH Private Key Access in Container

Detects reads of paths including:

```text
/.ssh/id_rsa
/.ssh/id_ed25519
/.ssh/id_ecdsa
/.ssh/id_dsa
/.ssh/authorized_keys
```

## 9. Kubectl Executed in Container

Detects `kubectl` execution inside a container.

A compromised workload with Kubernetes API credentials and `kubectl` can use it for reconnaissance or lateral movement.

## 10. Suspicious System Modification or Temp Execution

Detects:

- writes under `/etc/`
- writes under `/proc/sys/`
- cron-related writes
- execution from `/tmp/`
- execution from `/dev/shm/`
- execution from `/run/shm/`

### Tuning

The broad `/etc/` write condition originally alerted on `/bin/prometheus-config-reloader` in the `monitoring` namespace.

Legitimate generated paths included:

```text
/etc/prometheus/config_out/
/etc/alertmanager/config_out/
```

The final rule excludes only that specific process + namespace + generated-path combination instead of excluding the whole namespace.

## Rule matching mode

```yaml
falco:
  rule_matching: all
```

`rule_matching: first` originally caused a built-in shell rule to shadow the custom interactive-shell rule. `all` allows overlapping built-in and custom detections.

## Tuning method

```text
write rule
   |
   v
trigger safely
   |
   v
observe real alerts
   |
   v
identify false positives
   |
   v
add narrow exception
   |
   v
retest
```
