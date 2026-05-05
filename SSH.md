# How SSH Works

A practical guide to understanding the Secure Shell (SSH) protocol — how it establishes secure connections, authenticates users, and protects data in transit.

## Table of Contents

- [What is SSH?](#what-is-ssh)
- [Why SSH Exists](#why-ssh-exists)
- [Core Components](#core-components)
- [How an SSH Connection Works](#how-an-ssh-connection-works)
- [Authentication Methods](#authentication-methods)
- [SSH Keys Explained](#ssh-keys-explained)
- [Common Use Cases](#common-use-cases)
- [Practical Commands](#practical-commands)
- [Security Best Practices](#security-best-practices)
- [Troubleshooting](#troubleshooting)

---

## What is SSH?

**SSH (Secure Shell)** is a cryptographic network protocol that enables secure communication between two machines over an unsecured network. It is most commonly used to:

- Log into remote servers
- Execute commands remotely
- Transfer files securely
- Tunnel network traffic

SSH operates on **TCP port 22** by default and replaces older insecure protocols like Telnet, rlogin, and FTP.

---

## Why SSH Exists

Before SSH, protocols like Telnet transmitted data — including passwords — in **plaintext**. Anyone sniffing the network could capture credentials and sensitive information.

SSH solved this by introducing:

1. **Encryption** — All data is encrypted end-to-end.
2. **Authentication** — Both the server and client can verify each other's identity.
3. **Integrity** — Data cannot be tampered with in transit without detection.

---

## Core Components

SSH follows a **client-server architecture**:

| Component | Description |
|-----------|-------------|
| **SSH Client** | The program that initiates the connection (e.g., `ssh` command, PuTTY) |
| **SSH Server (sshd)** | The daemon running on the remote machine listening for incoming connections |
| **SSH Keys** | Cryptographic key pairs used for authentication |
| **Known Hosts** | A local file (`~/.ssh/known_hosts`) tracking trusted servers |
| **Authorized Keys** | A server-side file (`~/.ssh/authorized_keys`) listing permitted public keys |

---

## How an SSH Connection Works

An SSH connection is established in **three main phases**:

### Phase 1: TCP Connection & Protocol Negotiation

The client opens a TCP connection to the server (port 22 by default). Both sides exchange their supported SSH protocol versions and agree on a common version (typically SSH-2).

### Phase 2: Key Exchange & Encryption Setup

This is where the magic happens. SSH uses the **Diffie-Hellman key exchange** algorithm (or its elliptic-curve variant) to allow both parties to derive a shared secret over an insecure channel — without ever transmitting the secret itself.

The general flow:

1. The server sends its **host public key** to the client.
2. The client checks this key against `~/.ssh/known_hosts`. If unknown, you'll see the famous *"The authenticity of host can't be established"* prompt.
3. Both sides perform a Diffie-Hellman exchange to derive a **shared session key**.
4. From this point onward, all communication is **symmetrically encrypted** (using algorithms like AES-256).

> **Why symmetric encryption after asymmetric?** Symmetric encryption is much faster. Asymmetric crypto is only used to safely establish the shared key.

### Phase 3: User Authentication

Once the encrypted channel is set up, the client must prove who it claims to be. This is typically done via a password or — more commonly and securely — via SSH keys.

---

## Authentication Methods

SSH supports several authentication mechanisms:

### 1. Password Authentication

The simplest method. The user types a password, which is sent through the encrypted tunnel.

**Pros:** No setup required.
**Cons:** Vulnerable to brute-force attacks; weak passwords are dangerous.

### 2. Public Key Authentication (Recommended)

Uses an asymmetric key pair. The private key stays on the client; the public key is placed on the server.

**Pros:** Far more secure; supports passwordless login; resistant to brute force.
**Cons:** Requires initial setup.

### 3. Multi-Factor Authentication (MFA)

Adds an additional factor (TOTP, hardware key, etc.) on top of the above.

### 4. Host-Based & Kerberos

Less common, used in specific enterprise environments.

---

## SSH Keys Explained

An SSH key pair consists of:

- **Private key** (e.g., `id_ed25519`) — Kept secret on your local machine. **Never share this.**
- **Public key** (e.g., `id_ed25519.pub`) — Safe to share; placed on servers you want to access.

### How Key Authentication Works

1. The server has your public key in its `authorized_keys` file.
2. When you connect, the server generates a random challenge and encrypts it with your public key.
3. Only your private key can decrypt this challenge.
4. Your client decrypts it, signs the response, and sends it back — proving you hold the private key without ever transmitting it.

### Recommended Key Types

| Algorithm | Recommendation |
|-----------|----------------|
| **Ed25519** | Modern, fast, secure — preferred default |
| **RSA (4096-bit)** | Widely supported; use only if Ed25519 is unavailable |
| **ECDSA** | Acceptable, but Ed25519 is generally preferred |
| **DSA** | Deprecated — do not use |

---

## Common Use Cases

- **Remote Server Login** — Manage cloud servers (AWS EC2, DigitalOcean, etc.)
- **Git over SSH** — Push/pull from GitHub, GitLab using SSH URLs
- **File Transfer** — `scp` and `sftp` use SSH under the hood
- **Port Forwarding / Tunneling** — Securely access services behind firewalls
- **Running Remote Commands** — Automate operations across many machines

---

## Practical Commands

### Generate a New SSH Key

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

### Copy Your Public Key to a Server

```bash
ssh-copy-id user@remote-server
```

### Connect to a Remote Server

```bash
ssh user@remote-server
```

### Connect on a Custom Port

```bash
ssh -p 2222 user@remote-server
```

### Copy a File to a Remote Server

```bash
scp local-file.txt user@remote-server:/path/to/destination/
```

### Local Port Forwarding

```bash
ssh -L 8080:localhost:80 user@remote-server
```

This forwards your local port `8080` to port `80` on the remote machine.

### Use a Specific Key

```bash
ssh -i ~/.ssh/custom_key user@remote-server
```

---

## Security Best Practices

1. **Disable password authentication** on production servers — use keys only.
2. **Disable root login** — set `PermitRootLogin no` in `/etc/ssh/sshd_config`.
3. **Change the default port** (22) to reduce automated scan noise.
4. **Use a passphrase** on your private key.
5. **Use `ssh-agent`** to avoid retyping passphrases.
6. **Rotate keys periodically** and remove unused public keys from `authorized_keys`.
7. **Use `fail2ban`** or similar to block repeated failed attempts.
8. **Keep SSH software updated** — both client and server.
9. **Restrict access by IP** when possible (firewall rules).
10. **Audit your `~/.ssh/config`** and known_hosts regularly.

---

## Troubleshooting

### Permission Denied (publickey)

- Check that your public key is in the server's `~/.ssh/authorized_keys`.
- Verify file permissions: `~/.ssh` should be `700`, `authorized_keys` should be `600`.
- Run with verbose mode: `ssh -v user@host` to see what's happening.

### Connection Refused

- The SSH server may not be running on the target machine.
- A firewall may be blocking port 22.
- Wrong port or hostname.

### Host Key Verification Failed

- The server's host key changed (legitimate reinstall, or a possible MITM attack).
- Remove the old entry: `ssh-keygen -R hostname`.

### Slow Connection

- Try `UseDNS no` in `sshd_config` on the server.
- Use `GSSAPIAuthentication no` on the client.

---

## Further Reading

- [OpenSSH Official Documentation](https://www.openssh.com/manual.html)
- [RFC 4251 — The SSH Protocol Architecture](https://datatracker.ietf.org/doc/html/rfc4251)
- [SSH Academy by SSH.com](https://www.ssh.com/academy/ssh)

---

*Maintained as a quick reference for understanding how SSH works under the hood.*
