# Secure-Your-VoIP-SIP-Server-with-APIBan
Secure your VoIP/SIP server (MagnusBilling, FreePBX, etc.) from malicious attacks with APIBan 🔐🚫

**Author**: Shaheed
Apologies for the confusion. Here's the corrected version with everything in a single code window:


## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Installation Guide](#installation-guide)
  - [Install APIBan Client](#install-apiban-client)
  - [Create Configuration File](#create-configuration-file)
  - [Set Up Logging](#set-up-logging)
  - [Run APIBan Manually](#run-apiban-manually)
  - [Cron Job Setup](#cron-job-setup)
- [Continuous Operation Verification](#continuous-operation-verification)
- [Optional: Email Alerts](#optional-email-alerts)
- [License](#license)

## Overview
VoIP/SIP servers, like MagnusBilling, FreePBX, and others, are prime targets for attacks due to the nature of their communication protocols. To protect your system, **APIBan** is a tool that can help mitigate these risks by downloading a regularly updated list of known malicious IPs and blocking them at the iptables level.

This setup guide will walk you through installing and configuring APIBan on your server to block malicious IPs and improve your security posture.

## Prerequisites
Before you start, ensure you have the following:

- A Linux-based server running VoIP/SIP software (e.g., MagnusBilling, FreePBX, etc.).
- Root/sudo access to the server.
- An APIBan API key, which you can obtain from the APIBan website.
  https://www.apiban.org/getkey.html

## Installation Guide

### Install APIBan Client
1. **Download the APIBan iptables client**:
   ```bash
   wget https://github.com/apiban/apiban-client-go/releases/download/v1.0.0/apiban-iptables-client
   ```

2. **Make it executable**:
   ```bash
   chmod +x apiban-iptables-client
   ```

3. **Move the client to a system path**:
   ```bash
   sudo mv apiban-iptables-client /usr/local/bin/apiban-iptables
   ```

### Create Configuration File
1. **Create the configuration directory**:
   ```bash
   sudo mkdir -p /etc/apiban
   ```

2. **Create the JSON configuration file**:
   ```bash
   sudo nano /etc/apiban/apiban-iptables.conf
   ```

3. **Add the following content (replace the API key with your own)**:

```json
{
    "apikey": "YOUR_API_KEY",  // Replace with your actual API key from APIBan
    "lkid": "100",             // The list ID provided by APIBan, adjust based on your needs
    "version": "1.0",          // API version (leave as is if using APIBan v1.0)
    "set": "sip",              // The type of IP set to be used for blocking, e.g., "sip"
    "flush": "200"             // Number of IPs to flush or remove (adjust as needed)
}
```



### Set Up Logging
1. **Ensure the log file exists and is writable**:
   ```bash
   sudo touch /var/log/apiban-client.log
   sudo chmod 644 /var/log/apiban-client.log
   ```

### Run APIBan Manually
1. **Test the client**:
   ```bash
   apiban-iptables --config /etc/apiban/apiban-iptables.conf
   ```

2. **Check the log file to ensure no errors**:
   ```bash
   tail -f /var/log/apiban-client.log
   ```

3. **Verify iptables rules**:
   ```bash
   sudo iptables -L -n
   ```

   This should show entries for blocked IP addresses.

### Cron Job Setup
To ensure that APIBan runs periodically, follow these steps to set up a cron job:

1. **Edit the crontab for the root user**:
   ```bash
   sudo crontab -e
   ```

2. **Add the following line to run APIBan every 5 minutes**:
   ```bash
   */5 * * * * /usr/local/bin/apiban-iptables --config /etc/apiban/apiban-iptables.conf >> /var/log/apiban-client.log 2>&1
   ```

3. **Save and exit the crontab editor**.

This will ensure that APIBan runs every 5 minutes, updating the iptables rules with new malicious IPs.

## Continuous Operation Verification
Once the system is set up, follow these steps to ensure continuous protection:

1. **Check iptables rules periodically**:
   ```bash
   sudo iptables -L -n
   ```

2. **Monitor the log file for updates**:
   ```bash
   tail -f /var/log/apiban-client.log
   ```

   Look for log entries like:
   ```
   Downloaded X new IPs
   Updated iptables with Y new entries
   ```

## Optional: Email Alerts
To be notified of issues such as client failures or missing IP blocks, consider setting up email alerts based on log file monitoring or the cron job status.

## License
This project is licensed under the **GPLv2** License.
