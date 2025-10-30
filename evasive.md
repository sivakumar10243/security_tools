
# 1) Install packages

```bash
apt update
apt install -y libapache2-mod-evasive apache2-utils curl
```

**Why:** installs the `mod_evasive` module and `ab` (apachebench) for testing; `curl` is used by webhook scripts.

---

# 2) Create log directory for mod_evasive

```bash
mkdir -p /var/log/mod_evasive
chown -R www-data:adm /var/log/mod_evasive
chmod 750 /var/log/mod_evasive
```

**Why:** `mod_evasive` writes small marker files `/var/log/mod_evasive/dos-<IP>`. We make the directory owned by `www-data:adm` so Apache can create files; group `adm` allows controlled access. `750` keeps it private.

---

# 3) Create the mod_evasive config (explain each line)

Create `/etc/apache2/conf-available/mod_evasive.conf`:

```bash
cat > /etc/apache2/conf-available/mod_evasive.conf <<'EOF'
<IfModule mod_evasive20.c>
    DOSHashTableSize    3097
    DOSPageCount        5
    DOSPageInterval     1
    DOSSiteCount        10
    DOSSiteInterval     1
    DOSBlockingPeriod   300
    DOSLogDir           "/var/log/mod_evasive"
    DOSSystemCommand    "/usr/local/bin/mod_evasive_webhook.sh %s"
    DOSEmailNotify      root@localhost
    DOSWhitelist        127.0.0.1
    DOSWhitelist        10.0.0.0/8
    DOSWhitelist        172.16.0.0/12
    DOSWhitelist        192.168.0.0/16
</IfModule>
EOF
```

### Explanation — each directive

* `DOSHashTableSize 3097`
  Size of internal tracking table. Larger if many distinct IPs; `3097` is fine for moderate traffic.

* `DOSPageCount 5`
  **Block threshold for the same page**: if a single IP requests the *same URL* more than 5 times within `DOSPageInterval` seconds, it triggers a block.

* `DOSPageInterval 1`
  Time window in seconds for `DOSPageCount`. Here, 1 second.

* `DOSSiteCount 10`
  **Block threshold across the site**: if a single IP makes more than 10 requests to any endpoints within `DOSSiteInterval` seconds, it triggers a block.

* `DOSSiteInterval 1`
  Time window in seconds for `DOSSiteCount`.

* `DOSBlockingPeriod 300`
  Block duration in seconds (here 300s = 5 minutes). NOTE: *mod_evasive does not delete the marker file automatically when time passes; the file stays until the IP next requests the site (lazy cleanup).* We'll add a cron cleanup to notify and delete.

* `DOSLogDir "/var/log/mod_evasive"`
  Where `dos-<IP>` marker files are stored.

* `DOSSystemCommand "/usr/local/bin/mod_evasive_webhook.sh %s"`
  Runs this command when an IP is blocked. `%s` is replaced by the blocked IP. This is how we send a webhook for block alerts.

* `DOSEmailNotify root@localhost`
  Optional email fallback — mod_evasive tries to send email if available. We keep it harmless (local root).

* `DOSWhitelist ...` lines
  Whitelist private/local IP ranges (loopback & RFC1918 ranges). These **prevent internal/private IPs from being blocked**. Add any extra trusted public IPs you want to protect (monitoring or load-test hosts).

---

# 4) Enable config and module, reload Apache

```bash
a2enmod evasive
a2enconf mod_evasive
apachectl configtest
systemctl reload apache2
```

**Why:** load and enable the module and conf, test syntax, and apply changes.

---

# 5) Create the block page that shows countdown + auto-refresh

Create `/var/www/faveo/blocked.php` (adjust path to your DocumentRoot if different):

