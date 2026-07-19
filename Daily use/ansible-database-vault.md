# Ansible Database Role with Ansible Vault

## Overview

This document explains how to:

- Create an Ansible database role
- Install database packages
- Configure database services
- Create databases and users
- Store database passwords securely using Ansible Vault
- Execute playbooks using encrypted secrets

Ansible Vault is the standard Ansible method for protecting sensitive information such as:

- Database passwords
- API keys
- Certificates
- Tokens

Ansible Vault encrypts files at rest and decrypts them temporarily in memory during playbook execution.

---

# 1. Ansible Role Directory Structure

Example:

```
ansible-project/

├── inventory
├── site.yml
│
└── roles/
    │
    └── database/
        │
        ├── tasks/
        │   └── main.yml
        │
        ├── vars/
        │   ├── main.yml
        │   └── secrets.yml
        │
        ├── handlers/
        │   └── main.yml
        │
        └── templates/
```

---

# 2. Create Encrypted Secrets File

Create the encrypted secrets file:

```bash
ansible-vault create roles/database/vars/secrets.yml
```

The editor opens.

Add database credentials:

```yaml
---

db_root_password: "S3cur3P@ssw0rd!"

db_app_password: "AnotherSecret123"
```

Save and exit.

The file is encrypted automatically.

Check:

```bash
cat roles/database/vars/secrets.yml
```

Output:

```
$ANSIBLE_VAULT;1.1;AES256
66386439653...
```

The password is no longer visible.

---

# 3. Encrypt Existing File

If the file already exists in plain text:

```bash
ansible-vault encrypt roles/database/vars/secrets.yml
```

---

# 4. Edit Encrypted File

To modify the vault file:

```bash
ansible-vault edit roles/database/vars/secrets.yml
```

Workflow:

```
Decrypt temporarily
        |
Open editor
        |
Save changes
        |
Encrypt again
```

---

# 5. Database Role Variables

File:

```
roles/database/vars/main.yml
```

Example:

```yaml
---

db_name: myapp_db

db_user: app_user

database_package: mysql-server

database_service: mysqld
```

---

# 6. Database Secrets

File:

```
roles/database/vars/secrets.yml
```

Encrypted:

```yaml
---

db_root_password: "S3cur3P@ssw0rd!"

db_app_password: "AnotherSecret123"
```

---

# 7. Database Role Tasks

File:

```
roles/database/tasks/main.yml
```

Example:

```yaml
---

- name: Load database secrets
  ansible.builtin.include_vars:
    file: secrets.yml


- name: Install database package
  ansible.builtin.package:
    name: "{{ database_package }}"
    state: present


- name: Enable and start database service
  ansible.builtin.systemd:
    name: "{{ database_service }}"
    state: started
    enabled: yes


- name: Create database
  community.mysql.mysql_db:
    name: "{{ db_name }}"
    state: present
    login_password: "{{ db_root_password }}"


- name: Create database user
  community.mysql.mysql_user:
    name: "{{ db_user }}"
    password: "{{ db_app_password }}"
    priv: "{{ db_name }}.*:ALL"
    state: present
    login_password: "{{ db_root_password }}"
  no_log: true
```

---

# 8. Why Use no_log: true

Database passwords are sensitive.

Without:

```yaml
no_log: true
```

Secrets can appear in:

- Terminal output
- CI/CD logs
- Ansible Tower/AWX logs

Example:

```
changed: password=S3cur3P@ssw0rd!
```

With:

```yaml
no_log: true
```

Output:

```
changed: [database-server]
```

The password remains hidden.

---

# 9. Calling the Role from Playbook

File:

```
site.yml
```

Example:

```yaml
---

- name: Configure Database Server
  hosts: database
  become: yes

  roles:
    - database
```

---

# 10. Inventory File

File:

```
inventory
```

Example:

```ini
[database]

db01 ansible_host=192.168.1.20
```

---

# 11. Run Playbook with Vault Password

## Option 1: Interactive Password

```bash
ansible-playbook -i inventory site.yml --ask-vault-pass
```

Example:

```
Vault password:
```

---

## Option 2: Vault Password File

Create password file:

```bash
echo "MyVaultPassword" > ~/.vault_pass.txt
```

Secure it:

```bash
chmod 600 ~/.vault_pass.txt
```

Run:

```bash
ansible-playbook \
-i inventory \
site.yml \
--vault-password-file ~/.vault_pass.txt
```

Best practice:

- Do not commit vault password files to Git
- Restrict file permissions
- Use enterprise secrets managers

Examples:

- Hashicorp Vault
- AWS Secrets Manager
- Azure Key Vault

---

# 12. Configure Vault Password in ansible.cfg

File:

```
ansible.cfg
```

Example:

```ini
[defaults]

vault_password_file = ~/.vault_pass.txt
```

Now:

```bash
ansible-playbook -i inventory site.yml
```

No need to specify vault password every time.

---

# 13. Encrypt Only Individual Secret Values

Instead of encrypting the whole file:

```bash
ansible-vault encrypt_string \
'S3cur3P@ssw0rd!' \
--name db_root_password
```

Output:

```yaml
db_root_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66386439653...
```

Add into:

```
roles/database/vars/main.yml
```

Example:

```yaml
---

db_root_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66386439653...
```

Advantages:

- Configuration remains readable
- Only secrets are encrypted
- Easier Git reviews

---

# 14. Complete Execution Flow

```
Developer
    |
    |
Git Repository
    |
    |
Ansible Playbook
    |
    |
Database Role
    |
    |
Load Vault Secrets
    |
    |
Install Database Package
    |
    |
Enable Service
    |
    |
Create Database
    |
    |
Create User
    |
    |
Apply Permissions
```

---

# 15. Best Practices

## Security

- Never store passwords in plain text
- Use Ansible Vault for secrets
- Protect vault password files
- Use no_log for sensitive tasks
- Use separate variables for environments


## Automation

- Use roles instead of large playbooks
- Keep variables separate
- Make tasks idempotent
- Store automation code in Git


## Enterprise Architecture

```
Git
 |
CI/CD Pipeline
 |
Ansible Tower / AWX
 |
Ansible Vault
 |
Database Servers
```

---

# Interview Explanation

> I separate configuration variables from sensitive information. Normal database parameters are stored in vars files, while credentials are protected using Ansible Vault. During execution, the role loads encrypted secrets, installs and configures the database, creates users and permissions, and uses no_log to prevent password leakage. This provides secure, repeatable, and auditable infrastructure automation.
