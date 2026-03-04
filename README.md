# Production WordPress Stack with Ansible

## Problem

Deploying production web stacks manually leads to:

- Configuration drift between environments
- Inconsistent server state after partial failures
- No repeatable path for disaster recovery
- Credentials embedded in configuration files

## Solution

This repository demonstrates a fully automated, idempotent deployment of a production
WordPress stack using Ansible. A single command provisions a server from bare Ubuntu to a
running, TLS-secured, multi-site WordPress installation — and the same command is safe to
re-run on an already-configured server without side effects.

## Stack

| Component             | Role                                                    |
|-----------------------|---------------------------------------------------------|
| Nginx                 | Reverse proxy, static file serving, TLS termination     |
| PHP-FPM 8.2           | Application server via Unix socket                      |
| MariaDB               | Database with hardened installation                     |
| Let's Encrypt/Certbot | Automated TLS certificate provisioning                  |
| systemd               | Service lifecycle management with boot persistence      |
| Ansible Vault         | Secret management — no credentials in repository        |

## Design Decisions

**Runtime secret injection** — credentials are never stored in the repository. Secrets are
passed at runtime via `-e "secret_file=..."` and can be encrypted with Ansible Vault. The
repository contains only a documented example file with placeholder values.

**Idempotent tasks with `creates:`** — slow one-time operations (DH parameter generation,
certificate issuance) use `creates:` to skip execution if output already exists. Re-running
any playbook on a configured server produces no unintended changes.

**Nginx config validation before reload** — every config change is preceded by `nginx -t`.
In `certbot.yaml` this runs both before Certbot executes and after the SSL vhost is deployed,
preventing a broken configuration from ever being applied to a live server.

**MariaDB hardening in code** — `install_wordpress.yaml` runs the equivalent of
`mysql_secure_installation` as an Ansible task: removes anonymous users, drops the test
database, and restricts root login to localhost. Security posture is defined in source
control, not applied manually post-deploy.

**Timestamped backups** — backup filenames include a full timestamp
(`domain-fs-YYYY-MM-DD_HH-MM-SS.tar.gz`), so each run produces a new archive. No previous
backup is overwritten by a subsequent run.

**Backup pull to controller** — backups are transferred to the Ansible control machine via
`rsync` and `fetch`, not left on the target server. Loss of the server does not mean loss of
the backup.

**Pre-restore validation** — `deploy_wordpress_from_backup.yaml` checks that both the
files archive and the database dump exist and that extraction produced a valid WordPress
installation before touching the database. The playbook fails fast rather than leaving the
server in a half-restored state.

## Playbook Reference

| Playbook                               | Purpose                                              |
|----------------------------------------|------------------------------------------------------|
| `install_wordpress.yaml`               | Full stack provisioning (Nginx + PHP-FPM + MariaDB + WordPress) |
| `certbot.yaml`                         | TLS certificate provisioning and Nginx SSL configuration |
| `install_second_wordpress.yaml`        | Add a second isolated WordPress site                 |
| `backup_wordpress.yaml`                | Backup files + database, pull to control machine     |
| `deploy_wordpress_from_backup.yaml`    | Restore a site from backup                           |
| `deploy_second_wordpress_from_backup.yaml` | Restore second site from backup                 |
| `cleanup_host.yaml`                    | Full teardown for reinstallation                     |

## Requirements

- Ansible ≥ 2.10 with `community.mysql` collection installed
- Ubuntu 22.04 server with SSH access
- Ports 80 and 443 open on the target host

## Variables

### `vars.yaml`

| Variable      | Default                    | Description                |
|---------------|----------------------------|----------------------------|
| `php_version` | `8.2`                      | PHP version to install     |
| `backup_dir`  | `/var/backups/wordpress`   | Remote backup directory    |
| `ubuntu_user` | `ubuntu`                   | Server username            |

### Secrets (`vars_secret_example.yaml`)

All sensitive values are passed at runtime. Create your own file based on the example:

```yaml
domain: "example.com"
db_name: "example_db"
db_user: "example_user"
db_password: "change_me"
root_db_password: "change_me"

wordpress_install_dir: "/var/www/html/example"
wordpress_admin_email: "admin@example.com"

# Used only for restore playbooks
wordpress_files_backup: "example-files-backup.tar.gz"
wordpress_db_backup:    "example-db-backup.sql"
```

## Usage

```bash
# 1. Provision full stack
ansible-playbook -i hosts install_wordpress.yaml \
  -e "secret_file=vars_secret.yaml" \
  --vault-password-file ~/.vault_pass.txt

# 2. Obtain TLS certificate
ansible-playbook -i hosts certbot.yaml \
  -e "secret_file=vars_secret.yaml" \
  --vault-password-file ~/.vault_pass.txt

# 3. Backup
ansible-playbook -i hosts backup_wordpress.yaml \
  -e "secret_file=vars_secret.yaml" \
  --vault-password-file ~/.vault_pass.txt

# 4. Restore to a new server
ansible-playbook -i hosts deploy_wordpress_from_backup.yaml \
  -e "secret_file=vars_secret.yaml" \
  --vault-password-file ~/.vault_pass.txt

ansible-playbook -i hosts certbot.yaml \
  -e "secret_file=vars_secret.yaml" \
  --vault-password-file ~/.vault_pass.txt
```

## Repository Structure

```
.
├── install_wordpress.yaml              # Main provisioning playbook
├── certbot.yaml                        # TLS automation
├── install_second_wordpress.yaml       # Multi-site extension
├── backup_wordpress.yaml               # Backup + pull to control machine
├── deploy_wordpress_from_backup.yaml   # Restore from backup
├── cleanup_host.yaml                   # Full teardown
├── vars.yaml                           # Non-secret variables
├── vars_secret_example.yaml            # Secret variable template
├── hosts.example                       # Inventory example
└── templates/
    ├── nginx.conf.j2                   # Base Nginx configuration
    ├── nginx_wordpress.j2              # HTTP vhost (pre-SSL)
    ├── nginx_wordpress_ssl.j2          # HTTPS vhost with redirect
    ├── options-ssl-nginx.conf.j2       # TLS protocol and cipher configuration
    ├── wp-config.php.j2                # WordPress configuration
    ├── fastcgi-php.conf.j2             # FastCGI PHP handler snippet
    ├── fastcgi_params.j2               # FastCGI parameters
    ├── php.ini.j2                      # PHP-FPM tuning
    └── mime.types.j2                   # MIME type mappings
```