```bash
cat > /var/www/faveo/blocked.php <<'EOF'
<?php
$logdir = '/var/log/mod_evasive';
$blocking_period = 300; // must match DOSBlockingPeriod
$clientIp = $_SERVER['REMOTE_ADDR'] ?? '';
$safeIp = preg_replace('/[^0-9a-fA-F:\.]/','',$clientIp);
$dos_file = $logdir . '/dos-' . $safeIp;
$blocked = false; $remaining = 0; $blocked_at = null;
if ($safeIp !== '' && file_exists($dos_file) && is_readable($dos_file)) {
  $mtime = @filemtime($dos_file);
  if ($mtime !== false) {
    $blocked = true;
    $blocked_at = date('Y-m-d H:i:s', $mtime);
    $elapsed = time() - $mtime;
    $remaining = max(0, $blocking_period - $elapsed);
  }
}
http_response_code($blocked ? 403 : 200);
?>
<!doctype html><html><head><meta charset="utf-8"><title>Blocked</title>
<style>.count{padding:.3rem .6rem;border-radius:6px;color:#fff;font-weight:700}.g{background:#2ecc71}.o{background:#f39c12}.r{background:#e74c3c}</style>
</head><body>
  <h1>Access temporarily blocked</h1>
  <p>Your IP: <strong><?php echo htmlspecialchars($clientIp); ?></strong></p>
  <?php if ($blocked): ?>
    <p>Blocked at: <?php echo htmlspecialchars($blocked_at); ?></p>
    <p>Time remaining: <span id="t" class="count"><?php echo (int)$remaining;?>s</span></p>
    <script>
      var s = <?php echo (int)$remaining;?>;
      function colorClass(s){ return s<=60?'r':(s<=180?'o':'g'); }
      function human(s){ if(s<=0) return '0s'; var m=Math.floor(s/60); var sec=s%60; if(m) return m+'m '+sec+'s'; return s+'s'; }
      var el = document.getElementById('t'); el.className = 'count '+colorClass(s); el.textContent = human(s);
      var t = setInterval(function(){
        s--; if(s<=0){ el.textContent='0s'; el.className='count r'; clearInterval(t); setTimeout(function(){ location.reload(true); }, 1200); return; }
        el.textContent = human(s); el.className='count '+colorClass(s);
      },1000);
    </script>
  <?php else: ?>
    <p>You are not currently blocked.</p>
  <?php endif; ?>
</body></html>
EOF

chown www-data:www-data /var/www/faveo/blocked.php
chmod 640 /var/www/faveo/blocked.php
```

**Why:** When mod_evasive returns a 403, this page shows remaining time and auto-refreshes when countdown hits 0 — that refresh triggers Apache to allow the IP again (mod_evasive will delete the marker on that access if expired).

---

# 6) Wire ErrorDocument 403 via .htaccess (or vhost)

If using `.htaccess` in `/var/www/faveo`:

```bash
cat > /var/www/faveo/.htaccess <<'EOF'
ErrorDocument 403 /blocked.php
<Files "blocked.php">
  Require all granted
</Files>
<FilesMatch "^dos-">
  Require all denied
</FilesMatch>
EOF
chown www-data:www-data /var/www/faveo/.htaccess
chmod 644 /var/www/faveo/.htaccess
```

**Why:** ensures 403 responses show the blocked page. `AllowOverride All` must be set in vhost (you told me it is).

---

# 7) Block webhook (on block) — extract rDNS + User-Agent

Create webhook script `/usr/local/bin/mod_evasive_webhook.sh`:

