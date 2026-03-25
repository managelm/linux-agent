## What is ManageLM?

ManageLM lets you manage your entire Linux infrastructure with natural language through Claude, ChatGPT, VS Code, Slack, or any MCP-compatible client.

The **agent** is a lightweight Python daemon that runs on each managed server. It connects outbound to the ManageLM portal, receives tasks, uses an LLM to interpret and execute operations, and reports results back — all within strict security boundaries.

```
┌─────────────┐       MCP (HTTPS)       ┌──────────────────┐    WebSocket/HTTPS    ┌─────────────┐
│   Claude     │ ◄─────────────────────► │   ManageLM       │ ◄───────────────────► │   Agent     │
│  (MCP Client)│                         │   Portal         │                       │  (on host)  │
└─────────────┘                          └──────────────────┘                       └─────────────┘
                                                │                                         │
                                         ┌──────┴──────┐                           ┌──────┴──────┐
                                         │ PostgreSQL  │                           │ Local LLM   │
                                         │ Redis/Valkey│                           │(Ollama, etc)│
                                         └─────────────┘                           └─────────────┘
```

### Key features

- **Outbound-only connections** — agents always initiate. No inbound ports required on managed servers.
- **Skill-scoped execution** — agents can only run operations defined by assigned skills. Enforced in code, not just prompts.
- **4-layer command security** — injection blocking → binary allowlist → destructive argument guard → kernel sandbox (Landlock + seccomp-bpf).
- **Read-only by default** — no allowed commands = read-only. You explicitly grant what each agent can do.
- **Local LLM support** — use Ollama, LocalAI, or any cloud provider. Task interpretation stays on your infrastructure.
- **32 built-in skills** — packages, services, firewall, users, web servers, databases, containers, security hardening, and more.
- **Interactive tasks** — when the LLM needs input it can't determine on its own, it asks instead of guessing.
- **File change tracking** — every mutating task is git-tracked with full diffs and one-click revert.
- **Zero OS footprint** — all dependencies installed in `/opt/managelm/pip/`. Nothing touches system Python.

---

## Requirements

