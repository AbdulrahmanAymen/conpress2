# Self-Hosted Frappe Press on Ubuntu 24.04+/26.04 — Two-Server Setup

A complete, from-scratch guide to deploying Frappe Press on **two servers** instead of the
standard four, without AWS Route 53 or cloud provider API access. Written from an actual
end-to-end deployment on Contabo VPS instances running Ubuntu 26.04 ("resolute").

Every fix in this guide was hit in practice, in the order listed, so you can apply them
pre-emptively instead of discovering them one failed playbook at a time.

---

## Table of contents

- [Why this guide exists](#why-this-guide-exists)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Part 1 — Server preparation](#part-1--server-preparation)
- [Part 2 — Install Press on Server 1](#part-2--install-press-on-server-1)
- [Part 3 — Ubuntu 24.04+ compatibility fixes](#part-3--ubuntu-2404-compatibility-fixes)
- [Part 4 — Production setup and dashboard SSL](#part-4--production-setup-and-dashboard-ssl)
- [Part 5 — SSH chain](#part-5--ssh-chain)
- [Part 6 — Root Domain](#part-6--root-domain)
- [Part 7 — Wildcard TLS certificate](#part-7--wildcard-tls-certificate)
- [Part 8 — Proxy Server](#part-8--proxy-server)
- [Part 9 — Unified Server (App + Database)](#part-9--unified-server-app--database)
- [Part 10 — DNS records](#part-10--dns-records)
- [Part 11 — Apps, sources and releases](#part-11--apps-sources-and-releases)
- [Part 12 — Release Group](#part-12--release-group)
- [Part 13 — Docker registry](#part-13--docker-registry)
- [Part 14 — Deploy Candidate](#part-14--deploy-candidate)
- [Part 15 — Add Site to Proxy (Upstream Registration)](#part-15--add-site-to-proxy-upstream-registration)
- [Part 16 — Create a site](#part-16--create-a-site)
- [Part 17 — Static assets return 404 even though the files exist](#part-17--static-assets-return-404-even-though-the-files-exist)
- [Part 18 — Dashboard white screen on a fresh device](#part-18--dashboard-white-screen-on-a-fresh-device)
- [Part 19 — Migrating sites from an existing bench](#part-19--migrating-sites-from-an-existing-bench)
- [Troubleshooting reference](#troubleshooting-reference)
- [Production checklist](#production-checklist)
- [Known open issues](#known-open-issues)

---

## Why this guide exists

The official Press guides assume four separate servers, a real AWS account with Route 53, and
IAM credentials that let Press provision machines for you. This guide covers the case where:

- You have **two servers you provisioned yourself** (any provider — Contabo, Hetzner, AWS, bare metal)
- You have **no Route 53** and no cloud provider API access
- You are on **Ubuntu 24.04+/26.04**, which several parts of the Press codebase do not yet expect

### A note on the official guide's age

The widely-referenced community guide carries this warning at the top:

> Edit 1: this will work for Press before the below commit on develop branch...
> Commit Hash: `7c2069324ea20c40fd54e2d37a42e08ec3179a27` (Aug 12, 2025)

After that commit, server forms no longer take IP and Private IP directly — they are pulled from
a `Virtual Machine` doctype that only exists when Press provisions the machine itself. This is
why **`is_self_hosted` must be checked** on every server doc in this guide, even though the
official guide never mentions it. See [Part 8](#part-8--proxy-server) for the mechanics.

---

## Architecture

Press has four roles. They do **not** require four machines, but they cannot be naively stacked
either — each agent-based role writes to the same fixed paths (`/home/frappe/agent`,
`/etc/nginx/nginx.conf`, `/etc/nginx/conf.d/agent.conf`), so running two of them on one IP via
separate "Setup Server" runs will collide.

Press solves this officially with `unified_server.yml`, which runs the App and Database roles in
a **single coordinated Ansible play** rather than two separate ones.

| Server | Roles | Why it works |
|---|---|---|
| **Server 1** | Press dashboard + Proxy Server | Dashboard runs as user `press` under `/home/press/frappe-bench` (a plain bench, no agent). Proxy runs as user `frappe` under `/home/frappe/agent`. Different users, different paths — no collision. |
| **Server 2** | App + Database (Unified) | Handled by `unified_server.yml` as one play. |

### Example deployment (this guide's reference names)

| Doc name | Role | Physical host | IP |
|---|---|---|---|
| `press.example.com` | Dashboard site | Server 1 | `SERVER1_IP` |
| `n1.sites.example.com` | Proxy Server | Server 1 | `SERVER1_IP` |
| `db.sites.example.com` | Database Server | Server 2 | `SERVER2_IP` |
| `f1.sites.example.com` | App Server (Unified) | Server 2 (same box as `db`) | `SERVER2_IP` |

### Recommended sizing

| Setup | Spec | Notes |
|---|---|---|
| Two servers | 6 vCPU / 12 GB each | Recommended. Total 12 cores. |
| One server | 8 vCPU / 24 GB | Possible but not advised — single point of failure, DB competes with app workloads. |

### Provider considerations

If using a budget VPS provider rather than AWS, the factor that matters most is **disk I/O on
the database server**, since it directly affects response time for every hosted site. CPU on
oversold shared hosts is the second concern. Neither blocks the setup; both affect production
performance.

---

## Prerequisites

- **2 servers**, Ubuntu 24.04+ LTS, root/sudo access, static IPs
- **A registered domain** with real DNS record control (not just a single flat DuckDNS-style
  mapping — you need to create several independent A records under it; see
  [Part 10](#part-10--dns-records))
- **Open ports** on both servers: `22, 80, 443, 3306, 3022, 8000`
- **An SSH keypair** you control, usable on both servers

---

## Part 1 — Server preparation

Do this on **both** servers.

### 1.1 Swap (prevents OOM during Docker builds)

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 1.2 UID 1000 must be free

Press creates a `frappe` user with UID 1000. On Ubuntu cloud images, the default user often
already holds it.

```bash
getent passwd 1000        # see who holds it
```

If the default user (`ubuntu`) holds UID 1000, move it. **Run this from a session that is not
logged in as that user**, or you will kill your own connection:

```bash
sudo loginctl terminate-user ubuntu
sudo pkill -9 -u ubuntu
sudo usermod -u 1002 ubuntu
sudo find /home/ubuntu -uid 1000 -exec chown ubuntu {} +

getent passwd 1000        # must now be empty
```

### 1.3 sshd alias

Several Press playbooks reference `sshd.service`. On Ubuntu the unit is `ssh.service`.

```bash
sudo ln -sf /lib/systemd/system/ssh.service /etc/systemd/system/sshd.service
sudo systemctl daemon-reload
```

---

## Part 2 — Install Press on Server 1

### 2.1 Create the `press` user

```bash
sudo adduser press
sudo usermod -aG sudo press
sudo su - press
```

**Everything in Part 2 and Part 3 runs as `press`.**

### 2.2 Base packages

```bash
timedatectl set-timezone "Africa/Cairo"   # your timezone

sudo apt-get update -y && sudo apt-get upgrade -y
sudo apt-get install -y git python3-dev python3-pip python3-setuptools python3-venv
sudo apt-get install -y software-properties-common
sudo apt-get install -y mariadb-server mariadb-client
sudo apt-get install -y redis-server
sudo apt-get install -y xvfb libfontconfig libmysqlclient-dev pkg-config
```

### 2.3 wkhtmltopdf

```bash
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-2/wkhtmltox_0.12.6.1-2.jammy_amd64.deb
sudo apt install ./wkhtmltox_0.12.6.1-2.jammy_amd64.deb -y
wkhtmltopdf --version    # must say "(with patched qt)"
```

### 2.4 MariaDB configuration

```bash
sudo nano /etc/mysql/my.cnf
```

```ini
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
```

```bash
sudo service mysql restart
sudo mysql_secure_installation
```

| Prompt | Answer | Reason |
|---|---|---|
| Switch to unix_socket authentication? | **n** | See warning below |
| Change the root password? | **Y** | Save it — `bench new-site` needs it |
| Remove anonymous users? | **Y** | |
| Disallow root login remotely? | **Y** | |
| Remove test database? | **Y** | |
| Reload privilege tables? | **Y** | |

> **This contradicts the official ERPNext guide, deliberately.** Answering `Y` to unix_socket
> runs `ALTER USER 'root'@'localhost' IDENTIFIED VIA unix_socket`, which makes root accept logins
> **only** when the OS user is root. `bench` runs as `press` and connects with a password, so it
> gets `MySQL Access Denied (1698)`.

### 2.5 Node 20 (not 18)

```bash
cd ~
curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash
source ~/.profile
nvm install 20
nvm alias default 20
nvm use 20
sudo apt-get install npm -y
sudo npm install -g yarn
node --version    # must be v20.x
```

> The official guides say Node 18. Current Press `develop` pulls `@vitejs/plugin-vue@6.x`, which
> requires `^20.19.0 || >=22.12.0`. With Node 18 `bench get-app press` fails at `yarn install`.

### 2.6 Bench and site

```bash
sudo pip3 install frappe-bench --break-system-packages
cd ~
bench init --frappe-branch version-15 frappe-bench
cd frappe-bench/
sudo chmod -R o+rx /home/press/

bench new-site press.example.com
bench get-app press
```

**Do not run `install-app` yet.** Apply Part 3 first.

---

## Part 3 — Ubuntu 24.04+ compatibility fixes

Apply all of these before `install-app press`. They are ordered as they would otherwise fail.

### 3.1 Dependency pins

```bash
cd ~/frappe-bench/apps/press
sed -i 's/"ansible==3.4.0",/"ansible>=9,<12",/' pyproject.toml
sed -i 's/"stripe~=2.56.0",/"stripe>=7,<8",/' pyproject.toml

cd ~/frappe-bench
./env/bin/pip install -e apps/press
```

### 3.2 TLS libraries

```bash
cd ~/frappe-bench
./env/bin/pip install "cryptography~=50.0.0" "pyOpenSSL~=26.4.0" --break-system-packages
```

### 3.3 Ansible plugin loader — the critical one

Without this, **every** `Setup Server` fails instantly with
`ModuleNotFoundError: No module named 'ansible_collections.ansible.builtin'` — before SSH is even
attempted.

```bash
nano ~/frappe-bench/apps/press/press/runner.py
```

```python
# at the top, with the other imports
from ansible.plugins.loader import init_plugin_loader

...

class Ansible:
    def __init__(self, server, playbook, user="root", variables=None, port=22):
        init_plugin_loader()          # <-- ADD THIS LINE
        self.server = server
        ...
```

### 3.4 Install the app

```bash
cd ~/frappe-bench
bench --site press.example.com install-app press
```

**`urllib3.contrib.appengine` ImportError:**

```bash
perl -i -pe 's/^(\s+)import telegram\.vendor\.ptb_urllib3\.urllib3\.contrib\.appengine as appengine$/$1try:\n$1    import telegram.vendor.ptb_urllib3.urllib3.contrib.appengine as appengine\n$1except ImportError:\n$1    appengine = None/' ~/frappe-bench/env/lib/python3.12/site-packages/telegram/utils/request.py

perl -i -pe 's/^(\s+)import urllib3\.contrib\.appengine as appengine(\s+#.*)?$/$1try:\n$1    import urllib3.contrib.appengine as appengine$2\n$1except ImportError:\n$1    appengine = None/' ~/frappe-bench/env/lib/python3.12/site-packages/telegram/utils/request.py

sed -i 's/appengine\.AppEngineManager,/type(None),/' ~/frappe-bench/env/lib/python3.12/site-packages/telegram/utils/request.py
sed -i "s/if appengine\.is_appengine_sandbox():/if appengine and appengine.is_appengine_sandbox():/" ~/frappe-bench/env/lib/python3.12/site-packages/telegram/utils/request.py
```

**`sqlparse` `MAX_GROUPING_TOKENS` AttributeError:**

```bash
cd ~/frappe-bench
./env/bin/pip install --force-reinstall "sqlparse~=0.5.4"
```

### 3.5 MariaDB role — remove the dead 10.6 repository, and the `apt-key` module

Two separate problems live in this one role, hit at two different points (first when the
package is installed, then again on every server the role runs against):

**a) The apt repository itself is dead**, and **b) the task that adds its key uses Ansible's
`apt_key` module**, which shells out to the `apt-key` binary — removed entirely in Ubuntu
24.04+. Without this fix, `Setup Server`/`Setup Unified Server` fails on **every** new
Database Server with:

```
Failed to find required executable "apt-key" in paths: /usr/local/sbin:/usr/local/bin:...
```

```bash
cd ~/frappe-bench/apps/press/press/playbooks/roles/mariadb/tasks
cp main.yml main.yml.bak

sed -i '/- name: Add MariaDB Repository Key/,/- name: Update APT Cache/{/- name: Update APT Cache/!d}' main.yml
sed -i 's/libmariadbclient18/libmariadb3/' main.yml

python3 -c "import yaml; yaml.safe_load(open('main.yml')); print('YAML OK')"
```

### 3.6 Docker role — same `apt_key` problem, for two different keys

The `docker` role's task file adds **both** the Docker repo key and (oddly) the NGINX repo key
in the same file, both via the same broken `apt_key` module. This blocks `Setup Server` on the
**Proxy Server** with the identical `apt-key` error as 3.5, just in a different role.

```bash
cd ~/frappe-bench/apps/press/press/playbooks/roles/docker/tasks
cp main.yml main.yml.bak

python3 << 'PYEOF'
path = "main.yml"
with open(path) as f:
    content = f.read()

old = '''- name: Add Docker Repository Key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

# This seems to be breaking on some servers
- name: Add NGINX Repository Key
  apt_key:
    url: https://nginx.org/keys/nginx_signing.key
    state: present

- name: Add Docker Repository
  apt_repository:
    repo: deb https://download.docker.com/linux/ubuntu focal stable
    state: present
    update_cache: true'''

new = '''- name: Add Docker Repository Key
  ansible.builtin.get_url:
    url: https://download.docker.com/linux/ubuntu/gpg
    dest: /tmp/docker.asc

- name: Dearmor Docker Repository Key
  ansible.builtin.shell: gpg --yes --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg /tmp/docker.asc
  args:
    creates: /usr/share/keyrings/docker-archive-keyring.gpg

# This seems to be breaking on some servers
- name: Add NGINX Repository Key
  ansible.builtin.get_url:
    url: https://nginx.org/keys/nginx_signing.key
    dest: /tmp/nginx_signing.key

- name: Dearmor NGINX Repository Key
  ansible.builtin.shell: gpg --yes --dearmor -o /usr/share/keyrings/nginx-archive-keyring.gpg /tmp/nginx_signing.key
  args:
    creates: /usr/share/keyrings/nginx-archive-keyring.gpg

- name: Add Docker Repository
  apt_repository:
    repo: "deb [signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu focal stable"
    state: present
    update_cache: true'''

content = content.replace(old, new)
with open(path, "w") as f:
    f.write(content)
PYEOF
```

> `focal` in the Docker repo line is intentional — Docker has no official build for
> `resolute`/26.04 yet, so the repo stays pinned to the last codename it supports.

### 3.7 Agent role — `virtualenv_python` must be provisioned per-server, not hard-coded

The agent's Python dependency (`duckdb`, needed for binlog indexing) has no prebuilt wheel for
the system's default Python (3.14 on this Ubuntu release), so the agent must be installed into a
venv built with Python 3.12 specifically. Don't hard-code an absolute path found on one server —
provision it per-target-server instead, using `uv` (commonly already present system-wide):

```bash
cd ~/frappe-bench/apps/press/press/playbooks/roles/agent/tasks
cp main.yml main.yml.bak

python3 << 'PYEOF'
path = "main.yml"
with open(path) as f:
    content = f.read()

old = '''- name: Install Agent
  become: yes
  become_user: frappe
  pip:
    name: file:///home/frappe/agent/repo
    virtualenv: /home/frappe/agent/env
    virtualenv_python: python3
    editable: yes'''

new = '''- name: Ensure Python 3.12 is available for frappe
  become: yes
  become_user: frappe
  command: uv python install 3.12
  args:
    creates: /home/frappe/.local/bin/python3.12

- name: Install Agent
  become: yes
  become_user: frappe
  pip:
    name: file:///home/frappe/agent/repo
    virtualenv: /home/frappe/agent/env
    virtualenv_python: /home/frappe/.local/bin/python3.12
    editable: yes'''

content = content.replace(old, new)
with open(path, "w") as f:
    f.write(content)
PYEOF
```

> If `uv` isn't already on the target server's PATH, add a bootstrap task before this one:
> `command -v uv || curl -LsSf https://astral.sh/uv/install.sh | sh` as `become_user: frappe`.

### 3.8 nginx role — "Add www-data to Frappe group" is wrongly restricted to Jammy

The nginx role includes a task that adds `www-data` (the user nginx workers run as) to the
`frappe` group — necessary because `/home/frappe` is created `750`, so nginx can't otherwise
even traverse into the bench's asset directories. On this Press checkout, the task is gated
behind `when: ansible_distribution_release == 'jammy'`, so it silently never runs on
24.04+/26.04, and every fresh App Server needs it applied by hand (see
[Part 17](#part-17--static-assets-return-404-even-though-the-files-exist) for the symptom).

Find the task and remove the `when` condition so it runs unconditionally:

```bash
grep -rn "www-data.*frappe\|jammy" ~/frappe-bench/apps/press/press/playbooks/roles/nginx/tasks/main.yml
```

Remove the matched `when: ansible_distribution_release == 'jammy'` line from that task, then
validate:

```bash
python3 -c "import yaml; yaml.safe_load(open('/home/press/frappe-bench/apps/press/press/playbooks/roles/nginx/tasks/main.yml')); print('YAML OK')"
```

---

## Part 4 — Production setup and dashboard SSL

```bash
cd ~/frappe-bench
bench --site press.example.com list-apps        # expect frappe + press
bench --site press.example.com enable-scheduler
bench --site press.example.com set-maintenance-mode off

sudo env "PATH=$PATH" bench setup production press
bench setup nginx
```

### 4.1 Supervisor link

```bash
sudo supervisorctl status
```

If empty:

```bash
sudo ln -sf /home/press/frappe-bench/config/supervisor.conf /etc/supervisor/conf.d/frappe-bench.conf
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart all
```

### 4.2 Firewall

```bash
sudo ufw allow 22,25,143,80,443,3306,3022,8000/tcp
sudo ufw enable
```

### 4.3 Dashboard certificate — use bench, not `certbot --nginx`

```bash
bench config dnsmultitenant on
sudo bench setup lets-encrypt press.example.com
```

> **Do not use `sudo certbot --nginx`.** It restarts nginx with its own method while the original
> nginx process still holds port 80, producing a bind() address-in-use error and leaving nginx
> down. `bench setup lets-encrypt` stops nginx first, obtains the cert, updates
> `site_config.json`, rebuilds the nginx config, and starts it again.

---

## Part 5 — SSH chain

Press connects to every managed server as `root`, **by IP**. This includes Server 1 connecting
to itself for the Proxy role.

### 5.1 Enable root login on both servers

```bash
sudo nano /etc/ssh/sshd_config
```

```
PermitRootLogin yes
PasswordAuthentication no
```

```bash
sudo systemctl restart ssh
```

### 5.2 Put the private key on Server 1

Press initiates connections *from* Server 1, so the key must live there.

```bash
# on Server 1
sudo cp ~/yourkey.pem /home/press/.ssh/
sudo chown press:press /home/press/.ssh/yourkey.pem
sudo chmod 400 /home/press/.ssh/yourkey.pem
```

### 5.3 SSH config on Server 1, as user `press`

Entries must be keyed **by IP** — that is what Press dials.

```bash
nano /home/press/.ssh/config
```

```
Host SERVER1_IP
    User root
    IdentityFile ~/.ssh/yourkey.pem
    StrictHostKeyChecking no

Host SERVER2_IP
    User root
    IdentityFile ~/.ssh/yourkey.pem
    StrictHostKeyChecking no
```

### 5.4 Verify both paths

```bash
sudo su - press
ssh root@SERVER1_IP "whoami"    # loopback — must print root
ssh root@SERVER2_IP "whoami"    # must print root
```

---

## Part 6 — Root Domain

Create it in the UI at `https://press.example.com/app/root-domain/new` — the wildcard root
under which every customer site will live, e.g. `sites.example.com`.

---

## Part 7 — Wildcard TLS certificate

Press will try to obtain this automatically and fail, because the automatic path requires
Route 53. Obtain it manually with your own DNS provider's certbot plugin and insert the doc
yourself.

### 7.1 Request the certificate — force RSA

```bash
sudo certbot certonly \
  --non-interactive \
  --cert-name sites.example.com \
  --key-type rsa \
  --rsa-key-size 2048 \
  -d "*.sites.example.com" \
  --agree-tos \
  --email you@example.com \
  --config-dir /home/press/.certbot \
  --work-dir /home/press/.certbot/work \
  --logs-dir /home/press/.certbot/logs

sudo chown -R press:press /home/press/.certbot
```

> **`--key-type rsa` is mandatory.** Modern certbot defaults to ECDSA. Press validates the key
> against its `rsa_key_size` field and rejects an ECDSA key.

### 7.2 Insert the certificate doc

```bash
cd ~/frappe-bench
bench --site press.example.com console
```

```python
import frappe

base = "/home/press/.certbot/live/sites.example.com"
cert      = open(f"{base}/cert.pem").read()
chain     = open(f"{base}/chain.pem").read()
fullchain = open(f"{base}/fullchain.pem").read()
privkey   = open(f"{base}/privkey.pem").read()

doc = frappe.get_doc({
    "doctype": "TLS Certificate",
    "domain": "sites.example.com",
    "wildcard": 1,
    "provider": "Other",          # critical — see below
    "rsa_key_size": "2048",
    "team": "YOUR_TEAM_ID",
    "certificate": cert,
    "intermediate_chain": chain,
    "full_chain": fullchain,
    "private_key": privkey,
})
doc.insert(ignore_permissions=True)
frappe.db.commit()
```

> **`provider` must be `"Other"`, not `"Let's Encrypt"`.** With `"Let's Encrypt"`, `after_insert`
> triggers a real certbot run that needs Route 53 and fails, marking the doc `Failure` — and a
> scheduled renewal job will keep retrying on its own forever. With `"Other"`, validation just
> checks the key matches the certificate and sets `Active`, and the renewal job ignores the doc.

### 7.3 Renewal cron

```bash
sudo crontab -e
```

```
0 3 1 * * certbot certonly --non-interactive --force-renewal --cert-name sites.example.com --key-type rsa --rsa-key-size 2048 -d "*.sites.example.com" --agree-tos --email you@example.com --config-dir /home/press/.certbot --work-dir /home/press/.certbot/work --logs-dir /home/press/.certbot/logs && chown -R press:press /home/press/.certbot
```

> After each renewal you must also update the certificate content in the `TLS Certificate` doc —
> the files change but the doc does not follow automatically.

---

## Part 8 — Proxy Server

`https://press.example.com/app/proxy-server/new`

| Field | Value |
|---|---|
| Hostname | `n1` |
| Is Self Hosted | ✅ |
| Self Hosted Server Domain | `sites.example.com` |
| IP / Private IP | Server 1's IP |
| SSH User / Port | `root` / `22` |

Save, then **Actions → Setup → Setup Server**.

Expect to hit and apply [3.5](#35-mariadb-role--remove-the-dead-106-repository-and-the-apt_key-module)
and [3.6](#36-docker-role--same-apt_key-problem-for-two-different-keys) here if not already
applied ahead of time.

---

## Part 9 — Unified Server (App + Database)

Two docs on the **same** IP (Server 2):

**1) Database Server** — `/app/database-server/new`:

| Field | Value |
|---|---|
| Hostname | `db` |
| Is Self Hosted | ✅ |
| Self Hosted Server Domain | `sites.example.com` |
| IP / Private IP | Server 2's IP |

**2) App Server** — `/app/server/new`, same IP:

| Field | Value |
|---|---|
| Hostname | `f1` |
| Is Self Hosted | ✅ |
| **Is Unified Server** | ✅ |
| Self Hosted Server Domain | `sites.example.com` |
| Database Server | the doc above |
| Proxy Server | the `n1` doc from Part 8 |

Then **Actions → Setup → Setup Unified Server**.

### 9.1 MariaDB 11.8.6-5+ changed its default datadir

Starting with `1:11.8.6-5`, the Ubuntu/Debian MariaDB package moved its **default datadir from
`/var/lib/mysql` to `/var/lib/mariadb`** ("for better MySQL co-installability"). The package's
own post-install step auto-initializes the DB at the new path on first boot, but Press's
`mariadb.cnf` template hard-codes `datadir = /var/lib/mysql` — which has never been initialized
— so the very next task that restarts MariaDB fails:

```
Unable to restart service mysql: Job for mariadb.service failed because the control
process exited with error code.
```

```bash
grep datadir /etc/mysql/mariadb.conf.d/50-server.cnf
# confirms: datadir = /var/lib/mariadb  (new default)
```

Fix — re-init at the path Press expects (safe pre-site, no real data yet):

```bash
sudo systemctl stop mariadb
sudo rm -rf /var/lib/mariadb
sudo mkdir -p /var/lib/mysql
sudo chown mysql:mysql /var/lib/mysql
sudo mariadb-install-db --datadir=/var/lib/mysql --user=mysql
sudo systemctl start mariadb
```

### 9.2 AppArmor blocks `mariadbd` from writing to `/var/lib/mysql`

If 9.1's re-init instead fails with **permission denied** (`Errcode: 13`) despite correct
ownership:

```
Can't create test file '/var/lib/mysql/<hostname>.lower-test' (Errcode: 13 "Permission denied")
/usr/sbin/mariadbd: Cannot change uid/gid (errno: 1)
```

Check `dmesg | grep -i apparmor` for `DENIED ... capability=... capname="dac_override"` /
`"setgid"` on the `mariadbd` profile — the packaged AppArmor profile was tuned for the new
`/var/lib/mariadb` default and denies the capabilities `mariadbd` needs to init at
`/var/lib/mysql`.

```bash
sudo apt-get update && sudo apt-get install -y apparmor-utils   # not installed by default
sudo aa-complain mariadbd

sudo systemctl stop mariadb
sudo rm -rf /var/lib/mysql/*
sudo mariadb-install-db --datadir=/var/lib/mysql --user=mysql
sudo systemctl start mariadb
```

`aa-complain` leaves the profile loaded (violations still logged) but stops it from blocking
them — acceptable for a database host, and avoids hand-patching a profile file whose exact
location varies between images.

---

## Part 10 — DNS records

Two distinct concerns, easy to conflate — keep them separate.

### 10.1 Exact records for each Server/Proxy Server/Database Server doc

Press's `Agent` class (`press/agent.py`, `get_request_url`) builds every agent HTTP request as
`https://{server_doc_name}:{port}/agent/{path}` — **using the doc's own name as the connection
host whenever a public `ip` is set on the doc** (true for self-hosted servers). This means every
server doc name needs its own real DNS A record, or the dashboard cannot talk to that server's
agent at all — regardless of how the SSH/Ansible provisioning itself went:

| Type | Name | Value |
|---|---|---|
| A | `n1.sites.example.com` | Server 1's IP |
| A | `f1.sites.example.com` | Server 2's IP |
| A | `db.sites.example.com` | Server 2's IP |

Without these, Agent Jobs targeting that server stay stuck `Undelivered` forever with a DNS
resolution failure buried in the traceback (see [Troubleshooting](#troubleshooting-reference)).

> This is a real, general-purpose domain, not a DuckDNS-style single-record setup — you need
> genuine subdomain delegation to add these as independent records.

### 10.2 Wildcard record for customer sites

**Separately**, once you start creating actual customer sites, each one gets a subdomain like
`mysite.sites.example.com` that a browser needs to resolve — and it must point at **Server 1**
(the Proxy), not Server 2, since the Proxy is what receives visitor traffic and routes it
internally by Host header:

| Type | Name | Value |
|---|---|---|
| A | `*.sites.example.com` | Server 1's IP |

This does **not** conflict with the exact records in 10.1 — DNS exact matches take precedence
over wildcard matches, so `f1.sites.example.com` keeps resolving to Server 2 even with a
catch-all wildcard in place for everything else under `sites.example.com`.

---

## Part 11 — Apps, sources and releases

Frappe must be the first app in any release group.

### 11.1 Create the App

`https://press.example.com/app/app/new` — App Name `frappe`, Title `Frappe`.

### 11.2 Create the App Source

`https://press.example.com/app/app-source/new`

| Field | Value |
|---|---|
| App | `frappe` |
| Repository URL | `https://github.com/frappe/frappe` |
| Branch | `version-15` |
| Public | ✅ |
| Versions | `Version 15` |

### 11.3 Approve the App Release

An `App Release` is created automatically but **must be approved manually**:

```python
release = frappe.get_doc("App Release", "RELEASE_NAME")
release.status = "Approved"
release.save()
frappe.db.commit()
```

### 11.4 Repeat for other apps

Same three steps for `erpnext`, `hrms`, etc. App names must be lowercase and match the repo
name exactly — a mismatched case (e.g. an App doc named `ERPNext` instead of `erpnext`) causes
`ModuleNotFoundError` during install; rename the doctype record to fix it.

---

## Part 12 — Release Group

`https://press.example.com/app/release-group/new`

| Field | Value |
|---|---|
| Title | `Test Bench` |
| Version | `Version 15` |
| Apps | row 1 `frappe`, row 2 `erpnext` |
| Servers | `f1.sites.example.com` |

> Order matters — `frappe` first, then dependent apps.

---

## Part 13 — Docker registry

`https://press.example.com/app/press-settings` → **Docker** tab:

| Field | Value (Docker Hub example) |
|---|---|
| Docker Registry URL | `docker.io` |
| Docker Registry Namespace | your Docker Hub account name |
| Docker Registry Username | your Docker Hub account name |
| Docker Registry Password | a Docker Hub **Access Token** |
| Build Server | `f1.sites.example.com` |

Also required on the same doctype: **Certbot Directory** and **EFF Registration Email**, or it
won't save.

### 13.1 Docker Hub only supports one-level repository paths

Press builds the pushed image's repository name as:

```python
namespace = f"{settings.docker_registry_namespace}/{settings.domain}"
self.docker_image_repository = f"{settings.docker_registry_url}/{namespace}/{self.group}"
```

— a **3-level** path, e.g. `docker.io/youraccount/sites.example.com/bench-0001`. **Docker Hub
only accepts `namespace/repository` (2 levels)** — unlike AWS ECR, GHCR, or GitLab, which all
support arbitrary nesting. The push fails at the very last build step with:

```
no_id: Error denied: requested access to the resource is denied
```

even though `docker login` and a manual `docker push` with the same credentials succeed —
because a manual test naturally uses a valid 2-level name. This is **not** a credentials or
stale-session problem; re-logging in does not fix it.

Two ways to resolve it:

- **Switch registries (no code change):** point Docker Registry URL at `ghcr.io` (GitHub
  Container Registry) instead. GHCR accepts arbitrary path depth, so Press's existing 3-level
  name works unmodified. Needs a GitHub PAT with `write:packages`.
- **Stay on Docker Hub (patch Press):** flatten the path to 2 levels in
  `press/press/doctype/deploy_candidate_build/deploy_candidate_build.py`,
  `_update_docker_image_metadata()`:
  ```python
  repo = f"{settings.domain}-{self.group}".replace(".", "-")
  self.docker_image_repository = f"{settings.docker_registry_url}/{settings.docker_registry_namespace}/{repo}"
  ```

> ⚠️ *Fill in here which option was actually used once confirmed, plus any follow-up issues
> hit — this section was drafted from the diagnosis, not yet from a confirmed working
> end-to-end push.*

---

## Part 14 — Deploy Candidate

From the Release Group: **Actions → Create Deploy Candidate**, then **Deploy**.

### `Run Validations` fails: `/usr/bin/python3.14` not found

The agent's syntax-check step resolves a Python path by asking "could this app's
`requires-python` constraint theoretically match 3.14?" rather than "is 3.14 actually
installed?" Frappe's constraint is an open range like `>=3.10`, which matches — so the agent
commits to a Python that doesn't exist. The step only runs `compileall`, so symlinking is safe:

```bash
ssh root@SERVER2_IP
ln -sf /usr/bin/python3.12 /usr/bin/python3.14
```

### Docker socket permission denied on the build server

```bash
sudo chmod 666 /var/run/docker.sock
sudo usermod -aG docker frappe
```

### Build killed mid-way

Almost always OOM — confirm swap is active on Server 2 (`free -h | grep -i swap`).

---

## Part 15 — Add Site to Proxy (Upstream Registration)

Registers the App Server as an nginx upstream on the Proxy. Without this, no site on that App
Server is reachable, even if the site itself is `Active`.

### 15.1 Run it

From the **App Server** doc: **Actions → Add to Proxy**, or from the console:

```python
server = frappe.get_doc("Server", "f1.sites.example.com")
server.add_upstream_to_proxy()
frappe.db.commit()
```

> `add_upstream_to_proxy()` just enqueues an Agent Job — it returns immediately either way.
> Check the actual job:
> ```python
> jobs = frappe.get_all("Agent Job",
>     filters={"server": "n1.sites.example.com", "job_type": "Add Upstream to Proxy"},
>     fields=["name", "status", "creation"], order_by="creation desc", limit=3)
> ```
> **Note the `server` filter is the Proxy's name, not the App Server's** — the job actually
> targets the Proxy's own agent (it's the one writing the upstream file).

### 15.2 Job stuck `Undelivered`

`enqueue_http_request()` from an interactive console often doesn't actually fire the request.
Force it directly:

```python
job = frappe.get_doc("Agent Job", "JOB_NAME")
job.create_http_request()
frappe.db.commit()
```

If it's still `Undelivered` with no exception, check `should_skip_requests()`:

```python
from press.press.doctype.agent_job.agent_job import Agent
agent = Agent(job.server, server_type=job.server_type)
print(agent.should_skip_requests())
```

`True` almost always means the target server's own doc name doesn't resolve via DNS yet — see
[Part 10.1](#101-exact-records-for-each-serverproxy-serverdatabase-server-doc).

### 15.3 `401 Unauthenticated` despite a correct `agent_password`

The stored `agent_password` and the agent's own `access_token` hash in `config.json` can get out
of sync. Regenerate and push it — using a **single-quoted heredoc**, not `python3 -c "..."`,
because a double-quoted shell invocation eats the `$` characters in a pbkdf2 hash:

```bash
ssh root@TARGET_IP << 'REMOTE'
python3 << 'PYEOF'
import json, secrets
from werkzeug.security import generate_password_hash

new_password = secrets.token_hex(32)
config_path = "/home/frappe/agent/config.json"
with open(config_path) as f:
    config = json.load(f)
config["access_token"] = generate_password_hash(new_password, method="pbkdf2:sha256")
with open(config_path, "w") as f:
    json.dump(config, f, indent=2)
print("NEW_PASSWORD:", new_password)
PYEOF
REMOTE
```

Then update the matching `agent_password` field on the Server/Proxy Server doc to the printed
value, and restart the agent's **master** process, not just the supervisor entry:

```bash
sudo pkill -9 -f "gunicorn --bind 127.0.0.1:25052"
sudo supervisorctl start agent:web
```

`supervisorctl restart` alone leaves the old master process (and its old token) running.

### 15.4 Verify the nginx upstream was actually written

`is_upstream_setup` on the doc does **not** reliably reflect success — check the real files:

```bash
sudo find /home/frappe/agent/nginx/upstreams -type f
sudo grep -n "f1.sites.example.com" /etc/nginx/conf.d/proxy.conf
```

> Don't manually add `include /home/frappe/agent/nginx/proxy.conf;` to `nginx.conf` — that
> content is already included via `/etc/nginx/conf.d/proxy.conf`, and doubling it up produces
> `nginx: [emerg] "real_ip_header" directive is duplicate` and takes nginx down entirely.

---

## Part 16 — Create a site

Standard Press dashboard flow: pick the Release Group, name the site, deploy. The new site's
subdomain needs the wildcard from [10.2](#102-wildcard-record-for-customer-sites) to be
reachable in a browser.

---

## Part 17 — Static assets return 404 even though the files exist

### Symptom

The site's HTML loads (`200`), but every `/assets/...` request — CSS, JS, fonts — comes back
`404`, even though the files are confirmed present on disk with normal `644` permissions.

### Cause

`/home/frappe` is created `750` (`drwxr-x--- frappe frappe`). nginx's worker processes run as
`www-data`, which is in neither the owner nor the group, so it can't even **traverse** into the
directory — `try_files` reports this as `404`, not `403`. Confirm with:

```bash
namei -l /home/frappe/benches/BENCH_NAME/sites/assets/frappe/dist/css/website.bundle.XXXXXXXX.css
```

This is the exact scenario [3.8](#38-nginx-role--add-www-data-to-frappe-group-is-wrongly-restricted-to-jammy)
exists to prevent automatically on future servers.

### Fix (one-time per App Server)

```bash
sudo usermod -aG frappe www-data
sudo systemctl restart nginx
```

> Use `restart`, not `reload` — a running worker's OS-level group membership isn't
> re-evaluated by a config reload alone.

---

## Part 18 — Dashboard white screen on a fresh device

### Symptom

The dashboard loads fine on the machine you set it up from, but opens **blank/white** on any
other device — browser console shows a batch of `net::ERR_HTTP2_SERVER_REFUSED_STREAM` errors
partway through loading dozens of the dashboard's split JS chunks.

### Cause

nginx's `keepalive_requests` directive caps how many requests one HTTP/2 connection may serve
before nginx sends `GOAWAY` and closes it. The modern Press dashboard is code-split into 80-100+
small JS files loaded on a fresh page view. A device with a **warm cache** (the one you built the
server from) never notices, because it doesn't re-request everything on one connection — a
device with a **cold cache** hits the connection limit mid-load and the remaining chunks come
back refused.

```bash
grep -rn "keepalive_requests" /etc/nginx/nginx.conf /etc/nginx/conf.d/
```

### Fix

Raise the limit well above the dashboard's real asset count, in the `http {}` block of
`/etc/nginx/nginx.conf`:

```nginx
keepalive_requests 1000;
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Test from another device in a private/incognito window (cold cache) to confirm.

---

## Part 19 — Migrating sites from an existing bench

To bring sites from an existing (non-Press) bench into this system:

### 19.1 Export, on the old server

```bash
cd ~/frappe-bench
bench --site oldsite.example.com backup --with-files
```

Produces, under `sites/oldsite.example.com/private/backups/`:

```
{timestamp}-database.sql.gz
{timestamp}-files.tar             # public files
{timestamp}-private-files.tar
```

Check what apps the old site needs, so the target Release Group matches:

```bash
bench --site oldsite.example.com list-apps
```

### 19.2 Transfer

```bash
scp sites/oldsite.example.com/private/backups/*database* \
    sites/oldsite.example.com/private/backups/*files* \
    press@SERVER2_IP:/tmp/migration/
```

### 19.3 Import, on Server 2 (inside the bench container)

Benches on the App Server run as **Docker containers** (`docker ps` shows names like
`bench-0002-000002-f1`), so there is no `~/frappe-bench` on the host to run `bench` from.

First create the new site normally through the Press dashboard, on a Release Group with the
same apps and the same Frappe major version as the old site. Then:

```bash
ssh root@SERVER2_IP
docker ps                                   # find the bench container name
docker cp /tmp/migration/. CONTAINER:/home/frappe/frappe-bench/
docker exec -it CONTAINER bash
```

Inside the container, as `frappe`:

```bash
su - frappe
cd frappe-bench
bench --site newsite.sites.example.com --force restore \
    /home/frappe/frappe-bench/xxxx-database.sql.gz \
    --with-public-files /home/frappe/frappe-bench/xxxx-files.tar \
    --with-private-files /home/frappe/frappe-bench/xxxx-private-files.tar
bench --site newsite.sites.example.com migrate
bench --site newsite.sites.example.com clear-cache
```

If restore asks for the MariaDB root password, it is stored on the Database Server doc:

```python
frappe.get_doc("Database Server", "db.sites.example.com").get_password("mariadb_root_password")
```

### 19.4 Passwords and secrets

- The **Administrator login password** lives hashed in the database, so it survives the restore
  unchanged. If it is unknown, reset it on the *new* site: `bench --site newsite set-admin-password 'NEW'`.
- The site's **database password** (`db_password` in `site_config.json`) is not needed for
  `bench backup`, which reads it itself. The restored site gets fresh DB credentials from Press.
- If the old `site_config.json` contains an **`encryption_key`**, copy it into the new site's
  config after the restore. Without it, every stored Password field (email accounts,
  integration secrets) becomes unreadable. The backup's `*-site_config_backup.json` includes it.

> DocTypes, Customize Form changes, Property Setters, Client/Server Scripts and Print Formats
> travel automatically inside the database dump. Any **custom app** they belong to must already
> be installed on the target Release Group, since the dump restores data, not app code.

---

## Troubleshooting reference

| Symptom | Cause | Fix |
|---|---|---|
| `yarn install` fails, engine incompatible | Node 18 | Node 20 — [2.5](#25-node-20-not-18) |
| `MySQL Access Denied (1698)` | unix_socket auth enabled | Answer `n` — [2.4](#24-mariadb-configuration) |
| `ModuleNotFoundError: ansible_collections.ansible.builtin` | `init_plugin_loader()` never called | [3.3](#33-ansible-plugin-loader--the-critical-one) |
| `urllib3.contrib.appengine` ImportError | telegram + urllib3 2.x | [3.4](#34-install-the-app) |
| `MAX_GROUPING_TOKENS` AttributeError | sqlparse version | [3.4](#34-install-the-app) |
| MariaDB repo 404 / `apt-key` not found (mariadb role) | Rackspace 10.6 repo + removed `apt-key` binary | [3.5](#35-mariadb-role--remove-the-dead-106-repository-and-the-apt_key-module) |
| `apt-key` not found (docker role) | Same `apt_key` module, Docker+NGINX keys | [3.6](#36-docker-role--same-apt_key-problem-for-two-different-keys) |
| `RuntimeError: failed to find interpreter ... python3.12` | `virtualenv_python` hard-coded to one server's path | [3.7](#37-agent-role--virtualenv_python-must-be-provisioned-per-server-not-hard-coded) |
| `bind() to 0.0.0.0:80 failed` during certbot | `certbot --nginx` restart collision | `bench setup lets-encrypt` — [4.3](#43-dashboard-certificate--use-bench-not-certbot---nginx) |
| `Unit sshd.service not found` | Ubuntu names it `ssh` | [1.3](#13-sshd-alias) |
| TLS cert status flips to `Failure` | `provider = "Let's Encrypt"` triggers Route 53 path | Use `"Other"` — [7.2](#72-insert-the-certificate-doc) |
| `Private key length does not match` | certbot issued ECDSA | `--key-type rsa` — [7.1](#71-request-the-certificate--force-rsa) |
| `useradd: UID 1000 is not unique` | default user holds UID 1000 | [1.2](#12-uid-1000-must-be-free) |
| MariaDB restart fails, `No such file or directory` on `/var/lib/mysql` | Package moved default datadir to `/var/lib/mariadb` (11.8.6-5+) | [9.1](#91-mariadb-1186-5-changed-its-default-datadir) |
| MariaDB init fails, `Permission denied` despite correct ownership | AppArmor `mariadbd` profile denies `dac_override`/`setgid` | [9.2](#92-apparmor-blocks-mariadbd-from-writing-to-varlibmysql) |
| `/usr/bin/python3.14` not found during Deploy Candidate | agent version-resolution bug | [Part 14](#part-14--deploy-candidate) |
| `denied: requested access to the resource` on Docker push, despite working manual `docker login`/`push` | Docker Hub only supports 2-level repo paths; Press builds 3 | [13.1](#131-docker-hub-only-supports-one-level-repository-paths) |
| Agent Job stuck `Undelivered`, no exception | `should_skip_requests()` short-circuits when the target's DNS doesn't resolve | [10.1](#101-exact-records-for-each-serverproxy-serverdatabase-server-doc) / [15.2](#152-job-stuck-undelivered) |
| `401 Unauthenticated` despite correct `agent_password` | Stale gunicorn master serving an old token, or `$` eaten by unquoted shell | [15.3](#153-401-unauthenticated-despite-a-correct-agent_password) |
| `nginx: [emerg] "real_ip_header" directive is duplicate` | Manually added an `include` that's already included elsewhere | [15.4](#154-verify-the-nginx-upstream-was-actually-written) |
| Site HTML loads but all CSS/JS return `404` | `/home/frappe` is `750`; `www-data` not in `frappe` group | [Part 17](#part-17--static-assets-return-404-even-though-the-files-exist) |
| Dashboard is a blank white screen only on some devices | `keepalive_requests` too low for the dashboard's asset count | [Part 18](#part-18--dashboard-white-screen-on-a-fresh-device) |
| New site: `DNS_PROBE_POSSIBLE` / "site can't be reached" | No wildcard DNS for the sites domain | [10.2](#102-wildcard-record-for-customer-sites) |

---

## Production checklist

- [ ] **Domain** with real DNS record control — needed for the exact-match server records, the
      wildcard record, and the DNS-01 challenge every ~90 days
- [ ] **SMTP credentials** — Press sends signup, backup and failure notifications
- [ ] **S3-compatible backup storage** — access key, secret, bucket, region/endpoint
- [ ] **Container registry** credentials, with the Docker Hub path-depth issue resolved either
      way ([13.1](#131-docker-hub-only-supports-one-level-repository-paths))
- [ ] **Git access** (deploy keys or tokens) for any private app repositories

---

## Known open issues

- **`is_upstream_setup` flag** — does not reliably reflect whether upstream registration
  succeeded; verify against the actual generated nginx config instead ([15.4](#154-verify-the-nginx-upstream-was-actually-written)).
- **Agent gunicorn master caching** — any change to `/home/frappe/agent/config.json` requires
  killing the master process outright, not a supervisor-level restart, or the change is silently
  ignored.
- **nginx VTS block / `www-data`-to-`frappe` jammy condition / `python3.14` symlink** — all
  local patches to the Press playbooks; they will need reapplying after a Press update until
  fixed upstream.
- **Docker Hub registry path depth** — unresolved upstream; Press's image-naming assumes a
  registry that supports nested paths (ECR/GHCR/GitLab). Needs either a registry switch or a
  local patch per deployment.
- **Certificate renewal** — the cron renews the files, but the `TLS Certificate` doc content
  must be refreshed separately; worth scripting before the first renewal comes due.