```bash
cat > /usr/local/bin/mod_evasive_webhook.sh <<'EOF'
#!/usr/bin/env bash
WEBHOOK_URL="https://your.webhook.url"   # <--- replace with your webhook URL
WEBHOOK_TYPE="slack"                     # slack|discord|custom
IP="$1"
APACHE_LOG="/var/log/apache2/access.log"
[ -z "$IP" ] && exit 1

# reverse DNS
RDNS=$(host "$IP" 2>/dev/null | awk '/pointer/ {print $5}' | sed 's/\.$//')
[ -z "$RDNS" ] && RDNS=$(getent hosts "$IP" | awk '{print $2}')
[ -z "$RDNS" ] && RDNS="(no rDNS)"

# find last access.log line for this IP
LINE=$(grep -F " $IP " "$APACHE_LOG" 2>/dev/null | tail -n 1)
if [ -z "$LINE" ]; then
  for f in /var/log/apache2/access.log.* /var/log/httpd/access_log.*; do
    [ -e "$f" ] || continue
    LINE=$(grep -F " $IP " "$f" | tail -n 1)
    [ -n "$LINE" ] && break
  done
fi

UA="(not found)"; REF="(not found)"; TIME="(not found)"
if [ -n "$LINE" ]; then
  UA=$(echo "$LINE" | awk -F'"' '{print $6}')
  REF=$(echo "$LINE" | awk -F'"' '{print $4}')
  TIME=$(echo "$LINE" | sed -n 's/.*\[\(.*\)\].*/\1/p')
fi

TEXT="🚨 mod_evasive block detected\n• IP: ${IP}\n• rDNS: ${RDNS}\n• Time: ${TIME}\n• User-Agent: ${UA}\n• Referer: ${REF}\n• Server: $(hostname)"

if [ "$WEBHOOK_TYPE" = "slack" ]; then
  PAYLOAD=$(printf '{"text":"%s"}' "$(echo "$TEXT" | sed 's/"/\\"/g' | sed ':a;N;$!ba;s/\n/\\n/g')")
  curl -s -X POST -H "Content-Type: application/json" -d "$PAYLOAD" "$WEBHOOK_URL" >/dev/null 2>&1
elif [ "$WEBHOOK_TYPE" = "discord" ]; then
  PAYLOAD=$(printf '{"content":"%s"}' "$(echo "$TEXT" | sed 's/"/\\"/g' | sed ':a;N;$!ba;s/\n/\\n/g')")
  curl -s -X POST -H "Content-Type: application/json" -d "$PAYLOAD" "$WEBHOOK_URL" >/dev/null 2>&1
else
  curl -s -X POST -H "Content-Type: text/plain" --data-binary "$TEXT" "$WEBHOOK_URL" >/dev/null 2>&1
fi
exit 0
EOF
chmod 755 /usr/local/bin/mod_evasive_webhook.sh
```

**Why:** Called by `DOSSystemCommand` on block. It fetches rDNS and the last User-Agent in the access log for context and posts to webhook.

> **Important permission note:** `DOSSystemCommand` is run by Apache; the script must be readable and executable by Apache (`www-data`). The script reads `/var/log/apache2/access.log` — that file is usually readable by group `adm`. We made log dir group `adm` earlier. To let Apache read, add `www-data` to `adm` group (see step 8) OR change the script to read a trimmed log copy.

---

# 8) Allow Apache to read logs (so script can extract UA)

Add `www-data` to `adm` group:

```bash
usermod -a -G adm www-data
systemctl restart apache2
```

**Why:** allows the webhook script (run by Apache) to read `access.log`. Restart needed so group membership is effective.

**Security note:** this gives Apache group read access to many system logs — acceptable in many setups but check your security policy. If you want stricter, instead create a small rotated log snippet with limited perms and have the script read that instead.

---

# 9) Create cleanup + unblock notification script

We will have cron delete `dos-*` files older than 5 minutes and **send an unblock webhook before deleting**.

Create `/usr/local/bin/mod_evasive_cleanup.sh`:

```bash
cat > /usr/local/bin/mod_evasive_cleanup.sh <<'EOF'
#!/usr/bin/env bash
LOG_DIR="/var/log/mod_evasive"
WEBHOOK_URL="https://your.webhook.url"  # replace
for file in "$LOG_DIR"/dos-*; do
  [ -e "$file" ] || continue
  # if file mtime is older than 5 minutes, notify and remove
  if [ $(( $(date +%s) - $(stat -c %Y "$file") )) -ge 300 ]; then
    ip=$(basename "$file" | sed 's/^dos-//')
    TEXT="✅ mod_evasive Unblock Notice\n• IP: ${ip}\n• Time: $(date '+%Y-%m-%d %H:%M:%S')\n• Server: $(hostname)"
    PAYLOAD=$(printf '{"text":"%s"}' "$(echo "$TEXT" | sed 's/"/\\"/g' | sed ':a;N;$!ba;s/\n/\\n/g')")
    curl -s -X POST -H "Content-Type: application/json" -d "$PAYLOAD" "$WEBHOOK_URL" >/dev/null 2>&1
    rm -f "$file"
  fi
done
EOF
chmod 755 /usr/local/bin/mod_evasive_cleanup.sh
```

