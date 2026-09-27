# SOC L1 Automation Lab

A personal home-lab project that automates the first-pass triage work of a SOC (Security Operations Center) L1 analyst — using **Wazuh**, **n8n**, and **Google Gemini AI**.

Built by **Pavithra M**, final-year B.E. CSE (Cyber Security) student.

> This is a lab project, not a production tool. It was built to learn how SOC alert triage actually works, and to honestly test how far AI can go in doing that first pass. Findings and limitations are documented below.

---

## What it does

When a security alert fires on a monitored machine, this pipeline automatically:

1. **Detects** the event with Wazuh (SIEM)
2. **Enriches** the alert with n8n — pulls out severity, affected machine, MITRE ATT&CK technique, and source IP, and labels the IP as internal or external
3. **Triages** the alert with Gemini AI, acting as an L1 analyst — returns a verdict (true positive / false positive / needs investigation), a confidence level, a reason, and a recommended next step
4. **Delivers** the result by email, ready to act on — no dashboard required

```
Wazuh  →  n8n (enrich)  →  Gemini AI (triage)  →  Email alert
```

---

## Tech stack

| Component | Role |
|---|---|
| [Wazuh](https://wazuh.com/) | Open-source SIEM — detects and generates alerts |
| [n8n](https://n8n.io/) | Self-hosted workflow automation (via Docker) |
| [Google Gemini](https://ai.google.dev/) | AI model used for alert triage |
| SMTP (Gmail) | Delivers the final triage email |

Lab environment: a RockyLinux virtual machine (running the Wazuh agent) monitored by a Wazuh manager on the host machine, with n8n and Gemini handling enrichment and triage.

---

## Repository contents

| File | Description |
|---|---|
| `SOC_L1_Triage.json` | The exported n8n workflow — import directly into n8n |
| `README.md` | This file |

**Note:** the exported workflow file has placeholder values in place of the real email address and internal IP addresses used during testing. Credentials (API keys, SMTP password) are never included in n8n exports and must be configured separately when you import it.

---

## How to set it up

1. Install [Wazuh](https://documentation.wazuh.com/current/quickstart.html) (manager + agent).
2. Run n8n via Docker:
   ```bash
   docker run -d --name n8n --restart unless-stopped -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
   ```
3. Import `SOC_L1_Triage.json` into n8n.
4. Add your own credentials in n8n: a Google Gemini API key and SMTP email credentials.
5. In Wazuh's `ossec.conf`, add an integration block pointing to your n8n webhook URL:
   ```xml
   <integration>
     <name>shuffle</name>
     <hook_url>http://localhost:5678/webhook/wazuh-alert</hook_url>
     <level>5</level>
     <alert_format>json</alert_format>
   </integration>
   ```
6. Publish the workflow in n8n so it runs automatically.

---

## What I tested

I generated real alerts against my own lab machines only — no external targets:

- SSH login attempts with a fake username and repeated wrong passwords (password guessing)
- SSH login attempts with a real username, repeated wrong passwords in a loop (brute force)
- Deliberately wrong `sudo` passwords, to see how the system handled a plausible false alarm
- Harmless background noise Wazuh picked up on its own (port changes, new files)

Because every event was one I triggered myself, I knew the ground truth for each one and could score the AI's verdicts against reality.

---

## Findings

**1. The AI's accuracy depended heavily on the evidence it was given.**
Given only summarized fields (severity, technique, source IP), the AI hedged with "needs investigation" on a clear brute-force attack. Adding a single field — the raw log line — flipped the same alert to a correct "true positive, high confidence" verdict. Evidence quality drove the outcome more than anything else.

**2. On a small real-world batch, the AI was inconsistent on borderline cases.**
Across 8 automatically-processed alerts (2 real attacks, 6 harmless events):
- 2/2 real attacks were correctly flagged as true positives
- 1 false alarm: a harmless, self-inflicted mistyped `sudo` password was labeled a true positive
- 5/8 total alerts came back as "needs investigation"
- Two nearly identical mistyped-password events, a minute apart, received two *different* verdicts from the same model

**Takeaway:** automation handles the mechanical first pass well — enrichment, MITRE mapping, instant delivery — but the final judgment call on ambiguous events still needs a human who understands the environment. The model has no memory of what's "normal" here; a human analyst builds that context over time.

---

## Limitations

- Small sample size (10 tested alerts across two sessions) — not enough to generalize from
- Every "attacker" was my own laptop, so every source IP was internal; external attacker behavior was never tested
- Prompt wording, rule thresholds, and severity mapping are my own first attempt, not reviewed by anyone with professional SOC experience
- No deduplication/correlation — a single attack can trigger multiple separate alerts and emails

---

## Disclaimer

All testing was performed exclusively against my own personal virtual machines in an isolated home lab. No external systems, networks, or third parties were targeted at any point.
