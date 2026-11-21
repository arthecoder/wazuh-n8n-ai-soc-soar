# Autonomous AI SOC Analyst & Human-in-the-Loop SOAR Pipeline (Wazuh + n8n + Gemini)

An end-to-end **Security Orchestration, Automation, and Response (SOAR)** workflow built with **Wazuh SIEM**, **n8n**, **Google Gemini**, **Telegram**, and **Ubuntu (`ufw`)** to automate Tier 1 alert triage and active incident containment.

Instead of manually triaging raw SIEM logs or allowing an LLM to make unchecked firewall changes, this pipeline forwards high-severity Wazuh alerts to n8n, generates a structured AI threat triage report, and sends an interactive **Human-in-the-Loop** approval card to a SOC analyst via Telegram before executing an automated firewall block on the target endpoint.

---

## Architecture & Workflow

```text
┌─────────────────┐     POST JSON      ┌──────────────────┐     Extracted IOCs     ┌─────────────────────┐
│   Wazuh SIEM    │ ─────────────────► │  n8n Webhook &   │ ─────────────────────► │   AI SOC Agent      │
│ (custom-n8n.py) │   (Level 10+ Only) │  Field Extractor │                        │  (Google Gemini)    │
└─────────────────┘                    └──────────────────┘                        └──────────┬──────────┘
                                                                                              │
┌─────────────────┐    SSH UFW Deny    ┌──────────────────┐    "Block IP" Tap      ┌──────────▼──────────┐
│  Ubuntu Agent   │ ◄───────────────── │   n8n If Gate    │ ◄───────────────────── │  Telegram Bot API   │
│  (Containment)  │                    │ (Approval Check) │   (Cloudflare Tunnel)  │ (Human-in-the-Loop) │
└─────────────────┘                    └──────────────────┘                        └─────────────────────┘
```

### Pipeline Stages
1. **SIEM Alert Forwarding (`Wazuh` + `custom-n8n.py`):** A custom Python integration on the Wazuh Manager filters for high-severity events (`level >= 10`) and forwards JSON alert payloads to the n8n Webhook endpoint.
2. **Log Normalization (`Edit Fields` Node):** Strips noisy metadata to optimize LLM token usage and isolates core threat indicators (`rule.description`, `rule.level`, `agent.name`, `data.srcip`, `data.dstuser`, and `full_log`).
3. **AI Tier 1 Triage (`AI Agent` + `Google Gemini`):** Evaluates alert context, assigns a verdict (*True Positive / False Positive / Needs Investigation*), maps relevant **MITRE ATT&CK** tactics and techniques, and recommends concrete next investigation steps.
4. **Human-in-the-Loop Approval (`Telegram` + `Cloudflare Tunnel`):** Delivers the sanitized triage report to the analyst's Telegram chat with one-tap **`🚫 Block IP`** and **`✅ Ignore`** buttons routed back to n8n over an encrypted Cloudflare Quick Tunnel.
5. **Automated Endpoint Containment (`If` + `SSH` Nodes):** Upon analyst approval, n8n validates the boolean callback and connects via SSH to the affected Ubuntu endpoint to execute `sudo ufw deny from <attacker_ip> to any` using a least-privilege `sudoers` rule.

---

## Tech Stack
* **SIEM & Endpoint:** Wazuh SIEM, Ubuntu Linux (`ufw` host firewall)
* **SOAR Orchestration:** n8n (Self-Hosted via Docker)
* **LLM Engine:** Google Gemini (`gemini-3.8-flash`)
* **ChatOps & Tunneling:** Telegram Bot API (`Send and Wait for Response`), Cloudflare Tunnel (`cloudflared`)

---

## Security & Engineering Guardrails
* **Human-in-the-Loop Containment:** Prevents LLM hallucinations or spoofed log payloads from automatically blocking critical infrastructure by requiring explicit analyst approval prior to containment.
* **Principle of Least Privilege (`sudoers`):** Eliminates hardcoded root credentials and plaintext `sudo` passwords in n8n by scoping `/etc/sudoers.d/n8n-ufw` on the Ubuntu endpoint to allow passwordless execution *strictly* for `/usr/sbin/ufw`.
* **Alert Noise Filtering:** Configured at the Wazuh integration layer (`<level>10</level>`) to prevent informational log floods and excessive API consumption.

---

## Repository Structure

