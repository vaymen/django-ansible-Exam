# django-ansible

Deploys Django and PostgreSQL on a **single server** with Ansible and Docker Compose. The image (`vaymen/django-app`, tagged with the commit id) is pulled from Docker Hub. Application code: [django-app-Exam](https://github.com/vaymen/django-app-Exam).

```text
playbook.yml
├── roles/docker   install Docker, Compose, Python SDK; start the service
└── roles/app      app directory, env files, compose file, pull image, up, sync DB password
```

## Target host assumptions

- Ubuntu **24.04+** (the `docker-compose-v2` package does not exist in 22.04 or Debian 12).
- SSH access with a sudo-capable user; outbound internet to apt and Docker Hub.
- Controller: `ansible-core` ≥ 2.15 (on Windows use WSL).

## Usage

```bash
ansible-galaxy collection install -r requirements.yml

cp inventory/hosts.ini.example inventory/hosts.ini          # set IP and user
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
$EDITOR group_vars/all/vault.yml                            # db_password, django_secret_key
ansible-vault encrypt group_vars/all/vault.yml

ansible-playbook playbook.yml --ask-vault-pass
# or: ansible-playbook -i inventory/hosts.ini playbook.yml --ask-vault-pass
```

Non-secret settings live in `group_vars/all/main.yml`: `app_image_tag`, `app_port`, `app_allowed_hosts` (defaults to the host address; `*` is rejected), `db_name`, `db_user`.

## Upgrades and rollbacks

Change the tag and run the playbook again:

```bash
ansible-playbook playbook.yml --ask-vault-pass -e app_image_tag=<commit-id>
```

A rollback is the same command with the previous tag.

## Idempotency

A second run should report `changed=0`. The image is pulled by an immutable tag, and templates plus `recreate: auto` only act on real changes. The one task that always runs is the `ALTER ROLE` password sync, which is marked `changed_when: false`. Test with `--check --diff` or by running twice.

## Verification

```bash
curl -i http://<host>:8000/          # welcome page
curl -i http://<host>:8000/readyz    # 200 means Django reaches PostgreSQL
ssh <host> docker compose -f /opt/django-app/docker-compose.yml ps
```

## Running in different environments

| Environment | What to do |
|---|---|
| Test VM | use this README as is |
| Several servers | add hosts under `[django_hosts]`, but each would get its own database, so move the database out (e.g. RDS) and change `DB_HOST` |
| Private registry | run `docker login` on the host (or add a `docker_login` task) |
| Behind a reverse proxy / TLS | bind the port to `127.0.0.1`, put Nginx or Traefik in front, set `DJANGO_BEHIND_PROXY=true` |
| Debian or Ubuntu 22.04 | replace the `docker` role with Docker's official apt repository (`docker-compose-plugin`) |

## Design decisions

- **Compose instead of installing services by hand.** The simplest repeatable setup for one host, and close to how the app runs on Kubernetes.
- **Two env files (`app.env`, `db.env`, mode `0600`).** PostgreSQL only sees `POSTGRES_*`. `$` in secrets is escaped for Compose.
- **Ansible Vault for secrets.** Only `*.example` files are committed.
- **Database password sync.** PostgreSQL reads `POSTGRES_PASSWORD` only when the data volume is first created; without this task, changing `db_password` would silently break the app.
- **Database is not published.** It is reachable only on the Compose network.
- **Possible, but too much here:** Docker Swarm or Nomad, a backup role (`pg_dump` plus cron), Molecule tests for the roles.

## Limitations

No backups, single host (no HA), and the playbook has been syntax-checked but not run against a real server.

## How this was built

Written with the help of an AI coding assistant (Claude Code) and reviewed and tested by hand. [`AGENTS.md`](AGENTS.md) records the conventions and commands an agent must follow when changing this repository.