- **Linux** server (any distribution)
- **Python 3.9+**
- **systemd**
- A **ManageLM account** ([SaaS](https://app.managelm.com) or [self-hosted](https://hub.docker.com/r/managelm/portal))

---

## Quick install

Download the agent package from your ManageLM portal, then run:

```bash
sudo bash managelm-setup.sh
```

The setup script will:

1. Check prerequisites (Python 3.9+, systemd)
2. Create a dedicated `managelm` system user
3. Install files to `/opt/managelm/`
4. Install Python dependencies in an isolated directory (zero OS footprint)
5. Prompt for your admin email, portal URL, and server group
6. Enroll the agent with the portal
7. Wait for admin approval
8. Start the `managelm-agent` systemd service

### Self-update & uninstall

```bash
# Update to latest version (triggered from portal or manually)
sudo /opt/managelm/bin/managelm-update

# Complete removal (stops service, removes files, unregisters from portal)
sudo /opt/managelm/bin/managelm-remove
```

---

## Configuration

The agent is configured via `/opt/managelm/config.yaml`:

```yaml
server_url: "https://app.managelm.com"
access_token: "agt_..."
agent_id: "..."
signing_pubkey: "..."
secrets_file: "/opt/managelm/secrets.txt"
```

### LLM configuration

The LLM endpoint is configured from the portal (per-skill, per-agent, or account-wide). Supported providers:

| Provider | Type | Auto-detected from URL |
|----------|------|----------------------|
| **Ollama** | Local | `localhost:11434` |
| **LocalAI** | Local | Auto-detected |
| **Anthropic** | Cloud | `anthropic.com` |
| **OpenAI** | Cloud | `openai.com` |
| **Google Gemini** | Cloud | `generativelanguage.googleapis.com` |
| **Groq** | Cloud | `groq.com` |
| **Mistral** | Cloud | `mistral.ai` |
| **DeepSeek** | Cloud | `deepseek.com` |
| **xAI (Grok)** | Cloud | `x.ai` |
| **Together** | Cloud | `together.xyz` |
| **Fireworks** | Cloud | `fireworks.ai` |
| **Perplexity** | Cloud | `perplexity.ai` |
| **Cohere** | Cloud | `cohere.com` |

All non-Anthropic providers use the OpenAI-compatible `/v1/chat/completions` format.

### Secrets

Store credentials in `/opt/managelm/secrets.txt` (never leaves the server):

```
DB_PASSWORD=s3cret
API_KEY=sk-...
```

The LLM only sees variable names (`$DB_PASSWORD`), never values. Values are injected into command environment at execution time.

---

## How it works

1. Agent connects to the portal via WebSocket (outbound only)
2. Portal sends a task request (Ed25519-signed)
3. **Skill gate** validates the task against assigned skills
4. **Task executor** builds LLM prompt with skill scope, allowed operations, and system context
5. LLM generates commands using `<cmd>` tags
6. **Command validator** checks each command through 3 layers:
   - Layer 1: Injection blocking (regex — blocks `$()`, backticks, `eval`, etc.)
   - Layer 2: Binary allowlist (every binary must be explicitly allowed)
   - Layer 3: Destructive argument guard (blocks `rm -rf /`, `dd of=/dev/sda`, etc.)
7. **Kernel sandbox** (optional) — Landlock filesystem confinement + seccomp-bpf syscall filtering
8. Command executes, output feeds back to LLM for next turn
9. Continues until task is complete (max 10 turns)
10. File changes in `/etc/` are git-tracked for audit and revert

---

## Security

### 4-layer command validation

```
Command from LLM
  │
  ├─ Layer 1: Injection blocking       (regex — blocks $(), eval, backticks)
  ├─ Layer 2: Binary allowlist          (must be in skill's allowed_commands)
  ├─ Layer 3: Destructive argument guard (blocks catastrophic arguments)
  │
  ▼ subprocess.run(preexec_fn=sandbox)
  │
  ├─ Layer 4a: Landlock LSM             (kernel filesystem confinement)
  └─ Layer 4b: seccomp-bpf             (kernel syscall blocklist)
```

### Additional security measures

- **Ed25519 signature verification** — all portal-to-agent messages are cryptographically signed and verified before processing
- **Read-only by default** — empty `allowed_commands` means agents can only use base read-only commands (~45 commands: cat, ls, grep, ps, df, etc.)
- **Protected root directories** — 20+ root paths (`/`, `/bin`, `/etc`, `/usr`, `/var`, ...) are protected against recursive destructive operations
- **Path traversal protection** — `rm -rf /etc/..` resolves to `/` and is blocked
- **Secrets isolation** — credentials stay on the server, injected as environment variables at execution time
- **Kernel sandbox** — opt-in Landlock (filesystem confinement) + seccomp-bpf (syscall filtering) per skill
- **Execution limits** — max 10 turns per task, 120s command timeout, 8KB output truncation

---

## Built-in skills (32)

| Skill | Description | Key commands |
|-------|-------------|-------------|
| **Base** | Cross-skill read-only utilities | cat, ls, grep, ps, df, free, ip, curl (~45 cmds) |
| System | System info, monitoring, reboot | hostnamectl, timedatectl, sysctl, reboot |
| Files | File & directory management | tee, chmod, chown, cp, mv, rm, tar, rsync |
| Services | Services & process management | systemctl, journalctl, kill, crontab |
| Packages | Package management (all distros) | apt, dnf, yum, pacman, zypper, apk, pip, snap |
| Users | Users & access management | useradd, usermod, passwd, groupadd, visudo |
| Network | Network configuration | ip, nmcli, networkctl, ethtool, tc |
| Firewall | Firewall management | ufw, firewall-cmd, iptables, nft |
| Containers | Container management | docker, podman, buildah, compose |
| Web Server | Web server management | nginx, apache2ctl, caddy, haproxy, certbot |
| Web Apps | Web application management | pm2, gunicorn, uvicorn, supervisor |
| Database | SQL database management | mysql, psql, pg_dump, sqlite3 |
| NoSQL | NoSQL database management | mongosh, redis-cli, memcstat |
| Storage | Storage & disk management | fdisk, parted, lvm, mount, smartctl |
| Security | Security & hardening | fail2ban, certbot, openssl, semanage |
| Certificates | Certificates & PKI | openssl, certbot, keytool, easyrsa |
| Logs | Log analysis (read-only) | journalctl, zcat, zgrep |
| Monitoring | Monitoring & alerting | sar, mpstat, pidstat, iotop, iftop |
| Git | Git repository management | git, ssh-keygen |
| Backup | Backup & restore | tar, rsync, scp, gzip |
| DNS | DNS server management | named, unbound, dnsmasq |
| Email | Email server management | postfix, dovecot, opendkim, spamassassin |
| VPN | VPN & tunnel management | wg, openvpn, ipsec, easyrsa |
| Virtualization | VM management | virsh, virt-install, lxc, qm, vagrant |
| Kubernetes | Kubernetes management | kubectl, kubeadm, helm, k3s |
| Message Queue | Message queue management | rabbitmqctl, kafka-topics, nats |
| File Sharing | File sharing & FTP | exportfs, smbpasswd, vsftpd |
| LDAP | Directory services | ldapsearch, ldapadd, slapcat, ipa |
| Automation | Automation & IaC | ansible, terraform, packer |
| Proxy | Proxy & cache | squid, varnishadm, haproxy, keepalived |
| LLM | LLM & AI infrastructure | ollama, llamacpp, vllm, nvidia-smi |
| Developer | Developer tools | bash, python3, node, npm, gcc, make, go |

Custom skills can be created from the portal with your own allowed commands, operations, and system prompts.

---

## Built-in reports

The agent includes deterministic scanners that run without consuming task credits:

| Report | Description |
|--------|-------------|
| **Security Audit** | 19 checks: SSH config, firewall, open ports, users, SUID binaries, TLS, Docker, kernel params. Returns findings with severity and remediation steps. |
| **System Inventory** | 15 checks: services, packages, containers, network, storage, hardware, cron jobs. Categorized and structured output. |
| **Access Scan** | SSH authorized_keys + sudoers rules per system user, with identity mapping via ManageLM profiles. |
| **Dependency Scan** | Maps what services a server provides and what it depends on. Parses configs across web servers, databases, Docker, mail, LDAP, and more. Fully deterministic, no LLM. |

---

## System metrics

The agent collects lightweight metrics from `/proc` and sends them to the portal for 24h graphing:

- **CPU** — usage percentage (two-snapshot delta from `/proc/stat`)
- **Memory** — usage percentage (from `/proc/meminfo`)
- **Disk** — root filesystem usage
- **Load** — 1-minute load average

Collected every 5 minutes. No external monitoring tools required.

---

## File layout

```
/opt/managelm/
├── managelm-agent.py          # Main daemon
├── config.yaml                # Portal connection config
├── secrets.txt                # Local secrets (KEY=VALUE, never leaves the server)
├── requirements.txt           # Python dependencies
├── pip/                       # Isolated Python deps (zero OS footprint)
├── logs/                      # Rotating log files (5MB x 3)
├── git/                       # File change tracking (local git repo)
├── bin/
│   ├── managelm-update        # Self-update script
│   └── managelm-remove        # Uninstall script
└── lib/
    ├── task_executor.py       # Multi-turn LLM task execution
    ├── command_validator.py   # 3-layer command validation
    ├── sandbox.py             # Kernel sandbox (Landlock + seccomp-bpf)
    ├── skill_gate.py          # Skill execution gating
    ├── websocket_client.py    # WebSocket client with reconnection
    ├── changeset.py           # File change tracking (git)
    ├── config_loader.py       # YAML config loading
    ├── file_transfer.py       # Secure file upload/download
    ├── ed25519_verify.py      # Ed25519 signature verification
    ├── security_audit.py      # Built-in security audit
    ├── system_inventory.py    # Built-in system inventory
    ├── dependency_scan.py     # Built-in dependency scanner
    ├── ssh_keys_scan.py       # Built-in SSH & sudo access scanner
    └── command_logger.py      # Command execution logging
```

---

## Integrations

ManageLM works with your existing tools:

| Plugin | Description |
|--------|-------------|
| [Claude](https://www.managelm.com/plugins/claude.html) | MCP server for Claude Desktop & Claude Code |
| [ChatGPT](https://www.managelm.com/plugins/chatgpt.html) | GPT action for OpenAI ChatGPT |
| [VS Code](https://www.managelm.com/plugins/vscode.html) | VS Code extension |
| [n8n](https://www.managelm.com/plugins/n8n.html) | n8n community node for workflow automation |
| [Slack](https://www.managelm.com/plugins/slack.html) | Slack bot integration |

---

## Deployment options

| Mode | Description |
|------|-------------|
| **SaaS** | Hosted at [app.managelm.com](https://app.managelm.com). Free for up to 10 agents. |
| **Self-hosted** | Deploy with Docker. Image: [`managelm/portal`](https://hub.docker.com/r/managelm/portal). |

---

## Links

- [Website](https://www.managelm.com)
- [Documentation](https://app.managelm.com/docs)
- [Docker Hub](https://hub.docker.com/r/managelm/portal)
- [Changelog](CHANGELOG.md)

---

## License

Copyright ManageLM / RCDevs S.A. All rights reserved. See [LICENSE](LICENSE) for details.
