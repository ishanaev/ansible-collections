# Proxmox Management Collection

Коллекция Ansible для управления Proxmox VE, включающая плейбуки для создания и управления виртуальными машинами и LXC контейнерами.

## Требования

- Ansible 2.9 или выше
- Доступ к Proxmox VE через SSH
- Права администратора на Proxmox VE
- Доступные шаблоны LXC в хранилище Proxmox

## Установка

```bash
ansible-galaxy collection install git+https://github.com/your-username/proxmox-management.git
```

## Плейбуки

### proxmox_vm.yml
Создание и управление виртуальными машинами KVM.

### proxmox_lxc.yml
Создание и управление LXC контейнерами.

### proxmox_lxc_users.yml
Управление пользователями в LXC контейнерах.

## Использование

```bash
# Создание виртуальных машин
ansible-playbook -i inventory.yml collection/ansible_collections/proxmox/management/playbooks/proxmox_vm.yml

# Создание LXC контейнеров
ansible-playbook -i inventory.yml collection/ansible_collections/proxmox/management/playbooks/proxmox_lxc.yml

# Управление пользователями в LXC
ansible-playbook -i inventory.yml collection/ansible_collections/proxmox/management/playbooks/proxmox_lxc_users.yml
```

## Документация

Подробная документация по каждому плейбуку доступна в директории `docs/`. 