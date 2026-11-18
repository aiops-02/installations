# Blackbox Exporter — Manual Install on the Monitoring VM

**Lab:** Expense app (ALB → frontend → backend → MySQL) — domain `daws90s.shop`, app URL `https://expense.daws90s.shop`
**Where:** the **monitoring VM** (same VM as Prometheus, inside the expense VPC)
**Layout convention:** binary in `/opt/blackbox_exporter-<version>` + symlink `/opt/blackbox_exporter`, config in `/etc/blackbox_exporter/`

---

## 0. What we are building

| Check | Module | Target | Proves |
|---|---|---|---|
| App is up over HTTPS, cert valid | `https_2xx` | `https://expense.daws90s.shop` | Users can load the app; cert valid & matches hostname |
| HTTP redirects to HTTPS | `http_redirects_to_https` | `http://expense.daws90s.shop` | No plain-HTTP serving |
| DNS from users' view | `dns_expense_a` | `8.8.8.8` | Public can resolve the app |
| DNS from app VMs' view | `dns_*_a` | `169.254.169.253` (Amazon VPC DNS) | VMs can resolve their dependencies |
| Zone serial | `dns_soa` | Route 53 authoritative NS | Detects zone changes |

**How it works (one line):** Prometheus calls
`localhost:9115/probe?target=<what>&module=<how>` every 15s;
`blackbox.yml` defines the *how* (modules), `prometheus.yml` defines the *what* (targets).

**Prerequisites**
- Prometheus already running on this VM at `/opt/prometheus` with `--web.enable-lifecycle`
- ALB has a 443 listener with an ACM cert covering `expense.daws90s.shop`

---

## 1. Install tools

```bash
sudo dnf install -y bind-utils   # dig
```

## 2. Download the Linux binary

Check the latest version at https://github.com/prometheus/blackbox_exporter/releases

```bash
VERSION=0.28.0
ARCH=$(uname -m); [ "$ARCH" = "x86_64" ] && ARCH=amd64; [ "$ARCH" = "aarch64" ] && ARCH=arm64

cd /tmp
curl -fLO https://github.com/prometheus/blackbox_exporter/releases/download/v${VERSION}/blackbox_exporter-${VERSION}.linux-${ARCH}.tar.gz
tar xzf blackbox_exporter-${VERSION}.linux-${ARCH}.tar.gz
sudo mv blackbox_exporter-${VERSION}.linux-${ARCH} /opt/blackbox_exporter-${VERSION}
sudo ln -sfn /opt/blackbox_exporter-${VERSION} /opt/blackbox_exporter

file /opt/blackbox_exporter/blackbox_exporter    # must say "ELF ... x86-64" — NOT Mach-O (darwin)
/opt/blackbox_exporter/blackbox_exporter --version
```

## 3. Create user and config directory

```bash
sudo useradd --system --no-create-home --shell /sbin/nologin blackbox 2>/dev/null || true
sudo mkdir -p /etc/blackbox_exporter
sudo chown blackbox:blackbox /etc/blackbox_exporter
```

## 4. Create the systemd service

```bash
sudo tee /etc/systemd/system/blackbox_exporter.service > /dev/null <<'EOF'
[Unit]
Description=Prometheus Blackbox Exporter
After=network-online.target
Wants=network-online.target

[Service]
User=blackbox
Group=blackbox
ExecStart=/opt/blackbox_exporter/blackbox_exporter \
  --config.file=/etc/blackbox_exporter/blackbox.yml \
  --web.listen-address=:9115
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
```

