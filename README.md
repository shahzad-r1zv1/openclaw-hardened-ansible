# OpenClaw Hardened Deployment (Ansible)

**Automated Tier 3+ Security Hardening for OpenClaw AI Agents**

This Ansible playbook implements and extends the security hardening measures described in the [OpenClaw Security Guide](https://nextkicklabs.substack.com/p/openclaw-hardened-deployment-security-with-ansible), providing a fully automated deployment with additional defense-in-depth layers.

## 🎯 What This Playbook Does

Deploys a **hardened OpenClaw installation** with:
- **Rootless Podman containers:** Strict isolation running as a non-privileged user.
- **Network egress filtering:** Squid proxy sidecar with a domain allowlist.
- **HTTPS termination:** Caddy reverse proxy with auto-generated self-signed certificates.
- **LiteLLM credential brokering:** OpenClaw never sees real API keys; LiteLLM spoofs models (e.g., Deepseek acting as Claude).
- **Consolidated Configuration:** Single `openclaw.json` master config for Gateway, Tools, and Agents.
- **Automated Identity:** EFF wordlist hostname generation and persistent SSH key management.
- **Multi-OS support:** Native tasks for **Arch Linux** and **Debian/Ubuntu** (AWS ready).
- **Security Monitoring:** Systemd-based weekly audits for prompt injections and blocked domains.

## 📊 Comparison: Article vs. This Implementation

| Feature | Original Article (Tier 3) | This Ansible Implementation |
|---------|---------------------------|------------------------------|
| **Container Runtime** | Docker | **Podman (rootless)** ⭐ |
| **Network Filtering** | Firewall only | **Firewall + Squid egress allowlist** ⭐ |
| **HTTPS** | Optional/Manual | **Caddy reverse proxy (Terminated HTTPS)** ⭐ |
| **Identity Management** | Manual setup | **Automated EFF wordlist generation** |
| **OS Support** | Ubuntu focus | **Arch + Debian/Ubuntu auto-detection** |
| **Deployment Method** | Manual | **Fully automated interactive script** |
| **Monitoring** | Manual cron | **Systemd timers + audit script** |
| **LLM Providers** | Anthropic focus | **Ollama (Deepseek) / Anthropic / OpenAI** |
| **Secrets Management** | Manual generation | **Auto-gen with PERSISTENCE across runs** ⭐ |
| **Access Control** | Token only | **Token + Manual Device Pairing** |

## 📋 Prerequisites

**Local Machine (Controller):**
- Ansible 2.10+
- OpenSSL (for cert generation)
- SSH Client (`ssh-keygen`)
- Python 3.8+

**Target Machine:**
- Arch Linux OR Debian/Ubuntu
- Initial root/sudo access (Password or AWS .pem key)
- 2GB+ RAM

## 🚀 Quick Start

### 1. Prepare
```bash
cd openclaw-hardened-ansible
chmod +x deploy.sh update-allowlist.sh
```

### 2. Deploy
Run the interactive script. It will prompt for your IP, provider, and keys.
```bash
./deploy.sh
```

**AWS/Cloud Example:**
```bash
./deploy.sh \
  --target 54.x.x.x \
  --ssh-user ubuntu \
  --ssh-key ~/my-aws-key.pem \
  --mgmt-cidr 192.168.20.0/24 \
  --provider ollama \
  --model "deepseek-r1:8b" \
  --url "http://10.100.1.25:11434"
```

### 3. Authenticate
Once finished, get your persistent token:
```bash
ssh -i ssh-keys/your-name.pem openclaw@IP "cat ~/openclaw-docker/.env | grep TOKEN"
```
Open **`https://IP:18789`**, click through the SSL warning, and paste the token in Settings.

### 4. Approve Device (Hardening Step 7)
Since device auth is enabled, you must approve your browser from the host CLI:
```bash
# Inside the OpenClaw host
podman exec openclaw-agent openclaw devices pending
podman exec openclaw-agent openclaw devices approve <YOUR_ID>
```

## 🔧 Maintenance

### Update Egress Allowlist
1. Edit `roles/tier3-setup/templates/allowlist.txt.j2`.
2. Run `./update-allowlist.sh -t IP --ssh-user USER --ask-pass`.

### Security Audits
A systemd timer runs `monitor-openclaw.sh` weekly. To run manually:
```bash
sudo /home/openclaw/openclaw-docker/monitor-openclaw.sh
```
Check the reports at `~/openclaw-docker/security-audit-YYYYMMDD.log`.

### Configuration Validation
If you see errors, run the OpenClaw "Doctor" to check the schema:
```bash
podman exec openclaw-agent openclaw doctor
```

## 📁 File Structure
- `deploy.sh`: Main entry point (interactive/CLI).
- `update-allowlist.sh`: Lightweight allowlist updater.
- `ssh-keys/`: Stores generated `.pem` and `.crt` files.
- `roles/tier3-setup/`: The core hardening logic.
- `requirements.yml`: Ansible dependencies (auto-installed).

## 🛠️ How to Improve This Playbook

Below are concrete areas where this playbook can be extended and hardened further.

### 1. Add Automated Testing with Molecule
Currently there are no automated tests. [Molecule](https://molecule.readthedocs.io/) provides role-level testing for Ansible against real or containerised targets.
```bash
pip install molecule molecule-docker
cd roles/tier3-setup
molecule init scenario default --driver-name docker
molecule test
```
Add `ansible-lint` as well for immediate syntax and best-practice feedback:
```bash
pip install ansible-lint
ansible-lint playbook.yml
```

### 2. Add a CI/CD Pipeline
Create `.github/workflows/ci.yml` to run `ansible-lint` and Molecule tests on every pull request. Example minimal step:
```yaml
- name: Lint playbook
  run: ansible-lint playbook.yml
```

### 3. Expand OS Support
The role currently handles **Arch Linux** and **Debian/Ubuntu**. Add task files for:
- **RHEL / Rocky Linux / AlmaLinux** (`dnf`-based) – popular for enterprise deployments.
- **Alpine Linux** – minimal footprint, good for container-host scenarios.

Create `roles/tier3-setup/tasks/rhel-system.yml` following the same pattern as `arch-system.yml`.

### 4. Integrate a Secrets Manager
Storing the LLM API key in a plain `.env` file increases risk if the host is compromised.  
Consider:
- **HashiCorp Vault** (`community.hashi_vault` collection) to inject secrets at runtime.
- **AWS Secrets Manager** or **GCP Secret Manager** for cloud deployments.
- **Ansible Vault** (`ansible-vault encrypt_string`) as a local zero-dependency option.

### 5. Add Centralised Logging
Wire the container stack into a log aggregator so security events survive container restarts:
- **Loki + Grafana** – lightweight, Podman-compatible.
- **OpenSearch** – full-text search on Squid access logs for blocked domain analysis.

Add a `log_driver` key in `docker-compose.yml.j2`:
```yaml
logging:
  driver: journald
  options:
    tag: "openclaw-{{.Name}}"
```

### 6. Add Health Checks to the Container Stack
The current `docker-compose.yml.j2` has no `healthcheck` entries. Adding them allows Podman to restart unhealthy containers automatically:
```yaml
healthcheck:
  test: ["CMD", "curl", "-sf", "http://localhost:4000/health"]
  interval: 30s
  retries: 3
```

### 7. Support Multi-Node / High-Availability Deployments
The playbook targets a single host. To support HA:
- Use an Ansible **inventory group** (`[openclaw_nodes]`) with multiple hosts.
- Add a shared Podman pod network or an external load balancer task.
- Pin secrets to a shared Vault/S3 back-end rather than regenerating per host.

### 8. Implement Automated Backups
Add a `backup.yml` playbook that archives `~/openclaw-docker/openclaw-data` to S3, Backblaze B2, or a local SFTP server on a schedule:
```bash
ansible-playbook -i inventory.ini backup.yml --extra-vars "target=IP"
```

### 9. Upgrade the Security Monitoring Script
The `monitor-openclaw.sh.j2` script runs weekly. Consider:
- Sending audit reports to a Slack/Discord webhook.
- Adding `rkhunter` or `chkrootkit` host-level scans.
- Integrating with `AIDE` for file-integrity monitoring.

### 10. Document Architecture with a Diagram
Add a `docs/architecture.md` with an ASCII or Mermaid diagram showing the full request flow (browser → Caddy → OpenClaw → LiteLLM → Squid → external LLM API) to help new contributors understand the trust boundaries at a glance.

---

## 🔍 Troubleshooting

### Deployment fails at SSH connection
```
UNREACHABLE! => {"msg": "Failed to connect to the host via ssh"}
```
- Verify the target IP is reachable: `ping <TARGET_IP>`
- Check your SSH key path: `ssh -i ssh-keys/<name>.pem <user>@<TARGET_IP>`
- If using password auth, re-run with `--ask-pass`

### Ansible reports "Python not found" on the target
```
MODULE FAILURE: No module named 'json'
```
Add `ansible_python_interpreter=/usr/bin/python3` to your inventory or pass it as an extra var.

### Podman containers do not start after deployment
SSH into the host and check container status:
```bash
ssh -i ssh-keys/<name>.pem openclaw@<IP>
podman ps -a
podman logs openclaw-agent
```

### SSL certificate warning in browser
The self-signed certificate will trigger a browser warning — this is expected. Click **Advanced → Proceed** to continue. To eliminate the warning, replace `ssh-keys/<name>.crt` with a certificate from Let's Encrypt or your internal CA and re-run the playbook.

### `openclaw doctor` reports schema errors
Run the auto-fix flag:
```bash
podman exec openclaw-agent openclaw doctor --fix --yes --non-interactive
```
Then restart the agent:
```bash
podman restart openclaw-agent
```

### Device approval never appears
Ensure you opened `https://IP:18789` in the **same browser** you want to approve, and that the token was entered correctly. Then check for pending devices:
```bash
podman exec openclaw-agent openclaw devices pending
```

---

## 🤝 Contributing

1. Fork the repository and create a feature branch.
2. Follow existing task/template naming conventions (see `roles/tier3-setup/tasks/`).
3. Run `ansible-lint playbook.yml` before opening a PR.
4. Open a pull request describing the change and which OS/provider combination was tested.

---

## 📄 License
Provided as-is for harm-reduction. OpenClaw is architecturally "spicy"—this deployment reduces the blast radius but prompt injection remains an inherent risk of LLMs. Use burner accounts only.
