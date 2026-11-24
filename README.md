# Standalone MySQL + WordPress on multiple servers (Ansible)

Every server in `[wordpress]` gets its **own** MySQL and its **own** WordPress site.
No database is shared; each stack is independent.

```
  wp1                 wp2                 wp3
  ┌───────────┐       ┌───────────┐       ┌───────────┐
  │ Apache+PHP│       │ Apache+PHP│       │ Apache+PHP│
  │ WordPress │       │ WordPress │       │ WordPress │
  │     │     │       │     │     │       │     │     │
  │ MySQL     │       │ MySQL     │       │ MySQL     │
  │ (127.0.0.1)│      │ (127.0.0.1)│      │ (127.0.0.1)│
  └───────────┘       └───────────┘       └───────────┘
```

Target OS: Ubuntu 22.04 / 24.04.

## Layout

```
site.yml                     # one play: common → mysql → wordpress, on every host
inventory/hosts.ini          # servers + per-host wp_domain
group_vars/all/vars.yml      # defaults (DB name, admin user, firewall...)
group_vars/all/vault.yml     # passwords + salt seed (encrypt this)
collections/requirements.yml # ansible.mysql, community.general
roles/common                 # apt, ufw, timezone
roles/mysql                  # MySQL 8, bound to 127.0.0.1, local DB + user
roles/wordpress              # Apache/PHP, WP core, wp-config, WP-CLI install
```

## Usage

```bash
ansible-galaxy collection install -r collections/requirements.yml

# 1. Add your servers + domains to inventory/hosts.ini
# 2. Set passwords and a long random vault_wp_salt_seed, then encrypt:
ansible-vault encrypt group_vars/all/vault.yml

ansible wordpress -m ping
ansible-playbook site.yml --ask-vault-pass

# Just one server:
ansible-playbook site.yml --limit wp2 --ask-vault-pass
```

Point each domain's A record at its own server's IP.

## Per-server settings

Anything in `group_vars/all/vars.yml` can be overridden per host, either inline in
the inventory (`wp_domain=...`) or in `host_vars/<host>.yml`:

```yaml
# host_vars/wp2.yml   (encrypt with ansible-vault if it holds passwords)
wp_site_title: "Client Two"
wp_admin_email: it@client2.co.ke
vault_wp_db_password: "a-different-password"
vault_wp_admin_password: "another-one"
```

## How it works

- MySQL listens on `127.0.0.1` only. Port 3306 is never opened in the firewall, and
  the WordPress DB user exists only as `wp_user@localhost`.
- Each site's WordPress keys/salts are a SHA-512 of `vault_wp_salt_seed` + hostname +
  key name. Every server gets different salts, and they're the same on every run, so
  nothing is stored on disk (this works on Semaphore/CI, where the repo is re-cloned).
  Changing the seed logs out every user on every site.
- `wp core install` runs on each server only if its own DB has no WordPress
  tables yet, so re-running the playbook is safe.
- ufw: SSH, 80 and 443 open; everything else denied.

## Semaphore

Repository → this repo; Inventory → Static (paste hosts.ini) or File `inventory/hosts.ini`;
Environment → `{}`; Task template → playbook `site.yml` with the vault password key attached.
Semaphore installs `collections/requirements.yml` automatically.

## HTTPS

Not automated. On each server: `apt install certbot python3-certbot-apache &&
certbot --apache -d <domain>`, then set `wp_site_url: https://<domain>` for that host.
# ansible-wordpress