**Why:** sends an unblock webhook and then removes the file. Because `mod_evasive` will only delete a file when the IP requests again or an external cleanup deletes it, this script ensures timely unblocks and notifications.

---

# 10) Install cron job to run cleanup every minute

Create `/etc/cron.d/mod_evasive_cleanup`:

```bash
cat > /etc/cron.d/mod_evasive_cleanup <<'EOF'
* * * * * root /usr/local/bin/mod_evasive_cleanup.sh
EOF
```

**Why:** runs every minute, checks files older than 5 minutes, notifies & removes them. Adjust frequency if you prefer.

---

# 11) Test the setup — recommended flow

### a) Sanity checks on server

```bash
apachectl -M | grep evasive
ls -l /var/log/mod_evasive
tail -n 50 /var/log/apache2/error.log
```

### b) Trigger a block (from a remote tester)

From a separate machine (not lwsvm20) run:

```bash
ab -n 200 -c 100 https://webtest.faveodemo.com/
```

**Why remote:** avoids blocking your admin IP. This will almost certainly exceed the 5/10 thresholds and create `/var/log/mod_evasive/dos-<tester-ip>`.

### c) Verify block (server)

```bash
ls -l /var/log/mod_evasive
sudo bash -c 'for f in /var/log/mod_evasive/dos-*; do [ -e "$f" ] || continue; ip=${f##*/dos-}; mtime=$(stat -c %Y "$f"); echo "$ip blocked at $(date -d @$mtime) (age $(( $(date +%s) - mtime ))s)"; done'
```

You should see the blocked IP file appear.

### d) Check webhook alert

Check your webhook endpoint (Slack/Discord) — you should receive a block notification with IP and User-Agent.

### e) Wait ~5 minutes (or let cron run)

The cleanup script runs every minute and deletes files older than 300s; when it deletes, it sends the unblock webhook.

### f) Confirm unblock notification

Check webhook channel for `✅ mod_evasive Unblock Notice` for the same IP.

### g) Manual unblock (if needed)

```bash
rm -f /var/log/mod_evasive/dos-1.2.3.4
```

**Why:** immediate unblock if you want to remove block now.

---

# 12) Important operational notes & recommendations

* **Whitelist internal networks** in `DOSWhitelist` so private IPs (LAN) never get blocked. We added `10/8`, `172.16/12`, `192.168/16`.
* **Tune thresholds** for production traffic. The guide uses low thresholds for easy testing. After validating, raise `DOSPageCount` / `DOSSiteCount`.
* **Security tradeoff:** adding `www-data` to `adm` lets Apache scripts read logs — check policy. Alternative: have a root cron copy last N lines of access.log to `/var/log/mod_evasive/recent.log` with tighter perms; let the webhook script read that instead.
* **Log rotation:** rotated access logs might be used to find UA; script checks rotated files too but a central log aggregator is better for heavy production.
* **Spam prevention:** consider rate-limiting webhook calls (avoid duplicate alerts for the same IP within short windows).
* **Testing caution:** run `ab` from non-critical IPs and avoid public production sites.

---

# 13) Quick checklist (copy/paste)

If you want one block to run now, copy these commands in order (they repeat earlier steps but safe):

```bash
# install
apt update && apt install -y libapache2-mod-evasive apache2-utils curl

# create log dir
mkdir -p /var/log/mod_evasive
chown -R www-data:adm /var/log/mod_evasive
chmod 750 /var/log/mod_evasive

# enable conf (assumes you created mod_evasive.conf as above)
a2enmod evasive && a2enconf mod_evasive
apachectl configtest
systemctl reload apache2

# ensure webhooks & cleanup scripts are in place and executable
chmod 755 /usr/local/bin/mod_evasive_webhook.sh /usr/local/bin/mod_evasive_cleanup.sh

# allow apache to read logs (optional)
usermod -a -G adm www-data
systemctl restart apache2
```

--



# Test
```
ab -n 200 -c 200 https://webtest.faveodemo.com/index.php

```
