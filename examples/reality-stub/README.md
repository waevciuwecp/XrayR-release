# REALITY Fallback Destination — Setup Reference

A minimal HTTPS static site served by nginx on port 443, acting as the
TLS fallback destination for a VLESS+REALITY inbound.  The page is
intentionally plain — just enough to look like a real website under
construction to any DPI classifier inspecting the TLS-terminated
content.

REALITY steals the TLS fingerprint from this nginx server but never
touches its cert — REALITY generates its own ephemeral keys internally.

## Nginx Setup

### Site config: `/etc/nginx/sites-available/reality`

```nginx
server {
    listen 443 ssl;
    server_name reality-dest.example.com;

    ssl_certificate     /etc/letsencrypt/live/reality-dest.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/reality-dest.example.com/privkey.pem;

    root /var/www/reality;
    index index.html;
}
```

### Document root

Use `/var/www/reality` — **not** `/root/reality`.  The `nginx` user
cannot traverse `/root`, which causes a 403 even if the file
permissions are correct.

```bash
mkdir -p /var/www/reality
cp index.html /var/www/reality/index.html
chown -R www-data:www-data /var/www/reality   # or nginx:nginx on RHEL
```

Enable the site and reload:

```bash
ln -s /etc/nginx/sites-available/reality /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

## Certbot / Certificate Renewal

Let's Encrypt certs expire after 90 days.  The initial cert is obtained
once:

```bash
certbot certonly --nginx -d reality-dest.example.com
```

### Automatic renewal

**Check if the systemd timer already exists** (most distro packages
include one):

```bash
systemctl list-timers | grep certbot
```

If present, just enable it:

```bash
sudo systemctl enable --now certbot.timer
```

**Otherwise, add a crontab entry.**  `certbot renew` runs twice daily
but `--post-hook` only fires if a renewal actually succeeded — no noisy
reloads every 12 hours:

```bash
(sudo crontab -l 2>/dev/null; echo '0 0,12 * * * certbot renew --quiet --post-hook "systemctl reload nginx"') | sudo crontab -
```

### Why only nginx, not XrayR?

XrayR does **not** need a restart after cert renewal.  REALITY generates
its own ephemeral keys internally — it only connects to nginx on
`127.0.0.1:443` to steal the TLS fingerprint.  Nginx is the sole cert
consumer, so `reload nginx` is sufficient.

## This directory

Only `index.html` is needed — it's the landing page served by nginx at
`/var/www/reality/index.html`.  Nothing else is required; nginx handles
TLS termination and HTTP serving entirely on its own.
