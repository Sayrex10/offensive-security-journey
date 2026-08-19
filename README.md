# Sayrex01 - Offensive Security Journey
## 🔐 Security Audits, Hardening & Continuous Learning

This repository also serves as a practical security engineering and learning environment, documenting hands-on work across **Linux administration, infrastructure security, network analysis, vulnerability assessment, and system hardening**.

Rather than treating security as a checklist, I use the repository to document the process of identifying weaknesses, understanding their root causes, applying mitigations, and validating the results.

### Security & Audit Work

Areas explored include:

* **Linux security auditing** — reviewing services, permissions, users, processes, exposed ports, SSH configuration, and system configuration.
* **Network security** — analyzing traffic, services, attack surfaces, and network exposure using tools such as Nmap and Wireshark.
* **Web security testing** — studying common web application vulnerabilities and testing defensive controls in authorized lab environments.
* **Infrastructure hardening** — reducing unnecessary attack surface, strengthening authentication, restricting exposed services, and applying least-privilege principles.
* **Vulnerability assessment** — identifying weaknesses, evaluating their potential impact, and documenting remediation approaches.
* **Security configuration review** — examining configurations for common weaknesses and improving defensive posture.
* **Incident-oriented analysis** — investigating suspicious behavior, logs, processes, network activity, and indicators of compromise in controlled environments.

### 🧪 Learning Through Practice

A major part of this repository is deliberately hands-on.

I use practical labs, deliberately vulnerable environments, CTF-style challenges, and controlled test systems to turn theoretical concepts into repeatable technical skills.

Topics studied include:

```text
Linux & System Administration
├── SSH & Authentication
├── Users, Groups & Permissions
├── Processes & Services
├── Networking
└── System Hardening

Cybersecurity
├── Reconnaissance
├── Vulnerability Assessment
├── Web Security
├── Network Analysis
├── Privilege & Access Control
└── Defensive Hardening

Security Engineering
├── Attack Surface Reduction
├── Logging & Monitoring
├── Secrets Management
├── Secure Configuration
└── Security Validation
```

### 📋 Security Review Methodology

For systems and projects where security is relevant, I generally approach an assessment as:

**1. Discover → 2. Enumerate → 3. Assess → 4. Exploit in a controlled environment → 5. Remediate → 6. Validate → 7. Document**

The objective is not simply to identify that something is vulnerable, but to understand **why it is vulnerable, what the realistic impact is, how it can be mitigated, and whether the mitigation actually works**.

### 🛡️ Repository Security

Where applicable, repositories are evaluated against common secure-development practices including:

* Secret exposure and credential handling
* Dependency vulnerabilities
* Access control and permissions
* Unsafe configuration
* Input validation
* Authentication and authorization
* Dependency and supply-chain risk
* Secure handling of sensitive information

For public repositories, GitHub provides security capabilities such as **Dependabot alerts, secret scanning, push protection, and code scanning** to help identify and prevent common security issues.

### 📚 Learning Log

This repository is continuously evolving.

New experiments, security findings, configurations, notes, and lessons learned are added as I continue developing my skills in:

> **Linux → Networking → Infrastructure → Security → Automation → Security Engineering**

The goal is to maintain a record of **what was tested, what failed, what was learned, and how the system was improved** — not just a collection of completed tutorials.

> ⚠️ **Ethical Use:** Security testing documented here is performed only against systems I own, explicitly authorized environments, or intentionally vulnerable training platforms. No unauthorized systems are targeted.