```text
.
├── README.md
├── integrations/
│   └── custom-n8n.py                 # Wazuh Manager custom webhook integration script
├── workflow/
│   └── ai-soc-analyst-soar.json      # Sanitized n8n SOAR workflow export
└── screenshots/
    ├── n8n-workflow-canvas.png       # End-to-end execution with green checkmarks
    ├── telegram-approval-prompt.png  # AI triage summary & Block/Ignore buttons
    └── ubuntu-ufw-containment.png    # Firewall rule applied after approval
```

---

## Setup & Deployment

### 1. Start Cloudflare Tunnel & n8n
Expose local n8n webhook callbacks to Telegram using `cloudflared` and pass the generated HTTPS URL into `WEBHOOK_URL`:

```bash
# Terminal 1: Start encrypted quick tunnel
docker run --rm --network host cloudflare/cloudflared:latest tunnel --url http://localhost:5678

# Terminal 2: Run n8n with persistent storage and callback URL
docker run -d --name n8n -p 5678:5678 \
  -e WEBHOOK_URL="https://<YOUR-TUNNEL-SUBDOMAIN>://" \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n
```

### 2. Configure Wazuh Manager Webhook Integration
Place `integrations/custom-n8n.py` into `/var/ossec/integrations/` on the Wazuh Server with proper permissions:

```bash
sudo chown root:wazuh /var/ossec/integrations/custom-n8n.py
sudo chmod 750 /var/ossec/integrations/custom-n8n.py
```

Add the integration block inside `/var/ossec/etc/ossec.conf` and restart `wazuh-manager`:

```xml
<integration>
  <name>custom-n8n.py</name>
  <hook_url>https://<YOUR-TUNNEL-SUBDOMAIN>://webhook/wazuh-alerts</hook_url>
  <level>10</level>
  <alert_format>json</alert_format>
</integration>
```

### 3. Configure Least-Privilege Containment on Ubuntu Agent
Allow the automation service account on the monitored Ubuntu endpoint to run `ufw` without storing passwords in n8n:

```bash
echo "<SERVICE_USER> ALL=(ALL) NOPASSWD: /usr/sbin/ufw" | sudo tee /etc/sudoers.d/n8n-ufw
sudo chmod 440 /etc/sudoers.d/n8n-ufw
sudo ufw allow ssh && sudo ufw --force enable
```

### 4. Import Workflow & Test Alert Simulation
1. Open n8n (`http://localhost:5678`), click **`⋮` → Import**, load `workflow/ai-soc-analyst-soar.json`, and configure credentials for **Google Gemini**, **Telegram**, and **Ubuntu Agent SSH**.
2. Simulate a Wazuh SSH brute-force alert (`rule.id: 5712`) via `curl`:

```bash
curl -X POST "https://<YOUR-TUNNEL-SUBDOMAIN>://webhook-test/wazuh-alerts" \
  -H "Content-Type: application/json" \
  -d '{
    "timestamp": "2026-11-20T18:51:00.000+0000",
    "rule": {
      "level": 10,
      "description": "sshd: brute force trying to get access to the system.",
      "id": "5712"
    },
    "agent": {
      "id": "001",
      "name": "ubuntu-agent"
    },
    "data": {
      "srcip": "185.220.101.45",
      "dstuser": "root"
    },
    "full_log": "Nov 20 18:51:00 ubuntu-agent sshd: Failed password for root from 185.220.101.45 port 54322 ssh2"
  }'
```

3. Tap **`🚫 Block IP`** in Telegram and verify active containment on the Ubuntu endpoint (`sudo ufw status`).

---

## Screenshots

### n8n SOAR Workflow Canvas
![n8n Workflow](screenshots/n8n-workflow-canvas.png)

### Telegram AI Triage & Approval Card
![Telegram Alert](screenshots/telegram-approval-prompt.png)

### Ubuntu Endpoint Firewall Containment
![UFW Block](screenshots/ubuntu-ufw-containment.png)

---

## Future Improvements
* **Threat Intelligence Enrichment:** Query VirusTotal or AbuseIPDB APIs prior to LLM triage to append historical IP reputation scores.
* **Private IP Allowlist Guardrail:** Add a pre-containment `If` node preventing accidental firewall blocks on RFC1918 internal subnets (`10.0.0.0/8`, `192.168.0.0/16`).
* **Timed Auto-Unblock:** Schedule a follow-up `Wait` node to remove temporary firewall blocks after 24 hours.
