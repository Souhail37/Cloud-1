# Cloud-1

This repository deploys WordPress, MariaDB, phpMyAdmin and Nginx to Ubuntu 22.04 with Ansible and Docker Compose. WordPress and database files live in Docker volumes. The services restart after a host reboot. Nginx serves WordPress at `/` and protects phpMyAdmin at `/phpmyadmin/` with HTTP Basic authentication.

## Before deployment

The Ansible control machine needs Ansible and the Docker collection:

```sh
ansible-galaxy collection install -r requirements.yml
```

Each target needs a fresh Ubuntu 22.04-compatible system with SSH, Python and root access. Point a DNS A record at each target before deploying. Configure the cloud provider's firewall or security group to allow inbound TCP 22, 80 and 443 only. The playbook also applies these rules with UFW on the host.

Set `site_domain` in `group_vars/all/main.yml`, or override it per host in `host_vars/<inventory_hostname>.yml`. Each independent site needs its own domain whose DNS record points to that host; ACME HTTP validation must reach the target on port 80. Update `inventory.ini` with the target addresses. The playbook accepts multiple hosts in the `wordpress_hosts` group and Ansible provisions them in parallel using its normal fork setting.

The encrypted `group_vars/all/vault.yml` must define these values:

| Variable | Purpose |
| --- | --- |
| `vault_mariadb_root_password` | MariaDB root password |
| `vault_mariadb_wordpress_password` | WordPress database password |
| `vault_pma_gate_user` | phpMyAdmin Basic Auth user |
| `vault_pma_gate_password` | phpMyAdmin Basic Auth password |
| `vault_letsencrypt_email` | Let's Encrypt contact email |
| `vault_wp_admin_user` | WordPress administrator user |
| `vault_wp_admin_email` | WordPress administrator email |
| `vault_wp_admin_password` | WordPress administrator password |

Keep the Vault password outside Git. For a new installation, create or edit the encrypted file with `ansible-vault`; do not put plaintext credentials in inventory, Compose or `.env`.

## Deploy

From this directory:

```sh
ansible-playbook -i inventory.ini lab.yml --ask-vault-pass
```

The playbook installs Docker, limits inbound ports, starts the Compose stack, obtains a TLS certificate and installs WordPress once. Running the same command again should leave the site working with its existing users, posts and uploaded files. Certbot's timer renews certificates; a deploy hook reloads Nginx after renewal.

## Verify on a target

```sh
cd /opt/cloud-1
docker compose ps
docker volume ls
ufw status verbose
systemctl status certbot.timer
```

Visit `https://<site_domain>/` and `https://<site_domain>/phpmyadmin/`. Check that HTTP redirects to HTTPS, the same WordPress content appears after refresh, and the database has no published host port. Reboot the target and confirm the post, account and media persist. For portability and idempotence, run the playbook twice against a second fresh target with a domain pointing to it.
