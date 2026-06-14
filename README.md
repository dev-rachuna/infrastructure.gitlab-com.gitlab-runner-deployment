# <img src=".gitlab/ansible.png" alt="ansible" height="30"/> Ansible – GitLab Runner Infrastructure

::include{file=.gitlab/badges.md}

Repozytorium automatyzacji i konfiguracji infrastruktury GitLab Runner przy użyciu **Ansible**. 

Projekt automatyzuje instalację, konfigurację i hardening trzech maszyn GitLab Runner (`gitlab-runner-1002`, `gitlab-runner-1003`, `gitlab-runner-1004`) dla domeny **dev.rachuna**.

---

## 🎯 Cel Projektu

Standaryzacja i automatyzacja infrastruktury CI/CD runner machines poprzez Infrastructure-as-Code (IaC) podejście:

- ✅ **Hardening bezpieczeństwa** – SSH, sudo, users, firewall rules
- ✅ **Zarządzanie konfiguracją** – timezone, hostname, locale, packages
- ✅ **Automatyzacja GitLab Runner** – instalacja, rejestracja, cache NFS
- ✅ **Zarządzanie certyfikatami TLS** – integracja z HashiCorp Vault
- ✅ **Wersjonowanie infrastruktury** – wszystko w Git (Conventional Commits)

---

## 📋 Wymagania

### Minimum
- **Ansible** 2.9+ (rekomendacja: 2.11+)
- **Python 3.6+** na maszynie kontrolnej
- **SSH dostęp** do target hostów

### Opcjonalnie
- **Vault** (HashiCorp) – dla zarządzania certyfikatami/sekretami
- **Molecule** – dla testowania roles

### Instalacja Ansible

```bash
# macOS (Homebrew)
brew install ansible

# Ubuntu/Debian
sudo apt-get install ansible

# pip (zalecane)
pip install ansible>=2.11
```

---

## 🚀 Quick Start

### 1. Klonuj repozytorium

```bash
git clone https://gitlab.com/dev.rachuna/infrastructure/gitlab-com/gitlab-runner-deployment.git
cd gitlab-runner-deployment
```

### 2. Instalacja external roles

```bash
ansible-galaxy role install -f -r requirements.yml -p playbooks/roles
```

### 3. Test połączenia z hostami

```bash
ansible-playbook playbooks/test_connection.yml -i inventory/hosts.yml
```

### 4. Dry-run (test bez zmian)

```bash
ansible-playbook playbooks/install.yml -i inventory/hosts.yml --check
```

### 5. Pełna instalacja

```bash
ansible-playbook playbooks/install.yml -i inventory/hosts.yml
```

---

## 📁 Struktura Projektu

```bash
.
├── playbooks/
│   ├── install.yml                     # ⭐ Główny playbook
│   ├── test_connection.yml             # Test połączenia
│   └── roles/
│       ├── gitlab-runner/              # Local role: GitLab Runner setup
│       ├── certificates/               # Local role: Certificate handling
│       └── [external roles gitignored]
├── inventory/
│   ├── hosts.yml                       # Definicja hostów
│   ├── group_vars/
│   │   └── all/                        # Global variables
│   │       ├── main.yml
│   │       ├── certificates.yml
│   │       ├── users_accont.yml
│   │       ├── technical_account.yml
│   │       ├── groups.yml
│   │       ├── locale.yml
│   │       └── sshd_config.yml
│   └── host_vars/
│       ├── gitlab-runner-1002/
│       ├── gitlab-runner-1003/
│       └── gitlab-runner-1004/
├── requirements.yml                    # External roles (versioned)
├── .gitignore                          # External roles excluded
├── README.md                           # Ten plik
├── CONTRIBUTING.md                     # Contribution guidelines
└── LICENSE                             # MIT License
```

---

## 🏗️ Architektura

### Role Stack (w kolejności wykonania)

| # | Role | Typ | Opis |
|---|------|-----|------|
| 1 | `set-timezone` | External | Ustawia timezone, locale, language |
| 2 | `users-management` | External | Tworzy użytkowników, grupy, SSH keys |
| 3 | `sudo` | External | Konfiguruje sudo policies |
| 4 | `set-hostname` | External | Ustawia FQDN hostname |
| 5 | `ssh-hardening` | External | Hardening konfiguracji SSH (sshd_config) |
| 6 | `install-packages` | External | Instaluje pakiety (APT/DNF/APK) |
| 7 | `certificates` | External | Zarządza certyfikatami TLS + Vault |
| 8 | `gitlab-runner` | Local | Instaluje i rejestruje GitLab Runner |

### Zmienne (Hierarchia Priorytetów)

