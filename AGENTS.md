# AGENTS.md — django-ansible

Ansible playbook deploying Django + PostgreSQL with Docker Compose on one Ubuntu 24.04+ host. Image `docker.io/vaymen/django-app:<commit-id>` is **pulled**, never built here. App code lives in `django-app-Exam`; the Kubernetes variant in `django-helm-Exam`.

## Layout
- `playbook.yml` — roles `docker` then `app`.
- `roles/docker` — installs Docker, `docker-compose-v2`, Python SDK.
- `roles/app` — directory, `app.env` / `db.env` (0600), compose file, image pull, `docker_compose_v2`, DB password sync.
- `group_vars/all/main.yml` — non-secret vars (`app_image_tag`, `app_port`, `app_allowed_hosts`, ...).
- `group_vars/all/vault.yml.example`, `inventory/hosts.ini.example` — templates only.

## Commands
```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i <inventory> playbook.yml --syntax-check
ansible-playbook playbook.yml --ask-vault-pass --check --diff
```
Run the playbook twice: the second run must report `changed=0`.

## Rules
- Use FQCN modules (`ansible.builtin.*`, `community.docker.*`); prefer modules over `command`/`shell`.
- Every task must be idempotent. If a `command` is unavoidable, set `changed_when` explicitly.
- Secrets only via Ansible Vault. Never commit `group_vars/all/vault.yml` or `inventory/hosts.ini` (both are git-ignored). Use `no_log: true` on tasks that handle secrets.
- Keep `app.env` and `db.env` separate; the database container must only receive `POSTGRES_*`.
- `app_allowed_hosts` must not be `*`.
- The image tag is `app_image_tag` (a commit id). Bump it when a new image is pushed.
- Target OS is Ubuntu 24.04+ only; document any change to that assumption in the README.
- README stays in English.
