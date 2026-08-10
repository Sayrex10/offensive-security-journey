# 📬 Linux Mail Server — OVHcloud VPS

Full production SMTP mail server deployed from scratch on a Linux VPS. Covers OS hardening, Postfix configuration, DKIM signing, SPF/DMARC DNS setup, firewall rules, and deliverability testing. Achieved a **10/10 score on mail-tester.com**.

## Stack

| Component | Tool |
|---|---|
| VPS Provider | OVHcloud |
| OS | Ubuntu 22.04 LTS |
| MTA | Postfix |
| DKIM Signing | OpenDKIM |
| Firewall | iptables |
| TLS | Let's Encrypt (Certbot) |

---

## Phase 1 — VPS Initial Hardening

```bash
# Full system update
apt update && apt upgrade -y

# Create non-root admin user
adduser eliass
usermod -aG sudo eliass

# Disable root SSH, enforce key auth only
nano /etc/ssh/sshd_config
# Set: PermitRootLogin no
# Set: PasswordAuthentication no
systemctl restart sshd

# Set correct hostname (must match MX record)
hostnamectl set-hostname mail.yourdomain.com
echo "127.0.0.1 mail.yourdomain.com" >> /etc/hosts
```

## Phase 2 — Postfix Install & Config

```bash
apt install postfix mailutils -y
# Select: Internet Site
# System mail name: yourdomain.com
```

`/etc/postfix/main.cf`:
```ini
myhostname = mail.yourdomain.com
mydomain = yourdomain.com
myorigin = $mydomain
inet_interfaces = all
inet_protocols = ipv4
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
mynetworks = 127.0.0.0/8
home_mailbox = Maildir/

# TLS (after certbot setup)
smtpd_tls_cert_file = /etc/letsencrypt/live/yourdomain.com/fullchain.pem
smtpd_tls_key_file  = /etc/letsencrypt/live/yourdomain.com/privkey.pem
smtpd_use_tls = yes
smtp_tls_security_level = may
smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
```

## Phase 3 — DKIM with OpenDKIM

```bash
apt install opendkim opendkim-tools -y

# Generate keypair
mkdir -p /etc/opendkim/keys/yourdomain.com
opendkim-genkey -s mail -d yourdomain.com -D /etc/opendkim/keys/yourdomain.com/
chown -R opendkim:opendkim /etc/opendkim/keys/
chmod 600 /etc/opendkim/keys/yourdomain.com/mail.private

# Print public key to add to DNS
cat /etc/opendkim/keys/yourdomain.com/mail.txt
```

`/etc/opendkim.conf` key settings:
```ini
Canonicalization    relaxed/simple
Mode                sv
SubDomains          no
KeyTable            refile:/etc/opendkim/KeyTable
SigningTable        refile:/etc/opendkim/SigningTable
InternalHosts       refile:/etc/opendkim/TrustedHosts
Socket              inet:12301@localhost
```

`/etc/opendkim/KeyTable`:
```
mail._domainkey.yourdomain.com yourdomain.com:mail:/etc/opendkim/keys/yourdomain.com/mail.private
```

`/etc/opendkim/SigningTable`:
```
*@yourdomain.com mail._domainkey.yourdomain.com
```

Connect Postfix → OpenDKIM by adding to `main.cf`:
```ini
milter_protocol = 6
milter_default_action = accept
smtpd_milters = inet:localhost:12301
non_smtpd_milters = inet:localhost:12301
```

## Phase 4 — DNS Records

```dns
; MX
yourdomain.com.           IN  MX   10  mail.yourdomain.com.

; SPF — only your VPS IP is authorized
yourdomain.com.           IN  TXT  "v=spf1 ip4:YOUR.VPS.IP.HERE -all"

; DKIM — paste full output of mail.txt
mail._domainkey.yourdomain.com.  IN  TXT  "v=DKIM1; k=rsa; p=YOUR_PUBKEY"

; DMARC — start with p=none for monitoring
_dmarc.yourdomain.com.   IN  TXT  "v=DMARC1; p=none; rua=mailto:admin@yourdomain.com; ruf=mailto:admin@yourdomain.com; fo=1"
```

> ⚠️ Also set **PTR / Reverse DNS** at OVHcloud panel — maps your IP back to `mail.yourdomain.com`. Critical for inbox delivery.

## Phase 5 — Firewall Rules

```bash
# Allow established sessions
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT

# SSH (change 22 to your hardened port if applicable)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Mail ports
iptables -A INPUT -p tcp --dport 25  -j ACCEPT   # SMTP
iptables -A INPUT -p tcp --dport 587 -j ACCEPT   # Submission
iptables -A INPUT -p tcp --dport 465 -j ACCEPT   # SMTPS
iptables -A INPUT -p tcp --dport 993 -j ACCEPT   # IMAPS

# Drop all other inbound
iptables -P INPUT DROP

# Persist across reboots
apt install iptables-persistent -y
iptables-save > /etc/iptables/rules.v4
```

## Testing & Verification

```bash
# Send a test email
echo "Test body" | mail -s "Test Subject" yourtest@gmail.com

# Watch mail log in real time
tail -f /var/log/mail.log

# Check for DKIM signing in headers
grep -i dkim /var/log/mail.log
```

**Deliverability checklist:**
- [x] SPF record exists and matches VPS IP
- [x] DKIM signature present and validates
- [x] DMARC policy set
- [x] PTR reverse DNS configured at OVHcloud
- [x] TLS enabled on port 587
- [x] Tested on mail-tester.com → **10/10**
- [x] Tested delivery to Gmail, Outlook — lands in inbox

## Key Lessons

- PTR record is often the #1 reason mail goes to spam — set it at OVHcloud panel before anything else
- Start DMARC with `p=none` and monitor reports for 1-2 weeks before enforcing
- `tail -f /var/log/mail.log` while sending test mails is the fastest debug loop
- Gmail's Postmaster Tools gives long-term domain reputation data — worth setting up

---

*By [Ilyass Benfdil](https://github.com/Sayrex01) · Marrakech, Morocco*
