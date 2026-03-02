# WordPress Installation with Nginx, MariaDB, and PHP on Ubuntu 20.04

This repository contains Ansible playbooks and templates to set up one or more WordPress sites on an Ubuntu 20.04 server using Nginx, MariaDB, and PHP. Additionally, it includes a playbook for cleaning up the environment if you need to reinstall, as well as a backup playbook.

## Requirements

- Ansible installed on your control machine
- Ubuntu 20.04 server(s) with SSH access
- Open ports: 80 (HTTP), 443 (HTTPS) for web traffic
- Basic knowledge of Ansible and server configuration

## Variables

### General Variables

- `domain`: The domain name or IP address for your first site.
- `second_domain`: The domain name or IP address for your second site.
- `ubuntu_user`: The username for your Ubuntu server (default: ubuntu).
- `php_version`: Version of PHP to be installed (e.g., "8.2").
- `db_name`: The name of the WordPress database for the first site.
- `db_name_second`: The name of the WordPress database for the second site.
- `db_user`: The username for the WordPress databases.
- `wordpress_install_dir`: Installation directory for the WordPress site.
- `second_wordpress_install_dir`: Installation directory for the second WordPress site.
- `backup_dir`: Directory to store backup files on the remote host.

### Secret Variables

The sensitive data (e.g., passwords) are managed using two separate files:

- `vars_secret.yaml`: Secret variables for the first site.
- `vars_secret_second.yaml`: Secret variables for the second site.

You need to create these files with the following structure:

```yaml
# vars_secret.yaml
db_password: your_db_password
root_db_password: your_root_db_password
wordpress_admin_password: your_wordpress_admin_password
wordpress_admin_email: your_email

# vars_secret_second.yaml
db_password: your_db_password_second
root_db_password: your_root_db_password_second
wordpress_admin_password: your_wordpress_admin_password_second
wordpress_admin_email: your_email_second
```


## Usage

### Install the First WordPress Site
To install the first WordPress site:

```bash
ansible-playbook -i hosts install_wordpress.yaml --vault-password-file ~/ansible/.vault_pass.txt
```
This playbook sets up the first WordPress site at the specified domain. It installs Nginx, MariaDB, PHP, and configures the server.

Obtain SSL Certificates
To secure your site with SSL, run:

```bash
ansible-playbook -i hosts certbot.yaml --vault-password-file ~/ansible/.vault_pass.txt
This playbook uses Certbot to obtain and install an SSL certificate for your WordPress site.
```

Install the Second WordPress Site
To install a second WordPress site on the same server:


```bash
ansible-playbook -i hosts install_second_wordpress.yaml --vault-password-file ~/ansible/.vault_pass.txt
```

This playbook sets up a second WordPress site with its own database and user credentials, without interfering with the first site.

Create a Backup of the WordPress Site
To create a backup of the WordPress site:

```bash
ansible-playbook -i hosts backup_wordpress.yaml --vault-password-file ~/ansible/.vault_pass.txt
```

This playbook will backup both the WordPress files and database for the site specified in vars.yaml and store the backups on the Ansible host.

Restore WordPress from Backup
To restore a WordPress site from a backup:

```bash
ansible-playbook -i hosts deploy_wordpress_from_backup.yaml --vault-password-file ~/ansible/.vault_pass.txt
```

ansible-playbook -i hosts certbot.yaml --vault-password-file ~/ansible/.vault_pass.txt
(For each site, remember to use the appropriate vars_secret.yaml file.)

Clean Up the Host for Reinstallation
If you need to clean up the server for a fresh installation:

```bash
ansible-playbook -i hosts cleanup_host.yaml --vault-password-file ~/ansible/.vault_pass.txt
```

This playbook stops and removes all installed services (Nginx, MariaDB, PHP), and deletes configuration and data files, returning the server to a clean state.

Templates Explanation
fastcgi-php.conf.j2: Configuration snippet for handling PHP requests in Nginx.
fastcgi_params.j2: Defines parameters for FastCGI processes.
mime.types.j2: Provides MIME type mappings for Nginx.
nginx.conf.j2: Basic configuration file for Nginx.
nginx_wordpress.j2: Nginx configuration for serving the first WordPress site (HTTP only).
nginx_wordpress_ssl.j2: Nginx configuration for serving the first WordPress site with SSL.
nginx_wordpress_ssl_second.j2: Nginx configuration for serving the second WordPress site with SSL.
wp-config.php.j2: The main configuration file for WordPress, customized for the first site.
wp-config_second.j2: Configuration file for the second WordPress site.
Multiple Sites
You can use these playbooks to manage multiple WordPress sites on the same server. Each site will have its own database, user credentials, and SSL certificate. The provided templates ensure that configurations for each site do not conflict.

Example Workflow
Install the first WordPress site:

```bash
ansible-playbook -i hosts install_wordpress.yaml --vault-password-file ~/ansible/.vault_pass.txt
```

Obtain an SSL certificate:

```bash
ansible-playbook -i hosts certbot.yaml --vault-password-file ~/ansible/.vault_pass.txt
Install the second WordPress site:
```

Install the second WordPress site:
```bash
ansible-playbook -i hosts install_second_wordpress.yaml --vault-password-file ~/ansible/.vault_pass.txt
```
Create a backup of the site:

```bash
ansible-playbook -i hosts backup_wordpress.yaml --vault-password-file ~/ansible/.vault_pass.txt
```
Restore procedure:

```bash
ansible-playbook -i hosts deploy_wordpress_from_backup.yaml --vault-password-file ~/ansible/.vault_pass.txt
ansible-playbook -i hosts certbot.yaml --vault-password-file ~/ansible/.vault_pass.txt
```
(For each site, remember to use the appropriate vars_secret.yaml file.)

Clean Up the Host for Reinstallation:
If you need to clean up the server for a fresh installation:

```bash
ansible-playbook -i hosts cleanup_host.yaml --vault-password-file ~/ansible/.vault_pass.txt
```
This playbook stops and removes all installed services (Nginx, MariaDB, PHP), and deletes configuration and data files, returning the server to a clean state.

## Secrets

This repository does not contain real secrets.

Create your own secret variables file based on:

`vars_secret_example.yaml`

Run playbooks like this:

```bash
ansible-playbook install_wordpress.yaml \
  -e "secret_file=vars_secret.yaml" \
  --vault-password-file ~/.vault_pass.txt
```
