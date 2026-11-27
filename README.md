# Security Dashboard Migration Guide

[中文版](README-zh.md)

### Introduction

This is a vibe coding project, built entirely black-box, with MuseAI and Codex as the assistants. The dashboard covers two things: website security (attacks, bans) and host security (logins, keys).

#### Demo

[https://kkii.site/dashboard](https://kkii.site/dashboard)

#### GitHub repository

[https://github.com/sunorz/security-dashboard](https://github.com/sunorz/security-dashboard)

### Fresh deployment on a new machine

1. What's in this repository

```
security-dashboard/
├── README.md                          # This guide (English) — same content as README-zh.md
├── README-zh.md                       # This guide (Chinese)
├── dashboard-init.py.template               # Generator template (the core of the dashboard) → dashboard-init.py
├── push-host-data.py.template               # Host-2 push script template → push-host-data.py
├── refresh-security-data.py.template  # 30-minute refresh script template → refresh-security-data.py
├── dashboard.cron.template            # Two cron job templates → install into cron
├── nginx-dashboard.conf.template      # nginx config: blocklist include + dashboard site server block
├── index.html.template                # Dashboard page template → index.html in __DASHBOARD_DIR__
├── .dashboard-rules.json.template     # Rules file template → .dashboard-rules.json at __RULES_PATH__
└── assets/                            # Dashboard images (6 files: goldfish, empty-tank red X, medals) → __DASHBOARD_DIR__
    ├── fish-sprite.webp
    ├── fish-zero.webp
    ├── ie-redx.png
    ├── medal-gold.png
    ├── medal-silver.png
    └── medal-bronze.png
```

Fill in every placeholder in the templates with the new host's real values (see Section 3). The files cannot be run as-is until then. The templates contain no real passwords or real paths, so they are safe to keep as they are.

The state files (`.autoban-state.json`, `.autoban-7d.json`, `.dashboard-resolutions.json`) are not in this directory: on a fresh deployment they are rebuilt automatically (see Section 8); when migrating from an old host, copy them from there. The dashboard page `index.html` itself is the `index.html.template` in this directory (see Section 5, step 3). The images are already in `assets/`; no need to copy them from anywhere else.

Note: `dashboard-init.py.template` is a snapshot of the old host's `dashboard-init.py` at a point in time. If the generator is changed later, regenerate this template (or copy the new `dashboard-init.py` and change its path constants using the table in Section 3).

2. Architecture at a glance

```
nginx access logs (one <site>.log per site) ──┐
SSH logs (/var/log/secure or auth.log) ───────┼─→ dashboard-init.py ─→ s-<key>.json ─→ index.html (static page)
fail2ban logs (optional) ─────────────────┘         │                 │
                                                     ├─→ nginx blocklist.conf (autoban; the generator runs nginx -t itself, then reloads)
                                                     └─→ daily report email (Sohu SMTP, sent with the 07:00 full run)
```

- `index.html` is a plain static page that reads `s-<key>.json` via `?key=`. With no key or a wrong key it shows demo data; `?key=0` shows the error page.
- `dashboard-init.py` is the generator and uses only the Python standard library (runs on Python 3.6+). Running it directly is the 07:00 full run: it rotates the key, regenerates the data, and sends the daily report email.
- `refresh-security-data.py` runs every 30 minutes and only refreshes `s-<key>.json`; it does not rotate the key or send email.

3. Placeholder table

Use this table when filling in the templates. Placeholders are grouped by the file they appear in. **Required** means the program needs a real value; **Optional** means you may leave the placeholder as-is — the program still runs, and that entry simply never matches (network placeholders fall back to a documentation range that carries no real traffic). If you do not have a value for an optional placeholder, leave it alone; do not invent one.

#### Dashboard generator script

**dashboard-init.py.template** (deploy this file on the host that shows the dashboard, `host1`)

| Placeholder | What it is | Required? | What to fill in |
|---|---|---|---|
| `__DASHBOARD_DIR__` | Dashboard directory (where `index.html` and `s-<key>.json` live) | Required | e.g. `/www/wwwroot/example.com/dashboard` |
| `__LOG_DIR__` | nginx access log directory | Required | Directory holding the site logs; one `<site>.log` per site, no `.error.log` |
| `__RULES_PATH__` | Rules file | Required | Actual path of the rules file (root's home directory or alongside it is fine) |
| `__BLOCKLIST_CONF__` | nginx blocklist, as the generator sees it | Required | The blocklist.conf path; must be the same file as `__BLOCKLIST_PATH__` in the nginx config below |
| `__HOST2_JSON_PATH__` | Pushed data from the other host | Optional | Only if you monitor a second host: the path of the pushed file on this host, e.g. `/home/<receiver>/host-host2.json`. If you monitor only this host, remove the `_read_host2` call instead (see Section 9) |
| `__PUSH_USER__` | Receiving account username on host 1 | Optional | The SSH user host 2 pushes data as (the receiving account). Only needed for multi-host push; leave it if you monitor only one host |
| `__HOSTSEC_STATE_PATH__` | Host security state | Required | Actual path (keep it next to the generator); rebuilt on a fresh host (see Section 8) |
| `__HOSTSEC_IP_BASELINE_PATH__` | Host IP baseline | Required | Same as above (rebuilt on a fresh host, see Section 8) |
| `__USUALLY_KEY_PATH__` | Usual key location baseline | Required | Same as above (rebuilt on a fresh host, see Section 8) |
| `__RESOLUTIONS_PATH__` | Resolved-incident registry | Required | Actual path (keep it next to the generator) |
| `__AUTOBAN_STATE_PATH__` | Ban records | Required | Actual path (keep it next to the generator) |
| `__AUTOBAN_7D_STATE_PATH__` | 7-day ban counts | Required | Same as above |
| `__SMTP_HOST__` | Mail server host | Required | SMTP server for the daily report email (the template defaults to Sohu SMTP; change it if you use another provider) |
| `__SMTP_SENDER__` | Sender mailbox | Required | Mailbox used to send the daily report email (SMTP account) |
| `__SMTP_RECEIVER__` | Receiver mailbox | Required | Mailbox that receives the daily report email |
| `__SMTP_PASSWORD__` | Sender mailbox password | Required | Password of the sender mailbox; do not leak it |
| `__HOME_IP__` | Owner's own IP | Optional | Owner's usual public IP (a single address), used as a "known source" in the host panel. Leave it if you have no fixed IP |
| `__HOME_NET__` | Owner's home network | Optional | Owner's home broadband network (CIDR), used as a "known source" for panel logins. This is the /24 of `__HOME_IP__`; if you have no fixed home IP, leave both |
| `__HOST_PUBLIC_IP__` | Host's own public IP | Required | The host's own public IP (a single address); the panel/SSH sometimes log a login from the host itself via its public IP, that is not a stranger |
| `__EGRESS_NET_1__` | Assistant egress network 1 | Optional | Network (CIDR) the assistant tools connect from; if there are several, fill them in order, `__EGRESS_NET_2__` the same way. Leave both if no assistant tools connect to this host |
| `__EGRESS_NET_2__` | Assistant egress network 2 | Optional | Same as above |
| `__LAN_NET__` | Owner's LAN network | Optional | Owner's LAN network (CIDR), the never-ban safety floor. Leave it if this host has no LAN |
| `__LAN_PREFIX__` | LAN IP prefix | Optional | Owner's LAN IP prefix (e.g. `172.32.`); the generator uses it to recognise LAN IPs and skip geolocation. Leave it if this host has no LAN |
| `__TAILSCALE_NET__` | Tailscale network | Optional | Tailscale network (CIDR). Leave it if you do not use Tailscale |
| `__ZEROTIER_NET__` | ZeroTier network | Optional | ZeroTier network (CIDR). Leave it if you do not use ZeroTier |
| `__YANG_TAILSCALE_IP__` | Yang's Tailscale IP | Optional | Tailscale IP of the Yang node; maps SSH logins from it to the "yang" key label. Leave it if there is no such node |
| `__PEPPER_TAILSCALE_IP__` | Pepper's Tailscale IP | Optional | Tailscale IP of the Pepper node; maps SSH logins from it to the "pepper" key label. Leave it if there is no such node |

#### Host-security push script for other machines

**push-host-data.py.template** (place on `host2`)

| Placeholder | What it is | Required? | What to fill in |
|---|---|---|---|
| `__PUSH_SSH_KEY__` | SSH key for pushing host data | Required | Private key on host 2 that can scp to the dashboard host (host 1); put the matching public key in host 1's receiving account |
| `__SSH_PORT__` | SSH port | Required | SSH port of the dashboard host (host 1) |
| `__HOST2_PUSH_DEST__` | Push destination | Required | Where host 2 pushes its data: `user@host1-host:filename` (scp destination on host 1, in the receiving account's home directory) |
| `__PUSH_USER__` | Receiving account username on host 1 | Optional | Same as in the generator above; leave it if you monitor only one host |
| `__HOME_NET__` | Owner's home network | Optional | Same as in the generator above; leave it if you have no fixed home IP |
| `__LAN_NET__` | Owner's LAN network | Optional | Same as in the generator above; leave it if host 2 has no LAN |
| `__LAN_PREFIX__` | LAN IP prefix | Optional | Same as in the generator above |
| `__TAILSCALE_NET__` | Tailscale network | Optional | Leave it if you do not use Tailscale |
| `__ZEROTIER_NET__` | ZeroTier network | Optional | Leave it if you do not use ZeroTier |
| `__YANG_TAILSCALE_IP__` | Yang's Tailscale IP | Optional | Leave it if there is no such node |
| `__PEPPER_TAILSCALE_IP__` | Pepper's Tailscale IP | Optional | Leave it if there is no such node |

#### Scheduled dashboard-data refresh script

**refresh-security-data.py.template** (deploy this file on the host that shows the dashboard, `host1`)

| Placeholder | What it is | Required? | What to fill in |
|---|---|---|---|
| `__DASHBOARD_DIR__` | Dashboard directory | Required | Same as in the generator above |
| `__DASHBOARD_INIT_PY__` | Generator path | Required | Actual path of `dashboard-init.py` on the new host, e.g. `/root/dashboard-init.py` |
| `__REFRESH_SCRIPT__` | This script's own path | Required | Actual path of `refresh-security-data.py` on the new host |

#### Scheduled tasks

**dashboard.cron.template** (deploy this file on the host that shows the dashboard, `host1`)

| Placeholder | What it is | Required? | What to fill in |
|---|---|---|---|
| `__SCRIPT_DIR__` | Generator directory | Required | Directory holding `dashboard-init.py` on the new host, e.g. `/root` |
| `__REFRESH_SCRIPT__` | 30-minute refresh script path | Required | Same as above |
| `__BAN_WATCH_LOG__` | 30-minute refresh log | Required | Any writable path |
| `__DAILY_LOG__` | 07:00 full-run log | Required | Any writable path |

#### Dashboard settings file

**nginx-dashboard.conf.template** (deploy this file on the host that shows the dashboard, `host1`)

| Placeholder | What it is | Required? | What to fill in |
|---|---|---|---|
| `__BLOCKLIST_PATH__` | nginx blocklist, as nginx sees it | Required | The blocklist.conf path; must be the same file as `__BLOCKLIST_CONF__` in the generator above |
| `__DASHBOARD_DOMAIN__` | Dashboard domain | Required | The new host's dashboard domain, e.g. `example.com` |
| `__SITE_ROOT__` | Root directory of the site hosting the dashboard | Required | The new host's site root (the dashboard is a subdirectory of it) |

#### Network whitelist

**.dashboard-rules.json.template** (deploy this file on the host that shows the dashboard, `host1`)

| Placeholder | What it is | Required? | What to fill in |
|---|---|---|---|
| `__LAN_NET__` | Owner's LAN network | Optional | Same as in the generator above |
| `__HOME_IP__` | Owner's own IP | Optional | Same as in the generator above; leave it if you have no fixed IP |
| `__EGRESS_NET_1__` | Assistant egress network 1 | Optional | Same as in the generator above |
| `__EGRESS_NET_2__` | Assistant egress network 2 | Optional | Same as above |
| `__TAILSCALE_NET__` | Tailscale network | Optional | Leave it if you do not use Tailscale |

### index.html.template

The only placeholder is `__SAMPLE__` (demo data). It needs no change: the demo uses `example.com` sample data and random private IPs, and contains no real addresses.

### Hardcoded paths (not placeholders)

These paths are written directly in the templates, not as placeholders. Change them in the file only if your host differs:

- SSH log: both scripts read `/var/log/secure` (RHEL family). On Debian/Ubuntu, change it to `/var/log/auth.log` in both scripts.
- Panel login database: the generator reads `/www/server/panel/data/db/log.db` (Baota). Without Baota, the code tolerates it being missing and the panel-login panel will be empty.
- nginx binary: the generator calls `/www/server/nginx/sbin/nginx`. Change `_NGINX_BIN` if nginx lives elsewhere.

The mail server port does not need to change; the template already has Sohu SMTP on port 465. If you switch to a different mail provider, change `SMTP_SERVER` / `SMTP_PORT` / `IMAP_SERVER` in the template.

### What to change in index.html.template

The sites and host names in the template are examples. Change them to your own when deploying:

- `SITE_DOTS` (site colour map): replace `example.com`, `shop.example.com`, `api.example.com`, `blog.example.com`, `docs.example.com`, `support.example.com` with the domains of the sites you monitor; keep the colours or change them.
- `SITE_LABELS` (site short-name map): replace `api.example.com` and `docs.example.com` with your own sites' domains and short names.
- Host names: the template shows the hosts as `Host 1` and `Host 2` (`var name` in `hostSection`). Change them to your real host names; do not put anything but the name there (no IPs).
- Demo data needs no change: the demo (`__SAMPLE__`) uses `example.com` sample data and random private IPs, and contains no real addresses.

### Monitoring multiple sites

The board shows every site that appears in the nginx access logs it reads (see `__LOG_DIR__`). There is no site list to configure in the generator. To give a site a colour and a short name on the board, add its domain to `SITE_DOTS` / `SITE_LABELS` in `index.html.template`.

4. New host prerequisites

- Python 3 (3.6+ works; the code is written for 3.6 and runs on newer versions too).
- nginx (or any web server that can serve static files) + a dashboard domain.
- nginx access logs: one file per site, named `<site>.log` (the site name is the filename minus `.log`), placed in `__LOG_DIR__`. The log format must be nginx combined (the fields `_LOG_RE` needs: IP, time, method, path, status, size, UA). If the new host's log format differs, update `_LOG_RE` in the generator.
- SSH logs (needed for the host security panel): the scripts read `/var/log/secure` (see "Hardcoded paths" in Section 3).
- fail2ban (optional): without it the fail2ban stats in the host panel are 0; nothing else is affected.
- The generator and refresh script must run as a user that can read the logs and write to the dashboard directory and the blocklist (on the old host, root runs them via cron).

5. Deployment steps (in order)

Follow these steps top to bottom on a fresh host.

1. Install Python 3 and nginx on the new host; set up the vhost and log format per Section 4.
2. Copy the templates from this directory to the new host, fill in the placeholders from Section 3, and drop the `.template` suffix:
   - `dashboard-init.py.template` → `__SCRIPT_DIR__` as `dashboard-init.py` (e.g. `/root/dashboard-init.py`)
   - `refresh-security-data.py.template` → `__REFRESH_SCRIPT__`
   - The two lines in `dashboard.cron.template` → root's crontab (`crontab -e`)
   - The two blocks in `nginx-dashboard.conf.template` → the http block of `nginx.conf` and the site configuration
3. Copy the dashboard page template `index.html.template` from this directory, change the sites and host names as described at the end of Section 3, drop the `.template` suffix and place it in `__DASHBOARD_DIR__` as `index.html`; then copy the 6 images from this directory's `assets/` (`fish-sprite.webp`, `fish-zero.webp`, `ie-redx.png`, `medal-gold.png`, `medal-silver.png`, `medal-bronze.png`) to the same directory. Do not copy `s-<key>.json`; the new host generates it itself.
4. Set up the rules file. Copy `.dashboard-rules.json.template` from this directory, fill in its placeholders (`__LAN_NET__`, `__HOME_IP__`, `__EGRESS_NET_1__`, `__EGRESS_NET_2__`, `__TAILSCALE_NET__`), drop the `.template` suffix and place it at `__RULES_PATH__` as `.dashboard-rules.json`. Note: if this file is missing or corrupt, the generator writes an unknown dashboard (the no-fallback rule settled on 2026-11-26), but banning keeps running. So confirm the rules file is in place before verifying.
5. State files: there is nothing to copy — the generator rebuilds them (see Section 8).
6. Include the blocklist in nginx (first block of the template), then `nginx -t` and reload once it passes.
7. Manually run one full run to verify: `cd __SCRIPT_DIR__ && python3 dashboard-init.py`. Confirm a new `s-<key>.json` is generated in `__DASHBOARD_DIR__`, the dashboard opens with data using the new key's link, and the daily report email arrives.
8. Manually run the 30-minute refresh once to verify: `__REFRESH_SCRIPT__`. Confirm `generated_at` changes.
9. Connect the cross-host wiring (Section 9), if you have a second host.
10. Watch one full statistics window (07:00 → 22:00), confirm the grades, bans, and host panels are all normal, then switch DNS.

6. Post-deployment checklist

- [ ] The dashboard opens with the new key's link, with grades, attack counts, attackers, and incidents all showing data.
- [ ] `generated_at` in `s-<key>.json` is changing (the 30-minute refresh is working).
- [ ] The 07:00 daily report email arrives and its link opens.
- [ ] The IP ban list matches nginx `blocklist.conf`.
- [ ] The host panel has data; stranger-login prompts from the first day or two stop after each is confirmed.
- [ ] The rules file is in place (the dashboard is not the unknown page).

### Migration from an old host

7. Migration steps (from an old host)

Moving the dashboard from an old host to a new one. Follow Section 5, with these differences:

1. Prerequisites and templates: same as Section 5, steps 1–3.
2. Rules file: copy the old host's `.dashboard-rules.json` to `__RULES_PATH__` (do not use the `.dashboard-rules.json.template` in this directory).
3. State files: copy the three files marked "Copy" in Section 8 from the old host (instead of letting the generator rebuild them).
4. Mail password: copy `SENDER_PASSWORD` unchanged from the old host's `dashboard-init.py` into `__SMTP_PASSWORD__`; do not reset it, do not leak it.
5. nginx, verification runs, cross-host wiring: same as Section 5, steps 6–9.
6. Watch one full statistics window (07:00 → 22:00), confirm the grades, bans, and host panels are all normal, then switch DNS and stop the old host. Do not stop the old host too early.

8. State files: which to copy, which to rebuild

| File | Action | Why |
|---|---|---|
| `.autoban-state.json` | Copy | Ban records; without it, ban history is lost |
| `.autoban-7d.json` | Copy | 7-day cumulative counts; without it, counts reset to zero and repeat attackers must accumulate to 100 again before being banned |
| `.dashboard-resolutions.json` | Copy | Resolved-incident registry; without it, resolved incidents reappear as "needs attention" |
| `.hostsec-state.json` | Do not copy; rebuild | Baseline of the machine's own SSH state; the new machine has different SSH host keys/fingerprints, so copying would cause false alarms |
| `.hostsec-ip-baseline.json` | Do not copy; rebuild | Same as above |
| `usually-key.json` | Do not copy; rebuild | Baseline of usual key locations; rebuilds naturally after a few runs on the new host (the old host does not have this file yet) |
| `.dashboard-init.lock` | Do not copy | Lock file; the new host creates it itself |
| `s-<key>.json` | Do not copy | Only one is kept at a time and deleted on rotation; the new host generates it itself |

While the host baselines rebuild, stranger-login prompts may appear for the first day or two; confirm each one and they will stop.

9. Two hosts: how host 2 pushes its data to host 1

The dashboard runs on host 1, but it can also show a second host (host 2). The design supports exactly two hosts; a third host needs code changes. The two hosts run **different scripts**: host 1 runs `dashboard-init.py` (the generator), host 2 runs `push-host-data.py` (a smaller script that only collects host 2's own host-security data and pushes it to host 1).

**How the push works.** On host 2, root's crontab runs the push script once a day at 06:30:

```
30 6 * * * /usr/bin/python3 /root/push-host-data.py --push-host >> /root/host-push.log 2>&1
```

`--push-host` only collects host 2's own host-security data (SSH logins, panel logins, key changes) and scp's it to host 1 through the receiving account's SSH channel, into `__HOST2_JSON_PATH__` (e.g. `/home/<receiver>/host-host2.json`) in the receiving account's home directory. It sends no email. Host 1 reads that file when it generates the board at 07:00 and merges host 2's row into the host panel. The push log is `/root/host-push.log` on host 2. The same `--push-host` also runs every 30 minutes (06:30–22:00) as part of the refresh script (Section 5, step 2), so host 2's data stays fresh through the day.

**Setting it up on a new deployment.** On host 2, copy the `push-host-data.py.template`, fill in the placeholders (the network placeholders and the push placeholders `__PUSH_SSH_KEY__`, `__SSH_PORT__`, `__HOST2_PUSH_DEST__`), and add the 06:30 cron line above. On host 1, fill in `__HOST2_JSON_PATH__` with the path of the pushed file. If you only monitor one host, skip the push entirely and remove the `__HOST2_JSON_PATH__` read from the generator (`_read_host2`); the host panel will show only the local host.

- **host 2 → dashboard host (30-minute refresh trigger)**: host 2 remotely triggers host 1's refresh every 30 minutes. On the new host, just run the 30-minute refresh from a local cron (Section 5, step 2); no need for the host 2 remote trigger.

10. Post-migration checklist

- [ ] The dashboard opens with the new key's link, with grades, attack counts, attackers, and incidents all showing data.
- [ ] `generated_at` in `s-<key>.json` is changing (the 30-minute refresh is working).
- [ ] The 07:00 daily report email arrives and its link opens.
- [ ] The IP ban list matches nginx `blocklist.conf`.
- [ ] The host panel has data; stranger-login prompts from the first day or two stop after each is confirmed.
- [ ] The rules file is in place (the dashboard is not the unknown page).

11. Notes

- The key rotates daily at 07:00; links with the old key stop working after rotation (they show demo data) — that is normal; the link in each day's report email is new.
- Only one `s-<key>.json` is kept at a time; the old one is deleted on rotation. Do not delete or copy it manually.
- Keep nginx 444 as-is; do not change it to 403 (settled 2026-11-26). 444 probes still count as incidents, with the verdict "not breached".
- If the rules file is lost or corrupt, the dashboard shows unknown, but banning keeps running (only the 7-day count update stops), and host grades are unaffected.
- Templates are snapshots: after `dashboard-init.py` changes, regenerate the template (or copy the new version and change its path constants); do not apply an old template to a new generator.