```bash
Role defaults
    ↓ overridden by
Role vars/
    ↓ overridden by
group_vars/all/
    ↓ overridden by
group_vars/gitlab_runners/
    ↓ overridden by
host_vars/gitlab-runner-*/
    ↓ overridden by
Playbook vars:
```

---

## ⚙️ Konfiguracja

### Global Variables (`inventory/group_vars/all/main.yml`)

```yaml
inv_all_root_root_domain: rachuna.dev
inv_all_timezone: "Europe/Warsaw"
inv_all_locale: "en_US.UTF-8"
inv_all_language: "en_US"
inv_all_lc_all: "en_US.UTF-8"
```

### Runner-Specific Config (`inventory/group_vars/gitlab_runners/configuration.yml`)

```yaml
inv_gitlab_runner_cache_nfs:
  path: /mnt/cache
  server: nfs.rachuna.dev

inv_host_gitlab_runner_configuration:
  concurrent: 4
  check_interval: 0
  # ... więcej opcji
```

### Per-Host Override (`inventory/host_vars/gitlab-runner-1002/main.yml`)

```yaml
fqdn: gitlab-runner-1002.rachuna.dev
```

---

## 📝 Użycie

### Uruchomienie pełnego playbooku

```bash
ansible-playbook playbooks/install.yml -i inventory/hosts.yml
```

### Uruchomienie tylko jednego role'a

```bash
ansible-playbook playbooks/install.yml -i inventory/hosts.yml -t gitlab-runner
```

### Dry-run z verbose output

```bash
ansible-playbook playbooks/install.yml -i inventory/hosts.yml --check -vvv
```

### Syntax check

```bash
ansible-playbook playbooks/install.yml --syntax-check
```

### Listy tasków bez wykonania

```bash
ansible-playbook playbooks/install.yml --list-tasks
```

### Custom user/password

```bash
ansible-playbook playbooks/install.yml -i inventory/hosts.yml -u ubuntu -k -K
```

---

## 🔧 Dodawanie Nowego Runner'a

1. **Dodaj do inventory:**

   ```yaml
   # inventory/hosts.yml
   gitlab_runners:
     children:
       nodes:
         hosts:
           gitlab-runner-1028:  # nowy host
   ```

2. **Utwórz host_vars:**

   ```bash
   mkdir -p inventory/host_vars/gitlab-runner-1028
   ```

3. **Konfiguruj hostname:**

   ```yaml
   # inventory/host_vars/gitlab-runner-1028/main.yml
   fqdn: gitlab-runner-1028.rachuna.dev
   ```

4. **Konfiguruj runner:**

   ```yaml
   # inventory/host_vars/gitlab-runner-1028/gitlab-runner.yml
   inv_host_gitlab_runner_configuration:
     concurrent: 4
     # ... reszta konfiguracji
   ```

5. **Uruchom playbook:**

   ```bash
   ansible-playbook playbooks/install.yml -i inventory/hosts.yml
   ```

---

## 🧪 Testowanie

### Test połączenia

```bash
ansible-playbook playbooks/test_connection.yml -i inventory/hosts.yml
```

### Inventory validation

```bash
ansible-inventory -i inventory/hosts.yml --list
ansible-inventory -i inventory/hosts.yml --host gitlab-runner-1002
```

### Check mode (dry-run)

```bash
ansible-playbook playbooks/install.yml -i inventory/hosts.yml --check
```

### Verbose output

```bash
# -v = minimal verbose
# -vv = more verbose
# -vvv = debug level
ansible-playbook playbooks/install.yml -i inventory/hosts.yml -vvv
```

---

## 🔐 Bezpieczeństwo

### Vault Integration

- Certyfikaty mogą być przechowywane w **HashiCorp Vault**
- Ustaw `inv_certificates_vault_addr` w inventory
- Runner musi mieć dostęp do Vault (via IAM lub token)

### SSH Hardening

- Role `ssh-hardening` modyfikuje `/etc/ssh/sshd_config`
- Wyłącza root login, słabe algorytmy, itp.
- **Ostrzeżenie**: Testuj w safe environment przed produkcją

### Secrets Management

- **Nigdy** nie commituj credentials, tokens, czy secrets do repozytorium
- Używaj `ansible-vault` dla sensitive data
- Przechowuj tokeny w environment variables lub Vault

---

## 📦 External Roles (z requirements.yml)

Wszystkie external roles są instalowane z pliku `requirements.yml` z GitLab repo:

```bash
https://gitlab.com/dev.rachuna/artifacts/ansible-roles/
```

**Gitignore**: External roles NIE są commitowane do tego repo (zobacz `.gitignore`)

---

::include{file=.gitlab/footer.md}
