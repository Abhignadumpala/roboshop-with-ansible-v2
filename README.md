# RoboShop with Ansible v2 (Roles)

The same RoboShop e-commerce app (10 servers) as [v1](https://github.com/Abhignadumpala/roboshop-with-ansible-v1), rebuilt with **Ansible roles**, a shared **common role**, a **handler**, and passwords moved from plain text → **Ansible Vault** → **AWS Secrets Manager**.

| About | |
|---|---|
| **Goal** | Turn 10 separate playbooks into reusable roles run by one entry playbook |
| **Run a server** | `ansible-playbook -e component=<name> main.yaml` |
| **Tools** | Ansible, AWS EC2, Route 53, IAM, Secrets Manager |
| **Builds on** | [roboshop-with-ansible-v1](https://github.com/Abhignadumpala/roboshop-with-ansible-v1) (same servers, same app) |
| **Lab guide** | [LAB.md](LAB.md) |

---

## v1 vs v2

| | v1 (playbooks) | v2 (roles) |
|---|---|---|
| **Entry point** | 10 playbooks: `01-mongodb.yaml` … `10-frontend.yaml` | **1** playbook: `main.yaml` |
| **Pick a server** | Run that server's file | `-e component=catalogue` |
| **Repeated tasks** | Copied into every app playbook (user, /app, download, systemd…) | Written **once** in the `common` role, reused with `include_role` |
| **Files / templates** | All in the repo root | Inside each role's `files/` and `templates/` |
| **nginx restart** | Restarted on every run | Restarted only when the config changes (**handler**) |
| **Passwords** | Plain text in `group_vars` (only the SSH password vaulted) | Vault file → **AWS Secrets Manager** |

> **Interview one-liner:** in v1 every server had its own playbook with the same setup tasks copied; in v2 each server is a role, the shared steps live once in a common role, and one `main.yaml` runs any role with `-e component=<name>`.

---

## What is a role?

**Definition:** a **role** is a folder with a fixed structure that holds everything needed to configure one thing: its tasks, files, templates, variables and handlers. Ansible knows the folder names, so it finds each part automatically.

```
roles/catalogue/
├── tasks/main.yaml          # the steps (Ansible starts here)
├── templates/catalogue.service.j2   # Jinja2 files, filled with variables
├── files/mongo.repo         # copied as-is
├── defaults/main.yaml       # default variables (lowest priority)
├── vars/main.yaml           # role variables (high priority)
├── handlers/main.yaml       # tasks that run only when notified
└── meta/main.yaml           # role dependencies
```

| Folder | What goes inside | Used in this project by |
|---|---|---|
| `tasks/` | The steps. `main.yaml` runs first; other files are called with `tasks_from` | every role |
| `files/` | Files copied as they are (`copy: src: mongo.repo`) | mongodb, catalogue, rabbitmq |
| `templates/` | `.j2` files with `{{ variables }}` (`template:`) | catalogue, user, cart, shipping, payment, frontend |
| `defaults/` | Default values, easy to override | catalogue (demo) |
| `vars/` | Fixed values for the role, hard to override | catalogue (demo) |
| `handlers/` | Tasks that run only when another task notifies them | frontend (restart nginx) |
| `meta/` | Other roles that must run first on the **same host** | catalogue (demo) |

You only create the folders a role needs. Most roles here have only `tasks/`.

---

## How one playbook runs any server

```yaml
- name: configure {{ component }}
  hosts: "{{ component }}"
  become: yes
  roles:
  - "{{ component }}"
```

```
ansible-playbook -e component=catalogue main.yaml
                    │
                    ├──► hosts: catalogue            (group in inventory.ini)
                    └──► roles: catalogue            (folder roles/catalogue/)
                               └──► every {{ component }} inside = catalogue
```

`-e` (extra vars) sets `component` from the command line. It picks the inventory group, the role folder, and fills the names inside the shared tasks (`/tmp/{{ component }}.zip`, `{{ component }}.service`).

---

## The common role

**Definition:** the **common role** holds the steps that several roles share. A role calls only the file it needs with `include_role` + `tasks_from`. It's the Ansible version of `common.sh` in the shell-script project.

| File in `roles/common/tasks/` | What it does | Used by |
|---|---|---|
| `app-setup.yaml` | roboshop user, clean `/app`, download + unzip the code | catalogue, user, cart, shipping, payment |
| `nodejs.yaml` | Node.js 24 + `npm install` | catalogue, user, cart |
| `java.yaml` | Maven + Java 25, build the jar | shipping |
| `python.yaml` | Python 3.14 + pip install into a venv | payment |
| `systemd-setup.yaml` | Copy `{{ component }}.service.j2` → `/etc/systemd/system/` | all 5 app roles |
| `restart.yaml` | daemon-reload + restart + enable | all 5 app roles |

```
roles/cart/tasks/main.yaml
   ├── include_role common → app-setup
   ├── include_role common → nodejs
   ├── include_role common → systemd-setup
   └── include_role common → restart
```

> **Interview one-liner:** the common role keeps the shared setup in one place; each app role calls the pieces it needs with `include_role` and `tasks_from`, so a fix in one file fixes all five services.

---

## Which role uses what

| Role | Runtime | common files used | Own extras |
|---|---|---|---|
| mongodb | — | — | `files/mongo.repo` |
| redis | — | — | — |
| mysql | — | — | — |
| rabbitmq | — | — | `files/rabbitmq.repo` |
| catalogue | Node.js | app-setup, nodejs, systemd-setup, restart | mongosh + load products, `defaults/` `vars/` `meta/` demo |
| user | Node.js | app-setup, nodejs, systemd-setup, restart | — |
| cart | Node.js | app-setup, nodejs, systemd-setup, restart | — |
| shipping | Java | app-setup, java, systemd-setup, restart | load cities into MySQL |
| payment | Python | app-setup, python, systemd-setup, restart | — |
| frontend | nginx | — | `templates/roboshop.conf.j2` + **handler** |

---

## My secrets journey

| Version | Where the passwords lived | Problem |
|---|---|---|
| **v1** | Plain text in `group_vars` (only the SSH password vaulted inline) | Anyone who reads the repo sees them |
| **v2, step 1** | `group_vars/all/vault.yaml`, encrypted with **Ansible Vault** | Every change = decrypt, edit, re-encrypt, push; the vault password has to be shared by hand |
| **v2, step 2** | **AWS Secrets Manager** (`roboshop/dev`, key/value pairs) | — AWS encrypts it (KMS), IAM controls who reads it, nothing secret in Git |

The whole project already runs on AWS, so Secrets Manager plugs in with the Ansible server's IAM role: no extra tool, no password file. **Secret management became easy.**

> **Interview one-liner:** I started with hardcoded passwords, moved them into an encrypted Ansible Vault file, and then to AWS Secrets Manager; Ansible reads them at run time with the `amazon.aws.secretsmanager_secret` lookup and the server's IAM role, so Git has no secrets at all.

---

## Concept notes

| Topic | Notes |
|---|---|
| Roles, folder structure, vars vs defaults, meta, module vs plugin | [devops-notes Day 24](https://github.com/Abhignadumpala/devops-notes/tree/main/day-24-nohup-tags-venv-ansible-roles) |
| Ansible Vault, handlers | [devops-notes Day 25](https://github.com/Abhignadumpala/devops-notes/tree/main/day-25-ansible-vault-roboshop-v2) |
| Secrets Manager, include vs import, dynamic inventory, forks and serial | [devops-notes Day 26](https://github.com/Abhignadumpala/devops-notes/tree/main/day-26-secrets-manager-include-import-dynamic-inventory-parallelism) |

---

## How to run this lab

The full step-by-step guide is in **[LAB.md](LAB.md)**.

```
 v1 servers ──► main.yaml + roles ──► common role ──► handler ──► Vault ──► Secrets Manager ──► http://<your-domain>
```
