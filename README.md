# 🚨 Automated Wazuh SIEM Alerting Pipeline with Python Flask & Telegram

An end-to-end Security Operations Center (SOC) automation project that integrates *Wazuh SIEM, a custom **Python Flask REST API, and the **Telegram Bot API* to deliver real-time security alerts directly to mobile and desktop devices.

## 📌 Project Overview

In modern SOC environments, immediate visibility into high-severity security events (such as SSH brute-force attacks or unauthorized access attempts) is critical for reducing Mean Time to Detect (MTTD). 

This project bridges Wazuh's local alert logs (alerts.json) with instant messaging by building a lightweight middleware API and polling service that runs continuously in the background on an Ubuntu Server via systemd.

### Architecture Flow
1. *Detection (Wazuh Manager):* Monitors system logs and triggers JSON alerts in /var/ossec/logs/alerts/alerts.json.
2. *Ingestion (Flask REST API - api.py):* Exposes a local HTTP endpoint (GET /alerts) that reads and serves recent Wazuh alert events.
3. *Filtering & Enrichment (alerting.py):* Polls the Flask API every 5 seconds, filters for alerts with *Severity Level 5+*, extracts key forensic context (Rule Description, Rule ID, Targeted User, Source IP, and Raw Log), and prevents duplicate notifications.
4. *Notification (Telegram Bot API):* Pushes formatted Markdown alerts instantly to a dedicated Telegram security channel/chat.
5. *Persistence (systemd):* Both Python services are daemonized to start automatically on system boot and recover from failures.

---

## 🛠️ Tech Stack
* *SIEM:* Wazuh (Manager, Indexer, Dashboard)
* *OS:* Ubuntu Server / Desktop 24.04 LTS
* *Backend & Scripting:* Python 3, Flask, Requests
* *Service Management:* Linux systemd
* *Alerting Channel:* Telegram Bot API

---

## 📂 Repository Structure

text
wazuh-telegram-alerting/
├── api.py                      # Flask REST API serving Wazuh alerts.json
├── alerting.py                 # Polling & Telegram notification service
├── systemd/
│   ├── wazuh-api.service       # Systemd unit file for Flask API
│   └── wazuh-telegram.service  # Systemd unit file for Telegram service
├── screenshots/
│   └── wazuh-bruteforce.png    # Wazuh Dashboard showing SSH brute-force events
└── README.md


---

## 🚀 Setup & Installation

### 1. Prerequisites
* A running *Wazuh Manager* instance on Ubuntu.
* A Telegram Bot created via *@BotFather* (Bot Token) and your *Chat ID* (via *@userinfobot*).
* Python 3 with flask and requests installed:
  bash
  sudo apt update && sudo apt install -y python3-flask python3-requests
  

### 2. Configure the Scripts
1. Clone this repository to your Wazuh server:
   bash
   git clone https://github.com
   cd wazuh-telegram-alerting

3. Open alerting.py and replace YOUR_BOT_TOKEN and YOUR_CHAT_ID with your Telegram credentials.
   
   ### Python code for alertin.py

import time
import requests

FLASK_API_URL = "http://localhost:5000/alerts"
BOT_TOKEN = "YOUR_BOT_TOKEN"
CHAT_ID = "YOUR_CHAT_ID"

seen_alerts = set()

def send_telegram_message(text):
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
    payload = {"chat_id": CHAT_ID, "text": text, "parse_mode": "Markdown"}
    requests.post(url, json=payload)

print("Monitoring Wazuh alerts via Flask API...")

while True:
    try:
        response = requests.get(FLASK_API_URL)
        if response.status_code == 200:
            alerts = response.json()
            for alert in alerts:
                alert_id = alert.get("id")
                rule = alert.get("rule", {})
                level = rule.get("level", 0)

                if alert_id and alert_id not in seen_alerts:
                    seen_alerts.add(alert_id)
                    if level >= 5:
                        data = alert.get("data", {})
                        src_ip = data.get("srcip") or data.get("src_ip") or data.get("remote_ip") or "Local / N/A"
                        user = data.get("dstuser") or data.get("srcuser") or "N/A"
                        full_log = alert.get("full_log", "N/A")

                        msg = (
                            f"🚨 Wazuh Security Alert\n"
                            f"Rule: {rule.get('description')} (ID: {rule.get('id')})\n"
                            f"Severity Level: {level}\n"
                            f"Agent: {alert.get('agent', {}).get('name')}\n"
                            f"User: {user}\n"
                            f"Source IP: {src_ip}\n"
                            f"Log: {full_log}"
                        )
                        send_telegram_message(msg)
    except Exception as e:
        print(f"Error polling API: {e}")

    time.sleep(5)
   

### 3. Configure systemd Background Services
Create /etc/systemd/system/wazuh-api.service:
ini
[Unit]
Description=Wazuh Flask Alerts API
After=network.target wazuh-manager.service

## Python api.py

from flask import Flask, jsonify
import json

app = Flask(_name_)
ALERTS_FILE = "/var/ossec/logs/alerts/alerts.json"

@app.route("/alerts", methods=["GET"])
def get_alerts():
    alerts = []
    try:
        with open(ALERTS_FILE, "r") as f:
            lines = f.readlines()[-20:]
            for line in lines:
                alerts.append(json.loads(line))
        return jsonify(alerts), 200
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if _name_ == "_main_":
    app.run(host="127.0.0.1", port=5000)

[Service]
User=root
WorkingDirectory=/path/to/wazuh-telegram-alerting
ExecStart=/usr/bin/python3 /path/to/wazuh-telegram-alerting/api.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target


Create /etc/systemd/system/wazuh-telegram.service:
ini
[Unit]
Description=Wazuh Telegram Alerting Service
After=network.target wazuh-api.service
Requires=wazuh-api.service

[Service]
User=YOUR_UBUNTU_USER
WorkingDirectory=/path/to/wazuh-telegram-alerting
ExecStart=/usr/bin/python3 /path/to/wazuh-telegram-alerting/alerting.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target


Enable and start both services:
bash
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-api.service wazuh-telegram.service


---

## 🧪 Attack Simulation & Verification

To validate the pipeline, an SSH brute-force / invalid-user login attack was simulated against the Ubuntu server:
bash
ssh hacker@<WAZUH_SERVER_IP>


### Results
* *Wazuh Dashboard:* Captured and indexed the authentication failures under sshd threat hunting events (Rule ID 5710 / 5763).
* *Telegram Alert:* Delivered a real-time notification within 5 seconds containing the attacker's source IP, targeted username, severity level, and raw log entry.

![Wazuh Brute Force Dashboard](screenshots/wazuh-bruteforce.png)

---

## 🔒 Security Best Practices Applied
* *Least Privilege:* Only the Flask reader service runs with elevated permissions to read /var/ossec/logs/alerts/alerts.json; the external-facing Telegram polling script runs as a standard unprivileged user.
* *Local Binding:* The Flask API binds strictly to 127.0.0.1 (localhost) to prevent unauthorized external access to raw SIEM logs.
* *Alert Fatigue Reduction:* Configured severity threshold filtering (level >= 5) to suppress noisy routine administrative events (sudo executions) while prioritizing actionable threats.
