# Docker Maintenance

Scope: DEV_ENV5 workspace (`qData/qData`, `dashboard`, `nginxmainproxy`).

## TLS Certificate Renewal

Let's Encrypt certificates expire every **90 days**. Set up a monthly cron job to auto-renew:

```bash
# Edit crontab
crontab -e

# Add this line (runs at 3am on the 1st of each month)
0 3 1 * * cd /root/REMOTEDEVELOPMENT/DEV_ENV5/qData/qData && docker compose run --rm certbot && docker compose exec nginx nginx -s reload >> /var/log/certbot-renew.log 2>&1
```

### Manual renewal

```bash
docker compose run --rm certbot
docker compose exec nginx nginx -s reload
```

If you also manage edge/domain certificates at the outer proxy, repeat in:

- `/root/REMOTEDEVELOPMENT/DEV_ENV5/nginxmainproxy`
- `/root/REMOTEDEVELOPMENT/DEV_ENV5/dashboard`

Example:

```bash
cd /root/REMOTEDEVELOPMENT/DEV_ENV5/nginxmainproxy
docker compose run --rm certbot
docker compose exec nginx nginx -s reload
```

### Check expiry date

```bash
echo | openssl s_client -connect uptoken.cloud:443 2>/dev/null | openssl x509 -noout -dates
echo | openssl s_client -connect uptoken.cloud:8443 2>/dev/null | openssl x509 -noout -dates
echo | openssl s_client -connect uptoken.cloud:50052 2>/dev/null | openssl x509 -noout -dates
```

> [!WARNING]
> If the certificate expires, HTTPS and gRPC-over-TLS will stop working. Clients will see connection errors.

---

## Disk Cleanup

Docker accumulates old images, stopped containers, and build cache over time. Run a periodic prune to reclaim disk:

```bash
# One-off cleanup (removes unused images, containers, networks)
docker system prune -f

# Also remove unused volumes (careful — this deletes data volumes not attached to any container)
docker system prune -f --volumes
```

### Automate with cron

```bash
# Weekly prune at 4am on Sundays
0 4 * * 0 docker system prune -f >> /var/log/docker-prune.log 2>&1
```

### Check current disk usage

```bash
docker system df
```

