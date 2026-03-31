# Proxmox VE Setup — Ansible Playbook

Установка и настройка Proxmox VE на dedicated server.

## Роли

| Роль | Тег | Что делает |
|------|-----|------------|
| `pve-install` | `install` | Hostname, PVE-репозиторий, установка proxmox-ve, reboot, удаление stock kernel, os-prober, LVM thin pool |
| `network` | `network` | Интерфейсы (/32 + pointopoint), vmbr0 (routed), vmbr1 (NAT), ip_forward, SSH hardening, nftables, fail2ban |
| `pve-config` | `pve-config` | Отключение enterprise-repo, создание GUI-пользователей (список), отключение root в GUI |
| `cloud-template` | `cloud-template` | Cloud-init шаблон Ubuntu 24.04 (VM ID 9000), vendor-data (qemu-guest-agent) |
| `monitoring` | `monitoring` | msmtp (Gmail), cron-алерты (RAM/Disk) |
| `vm-create` | — | Создание VM для клиента: авто-выбор IP, cloud-init, SSH-ключ, Telegram-уведомление |


## Предустановки

На сервере должен быть установлен Debian 13.
Должен быть ssh доступ на сервер по ключу.

**Пример разметки дисков:**

| Раздел | RAID | Размер | Назначение |
|--------|------|--------|------------|
| /boot | md0 | 1 GB | Загрузчик |
| swap | md1 | 4 GB | Swap |
| / | md2 | 30 GB | ОС |
| LVM (vg0) | md3 | All | Thin pool для VM дисков |

## Перед запуском

### 1. Создай vault и host_vars

```bash
cp secrets/vault.example.yml secrets/vault.yml
cp host_vars/your-server.example.yml host_vars/<your-hostname>.yml
```

Заполни `secrets/vault.yml` (см. примеры в `secrets/vault.example.yml`):

- Пользователи PVE GUI (имя, пароль, роль)
- Email и SMTP для алертов
- Telegram bot token и chat_id
- Публичная подсеть и IP сервера

Заполни `host_vars/<your-hostname>.yml` (см. примеры в `host_vars/your-server.example.yml`):

- `ansible_host` — IP-адрес сервера
- `ansible_user` — пользователь SSH
- `ansible_ssh_private_key_file` — путь к SSH-ключу
- `pve_domain` — домен для PVE

> **Gmail App Password**: https://myaccount.google.com/apppasswords
> **Telegram Bot**: создать через `@BotFather`, chat_id получить через `getUpdates`

### 2. Зашифруй

```bash
echo 'my-vault-password' > .vault_password_file
ansible-vault encrypt secrets/vault.yml
ansible-vault encrypt host_vars/<your-hostname>.yml
```

### 3. Обнови inventory.yml

Замени имя хоста на своё (должно совпадать с именем файла в `host_vars/`):

```yaml
proxmox:
  hosts:
    <your-hostname>:
```

## Запуск

```bash
# Полная установка (чистый Debian → готовый PVE)
ansible-playbook pve-setup.yml

# Пропустить установку PVE (уже установлен)
ansible-playbook pve-setup.yml --skip-tags install

# Только одна роль
ansible-playbook pve-setup.yml --tags network
```

## Сеть

| Bridge | Тип | Подсеть | Назначение |
|--------|-----|---------|------------|
| `vmbr0` | Routed | `<PUBLIC_SUBNET>/29` | Публичные VM (6 IP) |
| `vmbr1` | NAT | `10.10.10.0/24` | Приватные VM (выход через masquerade) |

## После запуска

- GUI: `https://<IP>:8006` — вход как `<user>@pam`
- SSH: `ssh root@<IP>` (только по ключу)
- root в GUI **отключён** (пользователи могут менять пароль через GUI)

### Создание VM

```bash
# Публичная VM (IP выбирается автоматически)
ansible-playbook vm-create.yml -e "vm_id=101 vm_name=client-001 vm_ssh_key='ssh-ed25519 AAAA...'"

# Приватная VM (NAT)
ansible-playbook vm-create.yml -e "vm_id=201 vm_name=client-002 vm_network=private vm_ssh_key='ssh-ed25519 AAAA...'"

# С кастомными ресурсами (дефолты: 1 CPU, 1024MB RAM, +10GB disk)
ansible-playbook vm-create.yml -e "vm_id=101 vm_name=client-001 vm_cores=2 vm_memory=4096 vm_disk=30 vm_ssh_key='ssh-ed25519 AAAA...'"
```

После создания — Telegram-уведомление с IP и количеством свободных адресов.

### Если нужно вернуть root в GUI

```bash
pveum user modify root@pam --enable 1
```
