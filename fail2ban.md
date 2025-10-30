
## 🧠 What we’re doing

We’ll install and configure **Fail2ban** so it protects your **Apache site** (`webtest.faveodemo.com`) from bots or attackers who:

* hit too many **404 Not Found** pages, or
* trigger too many **403 Forbidden** errors.

Fail2ban will **watch Apache logs**, and if the same IP causes too many errors in a short time, it will **block that IP temporarily** using your firewall.

---

# 🪜 STEP 1: Install Fail2ban

```bash
sudo apt update
sudo apt install -y fail2ban
```

Check service is running:

```bash
sudo systemctl status fail2ban
```

If you see “active (running)”, you’re good.

---

# 🪜 STEP 2: Create a filter for 404 errors

This tells Fail2ban what to look for in your **access.log**.

```bash
sudo nano /etc/fail2ban/filter.d/apache-404.conf
```

Paste this:

```ini
[Definition]
# Fail regex = pattern to detect 404 lines in Apache log
failregex = ^<HOST> - - .*"(GET|POST|HEAD).*" 404
# ignore regex = nothing to ignore
ignoreregex =
```

### 🧩 Explanation:

* `[Definition]` → start of the filter.
* `failregex` → pattern Fail2ban looks for in the log.

  * `<HOST>` means “capture the IP address”.
  * The rest matches Apache lines that end with status code **404**.
* `ignoreregex` → empty = don’t ignore anything.

Save & exit → **Ctrl + O, Enter, Ctrl + X**

---

# 🪜 STEP 3: Create a filter for 403 errors

Now we’ll make one that detects **403 Forbidden** errors or “client denied by server configuration”.

```bash
sudo nano /etc/fail2ban/filter.d/apache-403.conf
```

Paste this:

```ini
[Definition]
# Matches 403 status codes in access.log
failregex = ^<HOST> - - .*"(GET|POST|HEAD).*" 403
# Matches "client denied" messages in error.log
failregex += \[client <HOST>:\d+\].*client denied by server configuration
ignoreregex =
```

### 🧩 Explanation:

* First `failregex` → matches any request that ended with **403** in access.log.
* Second line (`failregex += ...`) → also matches entries in error.log like:

  ```
  [client 192.168.1.10:54213] client denied by server configuration
  ```
* `ignoreregex` → nothing ignored.

Save & exit (Ctrl + O, Enter, Ctrl + X)

---

# 🪜 STEP 4: Create the jail configuration

This is where we tell Fail2ban **how long to ban**, **how many tries are allowed**, and **which logs to watch**.

```bash
sudo nano /etc/fail2ban/jail.local
```

Paste this:

```ini
[DEFAULT]
bantime  = 3600
findtime = 120
maxretry = 5
ignoreip = 127.0.0.1/8 ::1

# Apache 404 jail
[apache-404]
enabled  = true
port     = http,https
filter   = apache-404
logpath  = /var/log/apache2/access.log
backend = polling
maxretry = 5
findtime = 120
bantime  = 3600
action   = iptables-multiport[name=Apache404, port="http,https"]

# Apache 403 jail
[apache-403]
enabled  = true
port     = http,https
filter   = apache-403
logpath  = /var/log/apache2/error.log
backend = polling
maxretry = 3
findtime = 120
bantime  = 3600
action   = iptables-multiport[name=Apache403, port="http,https"]
```

---

### 🧩 Line-by-line Explanation

#### **[DEFAULT] Section**

Applies to all jails unless overridden.

| Line                         | Meaning                                           |
| ---------------------------- | ------------------------------------------------- |
| `bantime = 3600`             | Block IP for **3600 seconds = 1 hour**            |
| `findtime = 120`             | Watch a 2-minute window for repeated bad attempts |
| `maxretry = 5`               | If 5 bad hits in 2 minutes → ban                  |
| `ignoreip = 127.0.0.1/8 ::1` | Don’t ever ban localhost or self                  |

---

#### **[apache-404]**

Protects against 404 floods (bots scanning missing pages).

| Line                                    | Meaning                                                            |
| --------------------------------------- | ------------------------------------------------------------------ |
| `enabled = true`                        | Turn this jail ON                                                  |
| `port = http,https`                     | Watch both 80 and 443 traffic                                      |
| `filter = apache-404`                   | Use the filter we made in `/etc/fail2ban/filter.d/apache-404.conf` |
| `logpath = /var/log/apache2/access.log` | File to monitor                                                    |
| `maxretry = 5`                          | Allow 5 missing pages per 2 min                                    |
| `findtime = 120`                        | Count within 120 seconds                                           |
| `bantime = 3600`                        | Ban IP for 1 hour                                                  |
| `action = iptables-multiport[...]`      | Use firewall to block both HTTP & HTTPS                            |

---

#### **[apache-403]**

Protects against forbidden access attempts.

| Line                                   | Meaning                           |
| -------------------------------------- | --------------------------------- |
| `enabled = true`                       | Turn this jail ON                 |
| `filter = apache-403`                  | Use the 403 filter we made        |
| `logpath = /var/log/apache2/error.log` | Watch Apache’s error log          |
| `maxretry = 3`                         | Stricter: only 3 tries before ban |
| `findtime = 120`                       | Time window 2 min                 |
| `bantime = 3600`                       | Ban for 1 hour                    |
| `action = iptables-multiport[...]`     | Block via firewall                |

---

# 🪜 STEP 5: Restart Fail2ban

After saving:

```bash
sudo systemctl restart fail2ban
```

---

# 🪜 STEP 6: Check Fail2ban status

List all active jails:

```bash
sudo fail2ban-client status
```

Check one jail:

```bash
sudo fail2ban-client status apache-404
```

You’ll see:

```
Status for the jail: apache-404
|- Filter
|  |- Currently failed: 0
|  `- Total failed: 7
`- Actions
   |- Currently banned: 1
   `- Banned IP list: 203.0.113.45
```

---

# 🪜 STEP 7: How blocking/unblocking works

| Action                               | What happens                                                               |
| ------------------------------------ | -------------------------------------------------------------------------- |
| IP triggers too many errors          | Fail2ban counts them                                                       |
| Count > `maxretry` within `findtime` | Fail2ban **adds a firewall rule** to drop that IP                          |
| Time passes                          | IP cannot access your site                                                 |
| After `bantime` seconds (1 hour)     | Fail2ban **automatically removes** the firewall rule                       |
| Manual unban                         | You can unban any time: `sudo fail2ban-client set apache-404 unbanip <IP>` |

---

# 🪜 STEP 8: (Optional) Test your setup

From a different IP (not localhost):

```bash
for i in {1..6}; do curl -s -o /dev/null -w "%{http_code}\n" https://webtest.faveodemo.com/nothing$i; done
```

After 5–6 hits, Fail2ban should ban your IP temporarily.
Check:

```bash
sudo fail2ban-client status apache-404
```

---

# ✅ Result

You now have:

* **Fail2ban installed**
* **404 & 403 protection active**
* Automatic **blocking** for repeated suspicious requests
* Automatic **unblocking** after ban period

---
