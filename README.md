# healthchecks

Native Ansible role for [Healthchecks](https://healthchecks.io) on Debian.
No Docker or Podman: a Python virtualenv, PostgreSQL (or SQLite), gunicorn,
and systemd units for the web app, `sendalerts`, `sendreports` and the
database cleanup timer.

## What it does

1. Installs Debian packages (Python 3.12+, build deps, libpq, PostgreSQL,
   `python3-psycopg2`).
2. Creates the `healthchecks` system user and `/opt/healthchecks`.
3. Creates a local PostgreSQL role/database (or uses SQLite).
4. Clones a pinned Healthchecks tag into `/opt/healthchecks/app`, creates a
   virtualenv, installs `requirements.txt` + gunicorn.
5. Runs `migrate`, `collectstatic`, `compress`, and creates the first admin.
6. Writes `/etc/healthchecks/healthchecks.env` (0600) with the site, database
   and SMTP configuration.
7. Installs and starts:
   - `healthchecks-web.service` (gunicorn on `127.0.0.1:8000`)
   - `healthchecks-sendalerts.service`
   - `healthchecks-sendreports.service`
   - `healthchecks-prune.service` + `.timer` (daily cleanup)

The role does **not** install a reverse proxy. Put nginx/Caddy in front and
proxy `https://health.example.org` to `127.0.0.1:8000`; set
`SECURE_PROXY_SSL_HEADER` is already emitted as
`HTTP_X_FORWARDED_PROTO,https`.

## Requirements

- Debian 12 (bookworm) or 13 (trixie) with systemd.
- Python 3.12+ (Debian 13 ships 3.13).
- `community.postgresql` collection for the PostgreSQL tasks.

```bash
ansible-galaxy collection install community.postgresql
```

## Example playbook

```yaml
- name: "Deploy Healthchecks"
  hosts: "healthchecks"
  become: true
  roles:
    - role: "healthchecks"
      vars:
        healthchecks_site_root: "https://health.example.org"
        healthchecks_site_name: "Example Healthchecks"
        healthchecks_allowed_hosts: ["health.example.org"]
        healthchecks_secret_key: "{{ vault_healthchecks_secret_key }}"
        healthchecks_admin_email: "health@example.org"
        healthchecks_admin_password: "{{ vault_healthchecks_admin_password }}"
        healthchecks_db_password: "{{ vault_healthchecks_db_password }}"
        healthchecks_email_host: "mail.example.org"
        healthchecks_email_port: 25
        healthchecks_email_use_tls: false
        healthchecks_default_from_email: "healthchecks@example.org"
```

The first admin is created idempotently from `healthchecks_admin_email` /
`healthchecks_admin_password`; change the password in the UI afterwards.

## Important variables

| Variable | Default | Meaning |
|---|---|---|
| `healthchecks_version` | `v5.0.3` | Pinned upstream tag |
| `healthchecks_site_root` | `https://health.example.org` | Public URL |
| `healthchecks_allowed_hosts` | `["health.example.org"]` | Django ALLOWED_HOSTS |
| `healthchecks_secret_key` | required | Django SECRET_KEY |
| `healthchecks_admin_email` | `admin@example.org` | First admin |
| `healthchecks_admin_password` | required | First admin password |
| `healthchecks_db_engine` | `postgresql` | `postgresql` or `sqlite` |
| `healthchecks_db_name` | `healthchecks` | DB name (SQLite: file path) |
| `healthchecks_db_user` | `healthchecks` | PostgreSQL role |
| `healthchecks_db_password` | required for PG | PostgreSQL password |
| `healthchecks_email_host` | `localhost` | SMTP host |
| `healthchecks_email_port` | `25` | SMTP port |
| `healthchecks_email_use_tls` | `false` | STARTTLS |
| `healthchecks_email_use_ssl` | `false` | Implicit TLS |
| `healthchecks_gunicorn_bind` | `127.0.0.1:8000` | Listen address |
| `healthchecks_gunicorn_workers` | `3` | Gunicorn workers |
| `healthchecks_manage_app` | `true` | Clone/install/migrate |
| `healthchecks_manage_database` | `true` | Create local DB |
| `healthchecks_manage_services` | `true` | Enable/start units |

See `defaults/main.yml` and `meta/argument_specs.yml` for the full list.

## Molecule

```bash
molecule test -s default
```

The scenario runs unprivileged on the controller, skips packages/users/
database/app/services, and verifies the rendered environment file and
systemd units.

## License

MIT.