(Don't start it yet — the config is written in step 5.)

---

## 5. Write `/etc/blackbox_exporter/blackbox.yml` (the HOW)

```bash
sudo tee /etc/blackbox_exporter/blackbox.yml > /dev/null <<'EOF'
modules:

  # ---------- HTTP / HTTPS ----------
  http_2xx:
    prober: http
    timeout: 5s
    http:
      preferred_ip_protocol: ip4       # VPC has no IPv6
      follow_redirects: true

  https_2xx:                           # HTTPS + valid cert + hostname match
    prober: http
    timeout: 5s
    http:
      preferred_ip_protocol: ip4
      follow_redirects: true
      fail_if_not_ssl: true
      tls_config:
        insecure_skip_verify: false

  http_redirects_to_https:             # http:// must 301/302/308 to https://
    prober: http
    timeout: 5s
    http:
      preferred_ip_protocol: ip4
      follow_redirects: false
      valid_status_codes: [301, 302, 308]
      fail_if_header_not_matches:
        - header: Location
          regexp: '^https://'

  # ---------- DNS (target = resolver to ask, query_name = record to check) ----------
  dns_expense_a:
    prober: dns
    timeout: 5s
    dns:
      preferred_ip_protocol: ip4
      query_name: expense.daws90s.shop
      query_type: A
      valid_rcodes: [NOERROR]
      validate_answer_rrs:
        fail_if_not_matches_regexp: ['.*\tIN\tA\t.*']

  dns_frontend_a:
    prober: dns
    timeout: 5s
    dns:
      preferred_ip_protocol: ip4
      query_name: frontend.daws90s.shop
      query_type: A
      valid_rcodes: [NOERROR]
      validate_answer_rrs:
        fail_if_not_matches_regexp: ['.*\tIN\tA\t.*']

  dns_backend_a:
    prober: dns
    timeout: 5s
    dns:
      preferred_ip_protocol: ip4
      query_name: backend.daws90s.shop
      query_type: A
      valid_rcodes: [NOERROR]
      validate_answer_rrs:
        fail_if_not_matches_regexp: ['.*\tIN\tA\t.*']

  dns_mysql_a:
    prober: dns
    timeout: 5s
    dns:
      preferred_ip_protocol: ip4
      query_name: mysql.daws90s.shop
      query_type: A
      valid_rcodes: [NOERROR]
      validate_answer_rrs:
        fail_if_not_matches_regexp: ['.*\tIN\tA\t.*']
EOF
sudo chown blackbox:blackbox /etc/blackbox_exporter/blackbox.yml

/opt/blackbox_exporter/blackbox_exporter --config.file=/etc/blackbox_exporter/blackbox.yml --config.check
sudo systemctl enable --now blackbox_exporter
systemctl status blackbox_exporter --no-pager
```

## 6. Write `/etc/prometheus/prometheus.yml` (the WHAT)

> Replaces the whole file — back up first. Assumes nginx-exporter (9113) on frontend VM and mysqld_exporter (9104) on MySQL VM — adjust if different.

```bash
sudo cp /etc/prometheus/prometheus.yml /etc/prometheus/prometheus.yml.bak-$(date +%F-%H%M)

sudo tee /etc/prometheus/prometheus.yml > /dev/null <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s

scrape_configs:

  # ================= Self =================
  - job_name: prometheus
    static_configs:
      - targets: [localhost:9090]

  # ================= White-box: hosts =================
  - job_name: node
    static_configs:
      - targets: [frontend.daws90s.shop:9100]
        labels: { tier: frontend }
      - targets: [backend.daws90s.shop:9100]
        labels: { tier: backend }
      - targets: [mysql.daws90s.shop:9100]
        labels: { tier: mysql }

  # ================= White-box: services =================
  - job_name: nginx
    static_configs:
      - targets: [frontend.daws90s.shop:9113]
        labels: { tier: frontend }

  - job_name: backend
    metrics_path: /metrics
    static_configs:
      - targets: [backend.daws90s.shop:9101]
        labels: { tier: backend }

  - job_name: mysql
    static_configs:
      - targets: [mysql.daws90s.shop:9104]
        labels: { tier: mysql }

  # ================= Black-box: HTTPS + cert + redirect =================
  - job_name: blackbox_http
    metrics_path: /probe
    static_configs:
      - targets: [https://expense.daws90s.shop]
        labels: { probe: alb, module: https_2xx }
      - targets: [http://expense.daws90s.shop]
        labels: { probe: alb_redirect, module: http_redirects_to_https }
    relabel_configs:
      - source_labels: [__address__]       # (1) website -> ?target=
        target_label: __param_target
      - source_labels: [module]            # (2) module label -> &module=
        target_label: __param_module
      - source_labels: [__param_target]    # (3) readable instance label
        target_label: instance
      - target_label: __address__          # (4) connect to the exporter (must be LAST)
        replacement: localhost:9115

  # ================= Black-box: DNS =================
  - job_name: blackbox_dns
    metrics_path: /probe
    static_configs:
      # Public resolver — what users see
      - targets: [8.8.8.8]
        labels: { resolver: public, module: dns_expense_a, record: expense.daws90s.shop }
      # Amazon VPC resolver — what the app VMs see
      - targets: [169.254.169.253]
        labels: { resolver: vpc, module: dns_expense_a, record: expense.daws90s.shop }
      - targets: [169.254.169.253]
        labels: { resolver: vpc, module: dns_frontend_a, record: frontend.daws90s.shop }
      - targets: [169.254.169.253]
        labels: { resolver: vpc, module: dns_backend_a, record: backend.daws90s.shop }
      - targets: [169.254.169.253]
        labels: { resolver: vpc, module: dns_mysql_a, record: mysql.daws90s.shop }
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [module]
        target_label: __param_module
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115

  # ================= Blackbox exporter's own health =================
  - job_name: blackbox_exporter
    static_configs:
      - targets: [localhost:9115]
EOF
```

## 7. Fill in the Route 53 name server, validate, reload

```bash
NS=$(dig NS daws90s.shop +short | head -1 | sed 's/\.$//')
echo "Authoritative NS: $NS"
sudo sed -i "s/AUTH_NS_PLACEHOLDER/${NS}/" /etc/prometheus/prometheus.yml

/opt/prometheus/promtool check config /etc/prometheus/prometheus.yml \
  && curl -fsX POST localhost:9090/-/reload && echo "Prometheus reloaded OK"
```

Later changes to `blackbox.yml`: edit → `--config.check` → `sudo systemctl reload blackbox_exporter`.
Later changes to `prometheus.yml`: edit → `promtool check config` → `curl -X POST localhost:9090/-/reload`.

---

## 8. Test probes by hand

```bash
# HTTPS + cert
curl -s 'localhost:9115/probe?target=https://expense.daws90s.shop&module=https_2xx' \
  | grep -E '^probe_(success|http_status_code|ssl_earliest_cert_expiry|tls_version_info)'

# HTTP -> HTTPS redirect
curl -s 'localhost:9115/probe?target=http://expense.daws90s.shop&module=http_redirects_to_https' | grep '^probe_success'

# DNS via VPC resolver / public resolver
curl -s 'localhost:9115/probe?target=169.254.169.253&module=dns_backend_a' | grep -E '^probe_(success|dns_lookup)'
curl -s 'localhost:9115/probe?target=8.8.8.8&module=dns_expense_a'        | grep -E '^probe_(success|dns_lookup)'
```

Want `probe_success 1`. If 0 → add `&debug=true` to the URL to see the full probe log.

## 9. Verify in Prometheus

`http://<monitoring-vm>:9090/targets` — all jobs UP. Each blackbox target shows the real URL
(`localhost:9115/probe?target=...&module=...`).

```promql
# Overall
probe_success
probe_success == 0                                         # anything failing

# HTTP
probe_http_status_code{probe="alb"}
probe_duration_seconds{job="blackbox_http"}
sum by (phase) (probe_http_duration_seconds{probe="alb"})  # resolve/connect/tls/processing/transfer

# Certificate
(probe_ssl_earliest_cert_expiry - time()) / 86400                 # days to leaf cert expiry
(probe_ssl_last_chain_expiry_timestamp_seconds - time()) / 86400  # days to any chain cert expiry
probe_tls_version_info

# DNS
probe_success{job="blackbox_dns"}
probe_dns_lookup_time_seconds{job="blackbox_dns"}
probe_dns_answer_rrs{job="blackbox_dns"}
changes(probe_dns_serial[1h]) > 0                          # zone was edited
```

---

## 10. How the Prometheus blackbox job works (for students)

```
blackbox.yml     = HOW to check   (modules)
prometheus.yml   = WHAT to check  (targets) + WHEN (every 15s)
relabel_configs  = build the URL  localhost:9115/probe?target=<what>&module=<how>
```

Label walk-through for `https://expense.daws90s.shop`:

| Step | Label set | Effect on scrape URL |
|---|---|---|
| start | `__address__=https://expense.daws90s.shop`, `module=https_2xx` | would hit the website — wrong |
| (1) | `__param_target=https://expense.daws90s.shop` | adds `?target=...` |
| (2) | `__param_module=https_2xx` | adds `&module=https_2xx` |
| (3) | `instance=https://expense.daws90s.shop` | readable results (not `localhost:9115`) |
| (4) | `__address__=localhost:9115` | connect to the exporter |

Final call: `http://localhost:9115/probe?target=https://expense.daws90s.shop&module=https_2xx`
Rule (1) **must** run before (4), otherwise the exporter probes itself.

**DNS is the exception:** target = the *DNS server to ask*; the record name lives in the module's `query_name`.
`169.254.169.253` = Amazon-provided VPC DNS (link-local; same as VPC base+2, e.g. `10.0.0.2`).
Other AWS link-local IPs: `.169.254` = instance metadata, `.169.123` = time sync.

**Why on the monitoring VM:** probes must come from *outside* the app tiers — if the probe lived on the frontend VM, it would die with the thing it's watching. Prometheus is the only caller, so `localhost:9115` needs no SG rule.

---

## 11. Break drills

| Drill | Do | Expect |
|---|---|---|
| Frontend down | `sudo systemctl stop nginx` on frontend VM | ALB returns 503 → `probe_success{probe="alb"}=0`, `probe_http_status_code=503`, but `up{job="node",tier="frontend"}=1` — **only black-box sees the user impact** |
| Hostname mismatch | Probe `https://<alb-name>.elb.amazonaws.com` with `https_2xx` | `probe_success=0` — cert is for `expense.daws90s.shop`, not the ELB name |
| DNS record gone | Delete/rename `backend` record in Route 53 | `probe_success{module="dns_backend_a"}=0` (NXDOMAIN); backend VM still `up=1` |
| Zone change | Edit any record | `probe_dns_serial` increases on the authoritative probe |

Restore after each and watch recovery.

**Starter alert thresholds:** ALB probe failing 1m (critical); cert < 14 days (warning) / < 3 days (critical); DNS probe failing 2m. ACM auto-renews, but the expiry alert still catches failed renewals.

---

## 12. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Exec format error` | Downloaded darwin/wrong-arch tarball — use `linux-amd64` / `linux-arm64` |
| `alb_redirect` fails | ALB port-80 listener isn't a redirect to HTTPS — change listener or drop that target |
| `https_2xx` fails but browser works | Run with `&debug=true`; usually cert doesn't cover `expense.daws90s.shop` |
| VPC DNS probes fail | Monitoring VM not in expense VPC, or VPC "DNS resolution" disabled |
| Prometheus reload fails | `promtool check config` — usually YAML indentation |
| Target shows `AUTH_NS_PLACEHOLDER` | Step 7 `sed` not run, or `dig` returned nothing |

**Next:** Alertmanager + alert rules for the probes, then automate all of this.