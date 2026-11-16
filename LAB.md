# RoboShop with Ansible v2: Lab Step by Step

[← Back to README](README.md)

> **In short:** we reuse the 10 RoboShop servers and the Ansible server from [v1](https://github.com/Abhignadumpala/roboshop-with-ansible-v1/blob/main/LAB.md). We write **one** entry playbook (`main.yaml`), turn each v1 playbook into a **role**, move the shared steps into a **common role**, add a **handler** to frontend, and move the passwords to **Ansible Vault** and then to **AWS Secrets Manager**. After each role: run it → check it → push.

| What | Details |
|---|---|
| **Ansible server** | RHEL 9 server from v1, with the IAM role (EC2 + Route 53), Ansible, boto3, sshpass and the collections already installed |
| **RoboShop servers** | The 10 servers from v1 (stopped → start them in Part 1) |
| **Repo** | `~/roboshop-with-ansible-v2` on the Ansible server |
| **Run a role** | `ansible-playbook -e component=<name> main.yaml` |
| **Passwords in this guide** | Shown as `<placeholders>`. Type your real values only in your terminal |

## Table of Contents

**Part 1: Start the servers**
- [Step 1: Start the stopped servers (laptop)](#step-1-start-the-stopped-servers-laptop)
- [Step 2: Log in to the Ansible server](#step-2-log-in-to-the-ansible-server)
- [Step 3: Fix the frontend public DNS record](#step-3-fix-the-frontend-public-dns-record)
- [Step 4: Check Ansible can reach every server](#step-4-check-ansible-can-reach-every-server)

**Part 2: Project setup**
- [Step 5: Clone the v2 repo](#step-5-clone-the-v2-repo)
- [Step 6: Copy the files that don't change](#step-6-copy-the-files-that-dont-change)
- [Step 7: Turn group_vars into a folder](#step-7-turn-group_vars-into-a-folder)
- [Step 8: Push the setup files](#step-8-push-the-setup-files)

**Part 3: The entry playbook and the first role**
- [Step 9: Create main.yaml](#step-9-create-mainyaml)
- [Step 10: Create the mongodb role](#step-10-create-the-mongodb-role)
- [Step 11: Run the mongodb role](#step-11-run-the-mongodb-role)

**Part 4: The other database roles**
- [Step 12: redis role](#step-12-redis-role)
- [Step 13: mysql role (with an idempotency check)](#step-13-mysql-role-with-an-idempotency-check)
- [Step 14: rabbitmq role (no hardcoded password)](#step-14-rabbitmq-role-no-hardcoded-password)

**Part 5: The common role**
- [Step 15: Find the repeated tasks](#step-15-find-the-repeated-tasks)
- [Step 16: app-setup.yaml](#step-16-app-setupyaml)
- [Step 17: nodejs.yaml, java.yaml, python.yaml](#step-17-nodejsyaml-javayaml-pythonyaml)
- [Step 18: systemd-setup.yaml and restart.yaml](#step-18-systemd-setupyaml-and-restartyaml)

**Part 6: catalogue (the first app role)**
- [Step 19: Create the catalogue role](#step-19-create-the-catalogue-role)
- [Step 20: Run and check catalogue](#step-20-run-and-check-catalogue)
- [Step 21: defaults vs vars](#step-21-defaults-vs-vars)
- [Step 22: meta dependencies](#step-22-meta-dependencies)

**Part 7: user, cart, shipping, payment**
- [Step 23: user and cart roles](#step-23-user-and-cart-roles)
- [Step 24: shipping role (Java + MySQL data)](#step-24-shipping-role-java--mysql-data)
- [Step 25: payment role (Python)](#step-25-payment-role-python)

**Part 8: frontend with a handler**
- [Step 26: Create the frontend role with notify](#step-26-create-the-frontend-role-with-notify)
- [Step 27: Add the handler](#step-27-add-the-handler)
- [Step 28: Run frontend twice and watch the handler](#step-28-run-frontend-twice-and-watch-the-handler)

**Part 9: Passwords → Ansible Vault**
- [Step 29: Create the encrypted vault.yaml](#step-29-create-the-encrypted-vaultyaml)
- [Step 30: Point main.yaml to the vault_ variables](#step-30-point-mainyaml-to-the-vault_-variables)
- [Step 31: Run and check Git has no plain passwords](#step-31-run-and-check-git-has-no-plain-passwords)

**Part 10: Vault → AWS Secrets Manager**
- [Step 32: Store the passwords in Secrets Manager](#step-32-store-the-passwords-in-secrets-manager)
- [Step 33: Allow the Ansible server's role to read the secret](#step-33-allow-the-ansible-servers-role-to-read-the-secret)
- [Step 34: Read the secret from the Ansible server](#step-34-read-the-secret-from-the-ansible-server)
- [Step 35: Point group_vars to Secrets Manager](#step-35-point-group_vars-to-secrets-manager)
- [Step 36: Remove the vault](#step-36-remove-the-vault)

**Part 11: Open the shop**
- [Step 37: Run every role in order](#step-37-run-every-role-in-order)
- [Step 38: Open the website and check the APIs](#step-38-open-the-website-and-check-the-apis)

**Part 12: Clean up**
- [Step 39: Stop or delete the servers](#step-39-stop-or-delete-the-servers)

**Wrap-up**
- [Troubleshooting](#troubleshooting)
- [What I learned](#what-i-learned)
- [Interview questions](#interview-questions)

---

# Part 1: Start the servers

## Step 1: Start the stopped servers (laptop)

```bash
aws ec2 describe-instances --filters "Name=instance-state-name,Values=stopped" --query "Reservations[].Instances[].[InstanceId,Tags[?Key=='Name']|[0].Value]" --output table
```

**What it does:** lists the stopped servers with their names.
**Expected:** 11 rows: the Ansible server + the 10 RoboShop servers ✅.

```bash
aws ec2 start-instances --instance-ids $(aws ec2 describe-instances --filters "Name=instance-state-name,Values=stopped" --query "Reservations[].Instances[].InstanceId" --output text)
```

**What it does:** starts all of them. (Or: EC2 console → select all → Instance state → Start.)
**Expected:** each instance moves to `pending` → `running` ✅.

---

## Step 2: Log in to the Ansible server

The public IP changes after a stop/start, so get the new one:

```bash
aws ec2 describe-instances --filters "Name=tag:Name,Values=ansible-server" "Name=instance-state-name,Values=running" --query "Reservations[].Instances[].PublicIpAddress" --output text
```

```bash
ssh ec2-user@<ansible-server-public-ip>
```

Password: `DevOps321`

---

## Step 3: Fix the frontend public DNS record

**Why:** the private-IP records (`mongodb.<domain>` …) still work after a restart, because private IPs don't change. The **apex record** (`<domain>`) points to frontend's **public** IP, and that changed.

```bash
cd ~/roboshop-with-ansible-v1
```

```bash
ansible-playbook roboshop.yaml -e "instances=['frontend']"
```

**What it does:** finds the frontend server and updates both of its records.
**Expected:** `changed=1` (the apex record now has the new public IP) ✅.

---

## Step 4: Check Ansible can reach every server

```bash
ansible all -m ping
```

**What it does:** logs in to every server in `inventory.ini` and checks Python works.
**Expected:** `"ping": "pong"` from all 10 servers + localhost ✅.

---

# Part 2: Project setup

## Step 5: Clone the v2 repo

```bash
cd ~ && git clone https://github.com/Abhignadumpala/roboshop-with-ansible-v2.git
```

```bash
cd ~/roboshop-with-ansible-v2
```

**Expected:** the folder has `README.md` and `LAB.md` ✅.

> Already cloned it before the notes were pushed? Run `git pull` first.

---

## Step 6: Copy the files that don't change

```bash
cp ../roboshop-with-ansible-v1/{ansible.cfg,inventory.ini,roboshop.yaml} .
```

| File | Why it's the same as v1 |
|---|---|
| `ansible.cfg` | Same inventory, log path and vault password file |
| `inventory.ini` | Same 10 groups and DNS names |
| `roboshop.yaml` | Same EC2 + Route 53 playbook; roles don't change how servers are created |

---

## Step 7: Turn group_vars into a folder

```bash
mkdir -p group_vars/all roles
```

```bash
cp ../roboshop-with-ansible-v1/group_vars/all.yaml group_vars/all/main.yaml
```

**Why a folder?** `group_vars/all.yaml` (a file) and `group_vars/all/` (a folder) mean the same to Ansible: variables for **all** hosts. A folder lets us keep **two** files for the same group: `main.yaml` (normal values) and, in Part 9, `vault.yaml` (encrypted secrets). Ansible loads every file in the folder.

```
group_vars/
└── all/
    ├── main.yaml     # hosts, users, references to secrets
    └── vault.yaml    # (Part 9) encrypted passwords
```

> We don't copy v1's `mongodb.yaml`, `mysql.yaml`, `cart.yaml`: the roles don't use them (`mysql_root_password` is already in `all.yaml`).

---

## Step 8: Push the setup files

```bash
git add . && git commit -m "v2: setup files" && git push
```

---

# Part 3: The entry playbook and the first role

## Step 9: Create main.yaml

```bash
vim main.yaml
```

```yaml
- name: configure {{ component }}
  hosts: "{{ component }}"
  become: yes
  roles:
  - "{{ component }}"
```

| Line | Meaning |
|---|---|
| `hosts: "{{ component }}"` | Run on the inventory group with that name |
| `roles: - "{{ component }}"` | Run the role folder with that name, from `roles/` next to `main.yaml` |
| `component` | Not written in any file: we pass it with `-e component=<name>` |

**Why:** this one file replaces `01-mongodb.yaml` … `10-frontend.yaml`. Each server's steps now live in its role.

---

## Step 10: Create the mongodb role

```bash
mkdir -p roles/mongodb/tasks roles/mongodb/files
```

```bash
cp ../roboshop-with-ansible-v1/mongo.repo roles/mongodb/files/
```

Now `roles/mongodb/tasks/main.yaml`. Take v1's `01-mongodb.yaml` and:
1. Delete the play lines at the top (`- name`, `hosts`, `become`, `tasks:`). `main.yaml` has them now.
2. Move every task **2 spaces left**, so each `- name:` starts at column 0.

This command does both. Check what it prints, then save it:

```bash
sed -n '/tasks:/,$p' ../roboshop-with-ansible-v1/01-mongodb.yaml | tail -n +2 | sed 's/^  //'
```

```bash
sed -n '/tasks:/,$p' ../roboshop-with-ansible-v1/01-mongodb.yaml | tail -n +2 | sed 's/^  //' > roles/mongodb/tasks/main.yaml
```

It should look like this:

```yaml
- name: copy mongo repo
  copy:
    src: mongo.repo            # found in roles/mongodb/files/
    dest: /etc/yum.repos.d/mongo.repo

- name: install mongodb server
  package:
    name: mongodb-org
    state: present

- name: start and enable mongodb
  service:
    name: mongod
    state: started
    enabled: yes

- name: allow remote connections
  replace:
    path: /etc/mongod.conf
    regexp: '127.0.0.1'
    replace: '0.0.0.0'

- name: restart mongod
  service:
    name: mongod
    state: restarted
```

> **Where does `copy` look for `mongo.repo`?** In a role, `src:` without a path is searched in that role's `files/` folder. `template` searches `templates/`. That's why the repo files move into the roles.

---

## Step 11: Run the mongodb role

```bash
ansible-playbook -e component=mongodb main.yaml
```

**Expected:** `failed=0` ✅. MongoDB is already installed from v1, so most tasks say `ok`; only the restart says `changed`.

```bash
git add . && git commit -m "v2: main.yaml + mongodb role" && git push
```

---

# Part 4: The other database roles

Same pattern for each: create the folder → convert the v1 playbook → run → push.

## Step 12: redis role

```bash
mkdir -p roles/redis/tasks
```

```bash
sed -n '/tasks:/,$p' ../roboshop-with-ansible-v1/02-redis.yaml | tail -n +2 | sed 's/^  //' > roles/redis/tasks/main.yaml
```

```bash
ansible-playbook -e component=redis main.yaml
```

**Expected:** `failed=0` ✅.

---

## Step 13: mysql role (with an idempotency check)

```bash
mkdir -p roles/mysql/tasks
```

```bash
sed -n '/tasks:/,$p' ../roboshop-with-ansible-v1/03-mysql.yaml | tail -n +2 | sed 's/^  //' > roles/mysql/tasks/main.yaml
```

**Problem:** MySQL already has a root password from v1. `mysql_secure_installation --set-root-pass` tries to log in **without** a password, so the second run fails with `Access denied`.

**Fix:** check first, and set the password only if logging in with it fails. Replace the last task with these two:

```yaml
- name: check root password already set
  command: mysql -h {{ mysql_host }} -u root -p{{ mysql_root_password }} -e "SELECT 1"
  register: root_check
  ignore_errors: true
  no_log: true

- name: setup mysql root password
  command: mysql_secure_installation --set-root-pass "{{ mysql_root_password }}"
  when: root_check.rc != 0
  no_log: true
```

| Line | Meaning |
|---|---|
| `-h {{ mysql_host }}` | Log in **over the network**, the same way shipping does. Without `-h` it logs in as `root@localhost`, which has **no** password here, so the check always fails |
| `register: root_check` | Save the result (`rc` = exit code) |
| `ignore_errors: true` | A failed login is expected on a new server; don't stop |
| `when: root_check.rc != 0` | Set the password only when the login failed |
| `no_log: true` | Don't print the command (it has the password) |

Also delete the commented `ansible.mysql.mysql_user` block from v1.

```bash
ansible-playbook -e component=mysql main.yaml
```

**Expected:** `failed=0`, and `setup mysql root password` is **skipping** ✅. Idempotent.

> **What I found:** MySQL has **two** root users. `root@%` (network login) has the password; `root@localhost` (login on the MySQL server itself) has **none**. `mysql_secure_installation --set-root-pass` set the password on the network user only. So a check without `-h` failed every run and the password task ran every time. Check it yourself:
>
> ```bash
> ansible mysql -b -m command -a "mysql -e \"SELECT user,host,plugin FROM mysql.user WHERE user='root'\""
> ```

---

## Step 14: rabbitmq role (no hardcoded password)

```bash
mkdir -p roles/rabbitmq/tasks roles/rabbitmq/files
```

```bash
cp ../roboshop-with-ansible-v1/rabbitmq.repo roles/rabbitmq/files/
```

```bash
sed -n '/tasks:/,$p' ../roboshop-with-ansible-v1/04-rabbitmq.yaml | tail -n +2 | sed 's/^  //' > roles/rabbitmq/tasks/main.yaml
```

In v1 the user and password were typed straight into the task. Change them to the variables that already exist in `group_vars/all/main.yaml`:

```yaml
- name: create rabbitmq user
  community.rabbitmq.rabbitmq_user:
    user: "{{ rabbitmq_user }}"
    password: "{{ rabbitmq_password }}"
    vhost: /
    configure_priv: .*
    read_priv: .*
    write_priv: .*
    state: present
```

**Why:** the password now lives in **one** place (group_vars). In Parts 9–10 we move it to Vault and Secrets Manager without touching the role again.

```bash
ansible-playbook -e component=rabbitmq main.yaml
```

```bash
git add . && git commit -m "v2: redis, mysql, rabbitmq roles" && git push
```

---

# Part 5: The common role

## Step 15: Find the repeated tasks

Open `05-catalogue.yaml`, `08-shipping.yaml` and `09-payment.yaml` from v1 side by side. The same blocks show up in all of them:

| Block | catalogue | user | cart | shipping | payment |
|---|---|---|---|---|---|
| roboshop user, clean /app, download, unzip | ✅ | ✅ | ✅ | ✅ | ✅ |
| Node.js + npm install | ✅ | ✅ | ✅ | | |
| Maven + Java + build jar | | | | ✅ | |
| Python + pip in venv | | | | | ✅ |
| copy `<name>.service` | ✅ | ✅ | ✅ | ✅ | ✅ |
| daemon-reload + restart | ✅ | ✅ | ✅ | ✅ | ✅ |

Each block goes into **one file** in the common role. The service name is replaced with `{{ component }}`, so one file works for every service.

```bash
mkdir -p roles/common/tasks
```

> The common role has **no** `main.yaml`: nobody runs it as a whole. Roles call one file at a time with `tasks_from`.

---

## Step 16: app-setup.yaml

```bash
vim roles/common/tasks/app-setup.yaml
```

```yaml
- name: create roboshop system user
  user:
    name: roboshop
    create_home: false
    home: /app
    system: true
    shell: /sbin/nologin
    comment: "roboshop system user"

- name: remove existing app directory
  file:
    path: /app
    state: absent

- name: remove {{ component }}.zip
  file:
    path: /tmp/{{ component }}.zip
    state: absent

- name: create app directory
  file:
    path: /app
    state: directory

- name: download {{ component }} application
  get_url:
    url: https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/{{ component }}-v4.zip
    dest: /tmp/{{ component }}.zip

- name: unzip {{ component }}
  unarchive:
    src: /tmp/{{ component }}.zip
    dest: /app
    remote_src: yes
```

**What changed from v1:** every `catalogue` became `{{ component }}`. When the cart role calls this file, `component` is `cart`, so it downloads `cart-v4.zip`.

---

## Step 17: nodejs.yaml, java.yaml, python.yaml

```bash
vim roles/common/tasks/nodejs.yaml
```

```yaml
- name: disable default nodejs
  command: dnf module disable nodejs -y

- name: enable nodejs24
  command: dnf module enable nodejs:24 -y

- name: install nodejs
  dnf:
    name: nodejs
    state: present

- name: install dependencies
  community.general.npm:
    path: /app
```

```bash
vim roles/common/tasks/java.yaml
```

```yaml
- name: enable maven 3.9
  command: dnf module enable maven:3.9 -y

- name: install maven and java25
  dnf:
    name:
    - maven
    - java-25-openjdk-devel
    - maven-openjdk25    # so we don't need to set JAVA_HOME
    - mysql
    state: present

- name: build {{ component }}
  command: mvn clean package
  args:
    chdir: /app

- name: move {{ component }} jar file
  command: mv target/{{ component }}.jar {{ component }}.jar
  args:
    chdir: /app
```

```bash
vim roles/common/tasks/python.yaml
```

```yaml
- name: install python3.14
  dnf:
    name:
    - python3.14
    - python3.14-pip
    state: present

- name: install dependencies in virtual environment
  ansible.builtin.pip:
    requirements: /app/requirements.txt
    virtualenv: /app/.venv
    virtualenv_command: python3.14 -m venv
```

> Leave out v1's commented `JAVA_HOME` tasks and the `mvn --version` debug tasks. They were for finding a problem, not part of the setup.

---

## Step 18: systemd-setup.yaml and restart.yaml

```bash
vim roles/common/tasks/systemd-setup.yaml
```

```yaml
- name: copy {{ component }} service
  template:
    src: "{{ component }}.service.j2"
    dest: "/etc/systemd/system/{{ component }}.service"
```

> **Where does this find `cart.service.j2`?** The template stays in the **calling role's** `templates/` folder (`roles/cart/templates/`). With `include_role`, Ansible searches the common role's `templates/` **and** the calling role's. So each app role keeps its own service file.

```bash
vim roles/common/tasks/restart.yaml
```

```yaml
- name: daemon reload
  systemd_service:
    daemon_reload: true

- name: start and enable {{ component }}
  service:
    name: "{{ component }}"
    state: restarted
    enabled: yes
```

Nothing to run yet: the common role only runs when another role calls it.

```bash
git add . && git commit -m "v2: common role" && git push
```

---

# Part 6: catalogue (the first app role)

## Step 19: Create the catalogue role

```bash
mkdir -p roles/catalogue/{tasks,templates,files}
```

```bash
cp ../roboshop-with-ansible-v1/catalogue.service.j2 roles/catalogue/templates/
```

```bash
cp ../roboshop-with-ansible-v1/mongo.repo roles/catalogue/files/
```

```bash
vim roles/catalogue/tasks/main.yaml
```

```yaml
- name: application setup
  include_role:
    name: common
    tasks_from: app-setup

- name: nodejs setup
  include_role:
    name: common
    tasks_from: nodejs

- name: systemd setup
  include_role:
    name: common
    tasks_from: systemd-setup

- name: copy mongo repo
  copy:
    src: mongo.repo
    dest: /etc/yum.repos.d/mongo.repo

- name: install mongodb client
  dnf:
    name: mongodb-mongosh
    state: present

- name: check products loaded or not
  command: mongosh --host "{{ mongodb_host }}" --quiet --eval 'db.getSiblingDB("catalogue").products.countDocuments()'
  register: product_count

- name: load the products
  shell: mongosh --host "{{ mongodb_host }}" < /app/db/master-data.js
  when: product_count.stdout | int == 0

- name: restart application
  include_role:
    name: common
    tasks_from: restart
```

| Task | Comes from |
|---|---|
| app-setup, nodejs, systemd-setup, restart | **common role** (shared) |
| mongo repo, mongosh, load products | **catalogue role** (only catalogue needs these) |

**`include_role` + `tasks_from`:** run only that one file of the common role, right here, with the same `component` variable.

---

## Step 20: Run and check catalogue

```bash
ansible-playbook -e component=catalogue main.yaml
```

**Expected:** `failed=0` ✅. In the output, the common tasks show with the role name in front: `TASK [common : download catalogue application]`. `load the products` is **skipping** (products are already in MongoDB from v1).

```bash
curl -s http://catalogue.<your-domain>:8080/health
```

**Expected:** `{"app":"OK","mongo":true}` ✅.

---

## Step 21: defaults vs vars

A small demo to see which one wins.

```bash
mkdir -p roles/catalogue/{defaults,vars}
```

```bash
vim roles/catalogue/defaults/main.yaml
```

```yaml
course: devops
duration: 100hrs
```

```bash
vim roles/catalogue/vars/main.yaml
```

```yaml
course: ansible
duration: 10hrs
```

Add this as the **first** task in `roles/catalogue/tasks/main.yaml`:

```yaml
- name: print variables
  debug:
    msg: "COURSE: {{ course }}, DURATION: {{ duration }}"
```

```bash
ansible-playbook -e component=catalogue main.yaml
```

**Expected:** `COURSE: ansible, DURATION: 10hrs` ✅. **vars** beats **defaults**.

```bash
ansible-playbook -e component=catalogue -e course=terraform main.yaml
```

**Expected:** `COURSE: terraform` ✅. `-e` beats everything.

| Priority (low → high) |
|---|
| role `defaults/` → `group_vars` → play `vars` → role `vars/` → `set_fact` → `-e` |

**When to use which:** `defaults/` for values users **should** change (port, version). `vars/` for values the role needs fixed.

---

## Step 22: meta dependencies

**Definition:** `meta/main.yaml` lists roles that must run **before** this role, **on the same host**.

```bash
mkdir -p roles/example/tasks roles/catalogue/meta
```

```bash
vim roles/example/tasks/main.yaml
```

```yaml
- name: this is the dependency role
  debug:
    msg: Hi from example role as dependency
```

```bash
vim roles/catalogue/meta/main.yaml
```

```yaml
dependencies:
  - role: example
```

```bash
ansible-playbook -e component=catalogue main.yaml
```

**Expected:** the first task is `TASK [example : this is the dependency role]`, before any catalogue task ✅.

> ⚠️ **Don't use meta for "catalogue needs MongoDB".** The dependency runs on the **same host**, so `- role: mongodb` would install MongoDB **on the catalogue server**. Servers that depend on other servers are handled by the **order** we run the roles (Part 11).

When you've seen it work, empty the dependency so normal runs stay clean:

```yaml
dependencies: []
```

```bash
git add . && git commit -m "v2: catalogue role + defaults/vars/meta demo" && git push
```

---

# Part 7: user, cart, shipping, payment

## Step 23: user and cart roles

Both are Node.js, like catalogue, but with no extra steps.

```bash
mkdir -p roles/user/{tasks,templates} roles/cart/{tasks,templates}
```

```bash
cp ../roboshop-with-ansible-v1/user.service.j2 roles/user/templates/
```

```bash
cp ../roboshop-with-ansible-v1/cart.service.j2 roles/cart/templates/
```

```bash
vim roles/user/tasks/main.yaml
```

```yaml
- name: application setup
  include_role:
    name: common
    tasks_from: app-setup

- name: nodejs setup
  include_role:
    name: common
    tasks_from: nodejs

- name: systemd setup
  include_role:
    name: common
    tasks_from: systemd-setup

- name: restart application
  include_role:
    name: common
    tasks_from: restart
```

```bash
cp roles/user/tasks/main.yaml roles/cart/tasks/main.yaml
```

**Why the same file works for both:** every name inside the common files is `{{ component }}`.

```bash
ansible-playbook -e component=user main.yaml
```

```bash
ansible-playbook -e component=cart main.yaml
```

```bash
curl -s http://user.<your-domain>:8080/health
```

```bash
curl -s http://cart.<your-domain>:8080/health
```

**Expected:** both `"app":"OK"` ✅. In v1, `06-user.yaml` was about 70 lines; the user role is **4 tasks**.

---

## Step 24: shipping role (Java + MySQL data)

```bash
mkdir -p roles/shipping/{tasks,templates}
```

```bash
cp ../roboshop-with-ansible-v1/shipping.service.j2 roles/shipping/templates/
```

```bash
vim roles/shipping/tasks/main.yaml
```

```yaml
- name: application setup
  include_role:
    name: common
    tasks_from: app-setup

- name: java setup
  include_role:
    name: common
    tasks_from: java

- name: systemd setup
  include_role:
    name: common
    tasks_from: systemd-setup

- name: check cities data exist or not
  command: mysql -h "{{ mysql_host }}" -u root -p{{ mysql_root_password }} -N -B -e "SELECT COUNT(*) FROM cities.cities;"
  register: cities_count
  ignore_errors: true
  no_log: true
  tags:
  - mysql

- name: load cities data into mysql
  shell: mysql -h "{{ mysql_host }}" -u root -p{{ mysql_root_password }} < /app/db/master-data.sql
  when: cities_count.rc != 0 or cities_count.stdout | int == 0
  no_log: true
  tags:
  - mysql

- name: create app user in mysql
  shell: mysql -h "{{ mysql_host }}" -u root -p{{ mysql_root_password }} < /app/db/app-user.sql
  no_log: true
  tags:
  - mysql

- name: restart application
  include_role:
    name: common
    tasks_from: restart
```

| Task | Why |
|---|---|
| `java` (common) | Maven + Java 25, builds `/app/shipping.jar` |
| `check cities…` | `rc != 0` = database missing, `count == 0` = empty → load only then |
| `no_log: true` | The command has the MySQL root password; hide it from the output |
| `tags: mysql` | Run only the data tasks with `--tags mysql` |

```bash
ansible-playbook -e component=shipping main.yaml
```

**Expected:** `failed=0`, `load cities data` is **skipping** (loaded in v1) ✅. The Maven build takes a minute or two.

```bash
curl -s http://shipping.<your-domain>:8080/health
```

---

## Step 25: payment role (Python)

```bash
mkdir -p roles/payment/{tasks,templates}
```

```bash
cp ../roboshop-with-ansible-v1/payment.service.j2 roles/payment/templates/
```

```bash
sed 's/tasks_from: nodejs/tasks_from: python/; s/name: nodejs setup/name: python setup/' roles/user/tasks/main.yaml > roles/payment/tasks/main.yaml
```

**What it does:** copies the user role's tasks and swaps `nodejs` for `python`.

```bash
cat roles/payment/tasks/main.yaml
```

**Expected:** app-setup → **python** → systemd-setup → restart ✅.

```bash
ansible-playbook -e component=payment main.yaml
```

```bash
curl -s http://payment.<your-domain>:8080/health
```

**Expected:** `"app":"OK"` and `"rabbitmq":true` ✅. (`rabbitmq:false` once right after a restart? Wait 10 seconds and check again.)

```bash
git add . && git commit -m "v2: user, cart, shipping, payment roles" && git push
```

---

# Part 8: frontend with a handler

## Step 26: Create the frontend role with notify

```bash
mkdir -p roles/frontend/{tasks,templates,handlers}
```

```bash
cp ../roboshop-with-ansible-v1/roboshop.conf.j2 roles/frontend/templates/
```

```bash
sed -n '/tasks:/,$p' ../roboshop-with-ansible-v1/10-frontend.yaml | tail -n +2 | sed 's/^  //' > roles/frontend/tasks/main.yaml
```

```bash
vim roles/frontend/tasks/main.yaml
```

Two changes at the bottom:
1. **Delete** the last task, `restart frontend`.
2. Add `notify: restart nginx` to the `copy roboshop config` task:

```yaml
- name: copy roboshop config
  template:
    src: roboshop.conf.j2
    dest: /etc/nginx/default.d/roboshop.conf
  notify: restart nginx
```

**Why:** v1 restarted nginx on **every** run, even when nothing changed. A restart drops the users connected at that moment. nginx only needs a restart when **its config** changes.

---

## Step 27: Add the handler

```bash
vim roles/frontend/handlers/main.yaml
```

```yaml
- name: restart nginx
  service:
    name: nginx
    state: restarted
```

**Definition:** a **handler** is a task that runs only when another task **notifies** it **and** that task is `changed`. It runs **once**, at the **end** of the play, even if many tasks notify it.

```
copy roboshop config ──changed?──► yes ──► notify "restart nginx" ──► runs at end of play
                          │
                          └────► no (ok) ──► handler doesn't run
```

> The name in `notify:` must match the handler's `name:` exactly.

---

## Step 28: Run frontend twice and watch the handler

```bash
ansible-playbook -e component=frontend main.yaml
```

**Expected:** `copy roboshop config` is **ok** (v1 already wrote the same file) → **no** `RUNNING HANDLER` line ✅.

Now change the config so the task becomes `changed`. Add a comment at the top of `roles/frontend/templates/roboshop.conf.j2`:

```
# managed by ansible - roboshop v2
```

```bash
ansible-playbook -e component=frontend main.yaml
```

**Expected:** `copy roboshop config` is **changed**, and at the end: `RUNNING HANDLER [frontend : restart nginx]` → `changed` ✅.

```bash
ansible-playbook -e component=frontend main.yaml
```

**Expected:** no handler again: the file hasn't changed since the last run ✅.

```bash
git add . && git commit -m "v2: frontend role with restart nginx handler" && git push
```

---

# Part 9: Passwords → Ansible Vault

Right now `group_vars/all/main.yaml` still has the MySQL, shipping and RabbitMQ passwords in **plain text**, like v1. Move them to an encrypted file.

## Step 29: Create the encrypted vault.yaml

```bash
ansible-vault create group_vars/all/vault.yaml
```

**What it does:** opens an editor; when you save, the file is written **encrypted**. `ansible.cfg` already has `vault_password_file`, so it doesn't ask for the vault password.

Type the secrets, each with the `vault_` prefix:

```yaml
vault_ansible_user: ec2-user
vault_ansible_password: <ssh-password>
vault_mysql_root_password: <mysql-root-password>
vault_shipping_db_user: shipping
vault_shipping_db_password: <shipping-db-password>
vault_rabbitmq_user: roboshop
vault_rabbitmq_password: <rabbitmq-password>
```

```bash
cat group_vars/all/vault.yaml
```

**Expected:** `$ANSIBLE_VAULT;1.1;AES256` and lines of numbers, no password ✅.

---

## Step 30: Point main.yaml to the vault_ variables

```bash
vim group_vars/all/main.yaml
```

Replace the password lines (and the inline `!vault |` block) with references:

```yaml
ansible_user: "{{ vault_ansible_user }}"
ansible_password: "{{ vault_ansible_password }}"
mysql_root_password: "{{ vault_mysql_root_password }}"
shipping_db_user: "{{ vault_shipping_db_user }}"
shipping_db_password: "{{ vault_shipping_db_password }}"
rabbitmq_user: "{{ vault_rabbitmq_user }}"
rabbitmq_password: "{{ vault_rabbitmq_password }}"
```

Keep the `*_host` lines as they are.

**Why the `vault_` pattern?** `main.yaml` stays readable: you can `grep` for `mysql_root_password` and see **where** it comes from, without decrypting anything. The roles still use the normal names, so **no role changes**.

> ⚠️ Delete the old password lines; don't comment them out. A commented password is still in Git.

---

## Step 31: Run and check Git has no plain passwords

```bash
ansible-playbook -e component=mysql main.yaml
```

```bash
ansible-playbook -e component=payment main.yaml
```

**Expected:** both `failed=0` ✅. Login, MySQL and RabbitMQ passwords now come from the vault.

```bash
git diff
```

**Expected:** `main.yaml` shows only `{{ vault_… }}` references; `vault.yaml` is new and encrypted ✅.

```bash
git add . && git commit -m "v2: move passwords to ansible vault" && git push
```

> **What goes to GitHub:** the **encrypted** `vault.yaml`. **What never goes:** the vault password file (`~/.vault_pass`). It's outside the repo.

**The problem we hit:** every new or changed password = `ansible-vault edit` → push → everyone pulls, and every new person or server needs the vault password copied to it by hand. That's why we move on to Secrets Manager.

---

# Part 10: Vault → AWS Secrets Manager

## Step 32: Store the passwords in Secrets Manager

**AWS Console → Secrets Manager → Store a new secret**

| Setting | Choose / type |
|---|---|
| **Secret type** | **Other type of secret** |
| **Key/value pairs** | One row per password (table below) |
| **Encryption key** | `aws/secretsmanager` |
| **Secret name** | `roboshop/dev` |
| **Rotation** | Off |

| Key | Value |
|---|---|
| `ansible_user` | `ec2-user` |
| `ansible_password` | `<ssh-password>` |
| `mysql_root_password` | `<mysql-root-password>` |
| `shipping_db_user` | `shipping` |
| `shipping_db_password` | `<shipping-db-password>` |
| `rabbitmq_user` | `roboshop` |
| `rabbitmq_password` | `<rabbitmq-password>` |

**Expected:** `roboshop/dev` shows in the secrets list ✅.

---

## Step 33: Allow the Ansible server's role to read the secret

On the Ansible server, find the role name:

```bash
aws sts get-caller-identity --query Arn --output text
```

**Expected:** `arn:aws:sts::<account-id>:assumed-role/<role-name>/i-…`. The part after `assumed-role/` is the role name.

On the **laptop**:

```bash
aws iam put-role-policy --role-name <role-name> --policy-name roboshop-read-secrets --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"secretsmanager:GetSecretValue","Resource":"arn:aws:secretsmanager:us-east-1:<account-id>:secret:roboshop/*"}]}'
```

**What it does:** lets the role **read only** secrets whose names start with `roboshop/`. The `*` is needed because AWS adds 6 random characters to the secret's ARN (`roboshop/dev-AbC123`).

---

## Step 34: Read the secret from the Ansible server

```bash
aws secretsmanager get-secret-value --secret-id roboshop/dev --query SecretString --output text --region us-east-1
```

**Expected:** one line of JSON with all 7 keys ✅. `AccessDeniedException` → Step 33 has the wrong role name.

```bash
ansible localhost -m debug -a "msg={{ (lookup('amazon.aws.secretsmanager_secret', 'roboshop/dev', region='us-east-1') | from_json).keys() | list }}"
```

**What it does:** the same read, through Ansible's lookup plugin. Prints only the **key names**.
**Expected:** a list of the 7 key names ✅.

---

## Step 35: Point group_vars to Secrets Manager

```bash
vim group_vars/all/main.yaml
```

Replace the `vault_` lines with:

```yaml
roboshop_secrets: "{{ lookup('amazon.aws.secretsmanager_secret', 'roboshop/dev', region='us-east-1') | from_json }}"
ansible_user: "{{ roboshop_secrets.ansible_user }}"
ansible_password: "{{ roboshop_secrets.ansible_password }}"
mysql_root_password: "{{ roboshop_secrets.mysql_root_password }}"
shipping_db_user: "{{ roboshop_secrets.shipping_db_user }}"
shipping_db_password: "{{ roboshop_secrets.shipping_db_password }}"
rabbitmq_user: "{{ roboshop_secrets.rabbitmq_user }}"
rabbitmq_password: "{{ roboshop_secrets.rabbitmq_password }}"
```

| Part | Meaning |
|---|---|
| `lookup('amazon.aws.secretsmanager_secret', 'roboshop/dev')` | Runs on the Ansible server, reads the secret with the server's IAM role |
| `\| from_json` | The secret comes back as a JSON **string**; this turns it into a dictionary |
| `roboshop_secrets.mysql_root_password` | Pick one key |

Again the variable names don't change, so **no role changes**.

```bash
ansible-playbook -e component=mysql main.yaml
```

**Expected:** `failed=0` ✅. The SSH login and the MySQL password both came from AWS.

---

## Step 36: Remove the vault

```bash
git rm group_vars/all/vault.yaml
```

```bash
vim ansible.cfg
```

Delete the `vault_password_file=…` line. Nothing is encrypted with Vault any more.

```bash
git diff --cached
```

**Expected:** only lookups and `roboshop_secrets.<key>` lines; **no password** anywhere in the repo ✅.

```bash
git add . && git commit -m "v2: passwords from AWS Secrets Manager, vault removed" && git push
```

| | Ansible Vault | AWS Secrets Manager |
|---|---|---|
| Where the secret is | Encrypted file in Git | AWS, outside Git |
| Who can read it | Anyone with the vault password | Only IAM roles allowed to |
| Change a password | Decrypt, edit, re-encrypt, push, pull | Edit it in AWS; next run picks it up |
| Audit | None | CloudTrail logs every read |

---

# Part 11: Open the shop

## Step 37: Run every role in order

The order matters: databases first, then the apps that use them, frontend last.

```bash
for c in mongodb redis mysql rabbitmq catalogue user cart shipping payment frontend; do ansible-playbook -e component=$c main.yaml || break; done
```

**What it does:** runs `main.yaml` once per component, and stops at the first failure.
**Expected:** 10 recaps, all `failed=0` ✅.

---

## Step 38: Open the website and check the APIs

Browser: `http://<your-domain>`

**Expected:** the RoboShop home page with products ✅. Register, log in, add to cart, check out.

```bash
for s in catalogue user cart shipping payment; do echo -n "$s: "; curl -s http://<your-domain>/api/$s/health; echo; done
```

**Expected:** 5 lines with `"app":"OK"` ✅.

---

# Part 12: Clean up

## Step 39: Stop or delete the servers

**Coming back to this soon?** Stop (keeps disks and setup, costs only storage):

```bash
aws ec2 stop-instances --instance-ids $(aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" --query "Reservations[].Instances[].InstanceId" --output text)
```

> After the next start, repeat **Step 3** (frontend public IP changes).

**Done with RoboShop?** Terminate the servers and records as in [v1 Step 28 and 29](https://github.com/Abhignadumpala/roboshop-with-ansible-v1/blob/main/LAB.md#step-28-delete-the-roboshop-servers-and-records-ansible-server), then delete the secret:

```bash
aws secretsmanager delete-secret --secret-id roboshop/dev --recovery-window-in-days 7 --region us-east-1
```

> Secrets Manager costs about $0.40 per secret per month. The 7-day window lets you undo the delete.

---

# Wrap-up

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `the role 'catalogue' was not found` | Folder name or path wrong, or running from another folder | Run from the repo root; folder must be `roles/catalogue/` |
| `'component' is undefined` | Forgot `-e component=…` | Add `-e component=<name>` |
| `Could not find or access 'mongo.repo'` | File not in the role's `files/` | Copy it into `roles/<role>/files/` |
| `Could not find or access 'cart.service.j2'` | Template not in the role's `templates/` | Copy it into `roles/cart/templates/` |
| YAML error: `mapping values are not allowed` | Tasks still indented 2 spaces after removing `tasks:` | Each `- name:` must start at column 0 |
| mysql: `Access denied` on the second run | `mysql_secure_installation` runs again on a server that has a password | Step 13 check (`when: root_check.rc != 0`) |
| mysql check always `FAILED ...ignoring`, password task always `changed` | Check logs in as `root@localhost` (no password); the password is on `root@%` | Add `-h {{ mysql_host }}` to the check (Step 13) |
| Handler never runs | Task was `ok`, or the `notify:` name ≠ handler `name:` | Change the file to test; copy the name exactly |
| `Attempting to decrypt but no vault secrets found` | `vault_password_file` removed while `vault.yaml` still exists | Remove both together (Step 36) |
| `AccessDeniedException` from the lookup | IAM role can't read the secret | Step 33: right role name, resource `secret:roboshop/*` |
| `KeyError` / `has no attribute 'shipping_db_password'` | Key missing or misspelt in the secret | Check key names with Step 34 |
| Website down after a restart | Apex record has the old public IP | Step 3 |

---

## What I learned

| Topic | One line |
|---|---|
| Roles | One folder per server with a fixed structure; Ansible finds tasks, files, templates by folder name |
| `-e component=` | One entry playbook runs any role; the variable picks hosts, role and names |
| Common role | Shared steps written once, called with `include_role` + `tasks_from` |
| Template search | With `include_role`, the calling role's `templates/` is searched too |
| defaults vs vars | `defaults/` = lowest priority, `vars/` = high; `-e` beats both |
| meta | Dependencies run first on the **same host**, not for server-to-server order |
| Handlers | Run once at the end of the play, only when a notifying task is `changed` |
| Idempotency | Check first (`register` + `when`), so re-runs don't break (mysql root password) |
| Vault → Secrets Manager | Encrypted file in Git → AWS-managed secret read with IAM; no secrets in the repo |

---

## Interview questions

### ⭐ Very Important

1. **What is an Ansible role and why use it over a playbook?**
   A fixed folder structure (tasks, files, templates, vars, defaults, handlers, meta) for one job. It's reusable, easy to share, and keeps one big playbook from growing into hundreds of lines.

2. **How did you run 10 servers with one playbook?**
   `main.yaml` uses `hosts: "{{ component }}"` and `roles: - "{{ component }}"`; I pass `-e component=cart` to pick the group and the role.

3. **What's the difference between `defaults` and `vars` in a role?**
   Both hold role variables. `defaults/` has the lowest priority (meant to be overridden); `vars/` has high priority (only `set_fact` and `-e` beat it).

4. **What are handlers?**
   Tasks that run only when notified by a task that changed something, once, at the end of the play. I use one to restart nginx only when its config changes.

5. **How do you keep secrets out of Git in Ansible?**
   Ansible Vault (encrypted file in Git) or, better on AWS, Secrets Manager read with the `amazon.aws.secretsmanager_secret` lookup and an IAM role. I moved from hardcoded → Vault → Secrets Manager.

6. **include_role vs import_role?**
   `import_role` is static (read at parse time, `when`/tags copied to every task, no loops). `include_role` is dynamic (runs when reached, `when` checked once, works with loops and variable names). I used `include_role` with `tasks_from` for the common role.

### Other Questions

7. **What does `meta/main.yaml` do, and when should you not use it?**
   Lists roles that run first on the same host. Not for "catalogue needs the MongoDB server": that's a different host, handled by run order.

8. **Where does `copy: src: mongo.repo` look inside a role?**
   The role's `files/` folder. `template:` looks in `templates/`.

9. **How did you make the mysql role idempotent?**
   A `register`ed login check with `ignore_errors`, and `when: root_check.rc != 0` on the set-password task.

10. **Why `secret:roboshop/*` in the IAM policy and not the exact name?**
    AWS appends 6 random characters to the secret's ARN, so an exact ARN without the suffix doesn't match.
