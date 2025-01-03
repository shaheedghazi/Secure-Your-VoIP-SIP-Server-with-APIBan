# 🛡️ APIBan Integration Guide for VoIP/SIP Server Security

<div align="center">

![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)
![License](https://img.shields.io/badge/license-GPLv2-green.svg)
![Platform](https://img.shields.io/badge/platform-Linux-lightgrey.svg)

Enhance your VoIP/SIP server security with automated threat blocking using APIBan + CrowdSec integration.

**Author**: Shaheed | **Last Updated**: 2024

</div>

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
- Minimum system requirements:
  - RAM: 512MB (1GB recommended)
  - CPU: 1 core
  - Storage: 100MB free space

## 📥 Installation Steps

### 1. Install APIBan Client
```bash
# Download the latest APIBan client
wget https://github.com/apiban/apiban-client-go/releases/download/v1.0.0/apiban-iptables-client

# Make executable and move to system path
chmod +x apiban-iptables-client
sudo mv apiban-iptables-client /usr/local/bin/apiban-iptables

# Verify installation
apiban-iptables --version
```

### 2. Configure APIBan
```bash
# Create configuration directory
sudo mkdir -p /etc/apiban

# Create and edit configuration file
sudo nano /etc/apiban/apiban-iptables.conf
```

Add the following configuration:
```json
{
    "apikey": "YOUR_API_KEY",  // Replace with your actual API key from APIBan
    "lkid": "100",             // List ID for tracking last known blocked IPs
    "version": "1.0",          // API version (current stable)
    "set": "sip",              // IP set name for iptables
    "flush": "200",            // Number of IPs to remove in each flush operation
    "debug": false,            // Enable for troubleshooting
    "ipv6": false             // Enable for IPv6 support
}
```

### 3. Set Up Logging
```bash
# Create log file with appropriate permissions
sudo touch /var/log/apiban-client.log
sudo chmod 644 /var/log/apiban-client.log

# Optional: Configure log rotation
sudo tee /etc/logrotate.d/apiban << EOF
/var/log/apiban-client.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
}
EOF
```

### 4. Test the Installation
```bash
# Run APIBan manually
apiban-iptables --config /etc/apiban/apiban-iptables.conf

# Check logs
tail -f /var/log/apiban-client.log

# Verify iptables rules
sudo iptables -L -n | grep "sip"
```

### 5. Automate with Cron
Add to root's crontab (`sudo crontab -e`):
```bash
# Update APIBan blocklist every 5 minutes
*/5 * * * * /usr/local/bin/apiban-iptables --config /etc/apiban/apiban-iptables.conf >> /var/log/apiban-client.log 2>&1

# Optional: Daily cleanup of old blocks (adjust retention period as needed)
0 0 * * * /usr/local/bin/apiban-iptables --flush >> /var/log/apiban-client.log 2>&1
```

## 🛡️ Enhanced Security with CrowdSec
Create a script to sync blocked IPs with CrowdSec:

```bash
#!/bin/bash

# Configuration
CROWDSEC_LOGIN="your_crowdsec_login"
CROWDSEC_KEY="your_crowdsec_api_key"
LOG_FILE="/var/log/apiban-crowdsec.log"
BLOCK_DURATION="24h"  # Adjust block duration as needed

# Logging function
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Get blocked IPs from iptables
log "Starting APIBan to CrowdSec sync"
blocked_ips=$(sudo iptables -L -n -v | grep 'reject-with icmp-port-unreachable' | awk '{print $8}' | sort -u)

# Exit if no IPs found
if [ -z "$blocked_ips" ]; then
    log "No blocked IPs found"
    exit 1
fi

# Add IPs to CrowdSec
for ip in $blocked_ips; do
    # Skip invalid IPs
    if [[ "$ip" =~ ^(0\.0\.0\.0|255\.255\.255\.255)$ ]]; then
        log "Skipping invalid IP: $ip"
        continue
    fi
    
    log "Processing IP: $ip"
    if sudo cscli decisions add --ip "$ip" --duration "$BLOCK_DURATION" --reason "APIBan Block" > /dev/null 2>&1; then
        log "Successfully blocked IP: $ip"
    else
        log "Failed to block IP: $ip"
    fi
done

log "Sync completed"
```

Save as `/usr/local/bin/apiban2crowdsec.sh` and configure:
```bash
# Make executable
sudo chmod +x /usr/local/bin/apiban2crowdsec.sh

# Add to crontab (runs every 15 minutes)
*/15 * * * * /usr/local/bin/apiban2crowdsec.sh
```

## 📊 Monitoring and Maintenance

### System Status Checks
```bash
# View current blocks
sudo iptables -L -n

# Monitor logs
tail -f /var/log/apiban-client.log

# Check CrowdSec decisions
sudo cscli decisions list

# View system resource usage
top -p $(pgrep -f apiban-iptables)
```

### Expected Log Entries
```
[INFO] Downloaded 150 new IPs
[INFO] Updated iptables with 150 new entries
[INFO] Removed 25 expired entries
```

### Performance Metrics
Monitor these key metrics:
- Number of blocked IPs
- System resource usage
- Log file size
- Response time for SIP registration

## 🔧 Troubleshooting
Common issues and solutions:

1. **APIBan not updating:**
   ```bash
   # Check API connectivity
   curl -I https://api.apiban.org/v1/check
   ```

2. **High resource usage:**
   ```bash
   # Optimize iptables rules
   sudo iptables-save | grep -v "sip" | sudo iptables-restore
   ```

3. **Log file growing too large:**
   ```bash
   # Implement log rotation
   sudo logrotate -f /etc/logrotate.d/apiban
   ```

## ⭐ Best Practices
1. **Regular Maintenance**
   - Monitor log files weekly
   - Update APIBan client monthly
   - Review blocked IPs quarterly

2. **Security Hardening**
   - Use strong passwords
   - Enable fail2ban
   - Keep system updated
   - Regular security audits

3. **Backup Strategy**
   - Backup configuration files
   - Document custom rules
   - Maintain IP whitelist

## 📧 Email Notifications
Configure email alerts for:
- Client failures
- Missing IP blocks
- Cron job status
- High block rates
- System resource alerts

Example script for email alerts:
```bash
#!/bin/bash
MAILTO="admin@yourdomain.com"
THRESHOLD=1000  # Alert if blocked IPs exceed this number

blocked_count=$(sudo iptables -L -n | grep "sip" | wc -l)
if [ $blocked_count -gt $THRESHOLD ]; then
    echo "Alert: High number of blocked IPs ($blocked_count)" | \
    mail -s "APIBan Alert" $MAILTO
fi
```

## 📜 License
Licensed under GPLv2. See [LICENSE](LICENSE) for details.

---
<div align="center">

**Made with ❤️ by the VoIP Security Community**

[Report Issues](https://github.com/yourusername/apiban-integration/issues) | [Contribute](CONTRIBUTING.md) | [Documentation](https://apiban.org/doc)

</div>
