# 🛡️ APIBan Integration Guide for VoIP/SIP Server Security

<div align="center">

![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)
![License](https://img.shields.io/badge/license-GPLv2-green.svg)
![Platform](https://img.shields.io/badge/platform-Linux-lightgrey.svg)

Enhance your VoIP/SIP server security with automated threat blocking using APIBan + CrowdSec integration.

**Author**: Shaheed | **Last Updated**: 2024

</div>
**Note**: Install CrowdSec from their official source, enroll your engine, and set up a bouncer before proceeding.

---


## 📚 Why This Guide?
While open-source VoIP/SIP servers like FreePBX, FusionPBX, and Asterisk are powerful and widely used, comprehensive security guides for these platforms are surprisingly scarce. Most available resources either:
- Focus only on basic security measures
- Are outdated and don't address modern threats
- Require expensive commercial solutions
- Don't provide practical, step-by-step implementation details

This guide fills that gap by providing a **free, comprehensive, and practical** approach to securing your open-source VoIP infrastructure using APIBan and CrowdSec. It's especially valuable for:
- Small to medium-sized businesses running open-source PBX systems
- IT administrators managing VoIP infrastructure
- Organizations looking for cost-effective security solutions
- Anyone who has struggled to find detailed VoIP security implementation guides

## 📚 Table of Contents
- [Overview](#-overview)
- [Key Features](#-key-features)
- [Prerequisites](#-prerequisites)
- [Installation Steps](#-installation-steps)
  - [Install APIBan Client](#1-install-apiban-client)
  - [Configure APIBan](#2-configure-apiban)
  - [Set Up Logging](#3-set-up-logging)
  - [Test the Installation](#4-test-the-installation)
  - [Automate with Cron](#5-automate-with-cron)
- [Enhanced Security with CrowdSec](#-enhanced-security-with-crowdsec)
- [Monitoring and Maintenance](#-monitoring-and-maintenance)
- [Troubleshooting](#-troubleshooting)
- [Best Practices](#-best-practices)
- [License](#-license)

## 🔍 Overview
VoIP/SIP servers are prime targets for various cyber attacks, including:
- Toll fraud attempts
- DDoS attacks
- Brute force password attempts
- Registration hijacking
- Call interception attempts

Despite the critical nature of these threats, the open-source VoIP/SIP community has lacked comprehensive security tools and documentation. Commercial solutions exist but are often expensive and closed-source, leaving many deployments vulnerable. This is particularly concerning given that a compromised VoIP system can lead to:
- Substantial financial losses through toll fraud
- Service disruption affecting business operations
- Data breaches compromising call privacy
- Damage to business reputation
- Legal and compliance issues

APIBan helps mitigate these risks by maintaining a dynamic blocklist of known malicious IPs and implementing them at the iptables level. When combined with CrowdSec, it creates a robust security perimeter for your VoIP infrastructure. This open-source solution provides enterprise-grade protection without the enterprise-grade price tag.

## 🌟 Key Features
- **Real-time Protection**: Automatically blocks known malicious IPs
- **Community-driven**: Benefits from collective threat intelligence
- **Low Resource Usage**: Minimal impact on system performance
- **Easy Integration**: Works with popular VoIP platforms
- **Automated Updates**: Regular updates to threat database
- **CrowdSec Integration**: Enhanced threat detection and sharing

## 🔧 Prerequisites
- Linux-based VoIP/SIP server (MagnusBilling, FreePBX, etc.)
- Root/sudo access
- APIBan API key ([Get your key here](https://www.apiban.org/getkey.html))
- iptables installed and configured

## 📥 Installation Steps

### 1. Install APIBan Client
```bash
wget https://github.com/apiban/apiban-client-go/releases/download/v1.0.0/apiban-iptables-client
chmod +x apiban-iptables-client
sudo mv apiban-iptables-client /usr/local/bin/apiban-iptables
```

### 2. Configure APIBan
```bash
sudo mkdir -p /etc/apiban
sudo nano /etc/apiban/apiban-iptables.conf
```

Add the following configuration:
```json
{
    "apikey": "YOUR_API_KEY",  // Replace with your actual API key from APIBan
    "lkid": "100",             // The list ID provided by APIBan, adjust based on your needs
    "version": "1.0",          // API version (leave as is if using APIBan v1.0)
    "set": "sip",              // The type of IP set to be used for blocking, e.g., "sip"
    "flush": "200"             // Number of IPs to flush or remove (adjust as needed)
}
```

### 3. Set Up Logging
```bash
sudo touch /var/log/apiban-client.log
sudo chmod 644 /var/log/apiban-client.log
```

### 4. Test the Installation
```bash
apiban-iptables --config /etc/apiban/apiban-iptables.conf
tail -f /var/log/apiban-client.log
sudo iptables -L -n
```

### 5. Automate with Cron
Add to root's crontab (`sudo crontab -e`):
```bash
0 0 * * * /usr/local/bin/apiban-iptables --config /etc/apiban/apiban-iptables.conf >> /var/log/apiban-client.log 2>&1
```

## 🛡️ Enhanced Security with CrowdSec
Create a script to sync blocked IPs with CrowdSec (`nano /usr/local/bin/apiban2crowdsec.sh`):
get your creds by running
```
cat /etc/crowdsec/local_api_credentials.yaml
```
```bash
#!/bin/bash

# Define the CrowdSec CLI login (use the correct user or system privileges)
LOGIN=""  # Replace with your CrowdSec login (if necessary)
PASSWORD=""  # Replace with your password/API key

# Fetch IPs from iptables that were blocked with 'reject-with icmp-port-unreachable'
blocked_ips=$(sudo iptables -L -n -v | grep 'reject-with icmp-port-unreachable' | awk '{print $8}' | sort -u)

# Check if any IPs were found
if [ -z "$blocked_ips" ]; then
  echo "No blocked IPs found."
  exit 1
fi

# Loop through each blocked IP and add it to CrowdSec's decision list
for ip in $blocked_ips; do
  # Skip invalid IPs (e.g., 0.0.0.0, 255.255.255.255)
  if [[ "$ip" == "0.0.0.0" || "$ip" == "255.255.255.255" ]]; then
    echo "Skipping invalid IP: $ip"
    continue
  fi

  echo "Blocking IP: $ip"

  # Call CrowdSec CLI to add the IP to the decision list (ban for 1 hour)
  sudo cscli decisions add --ip "$ip" --duration 1h --reason "Blocked by iptables (APIBan)"

  if [ $? -eq 0 ]; then
    echo "Successfully added IP $ip to CrowdSec decision list."
  else
    echo "Failed to add IP $ip to CrowdSec decision list."
  fi
done

```

Save this script as `/usr/local/bin/apiban2crowdsec.sh` and make it executable:
```bash
sudo chmod +x /usr/local/bin/apiban2crowdsec.sh
```

Add a cron job to run the script every 15 minutes:
```bash
# Add to root's crontab (sudo crontab -e)
0 0 * * * /usr/local/bin/apiban2crowdsec.sh >> /var/log/apiban-crowdsec.log 2>&1
```

## 📊 Monitoring and Maintenance

### System Status Checks
```bash
# View current blocks
sudo iptables -L -n

# Monitor logs
tail -f /var/log/apiban-client.log
```

### Expected Log Entries
```
Downloaded X new IPs
Updated iptables with Y new entries
```

## 📧 Email Notifications
Configure email alerts for:
- Client failures
- Missing IP blocks
- Cron job status

## 📜 License
Licensed under GPLv2.

---
<div align="center">

**Made with ❤️ by shaheed**

[Report Issues](https://github.com/shaheed/apiban-integration/issues) | [Contribute](CONTRIBUTING.md) | [Documentation](https://apiban.org/doc)

</div>
