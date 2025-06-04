# Документация по переменным

## Основные переменные аутентификации

Эти переменные используются во всех плейбуках и ролях для подключения к Proxmox:

```yaml
proxmox_api_host: "{{ ansible_host }}"  # IP-адрес или hostname Proxmox сервера
proxmox_api_user: "root@pam"           # Пользователь Proxmox
proxmox_api_password: "your_password"   # Пароль (рекомендуется использовать vault)
proxmox_node: "pve"                    # Имя ноды Proxmox
```

## Переменные для LXC контейнеров

### Основные параметры контейнера
```yaml
containers:
  - name: "container1"                  # Имя контейнера
    vmid: 200                          # ID контейнера
    cores: 2                           # Количество ядер CPU
    memory: 1024                       # RAM в МБ
    swap: 512                          # Swap в МБ
    ostype: "ubuntu"                   # Тип ОС
    ostemplate: "local:vztmpl/..."     # Путь к шаблону
    rootfs: "local-lvm:8"              # Корневая ФС
    onboot: 1                          # Запуск при старте
    unprivileged: 0                    # Режим непривилегированного контейнера
    features:                          # Дополнительные возможности
      nesting: 1
      keyctl: 1
    start: 1                           # Автозапуск после создания
```

### Сетевые настройки
```yaml
    net:
      net0: "name=eth0,bridge=vmbr0,ip=dhcp"  # Настройки сети
```

### Пользователи контейнера
```yaml
    users:
      - username: "admin"              # Имя пользователя
        password: "secure_password"    # Пароль (рекомендуется vault)
        groups: "sudo,adm"            # Группы пользователя
        shell: "/bin/bash"            # Оболочка
        state: "present"              # Состояние (present/absent)
```

## Взаимодействие переменных между ролями

### Роль proxmox_auth
- Использует основные переменные аутентификации
- Проверяет доступность Proxmox
- Регистрирует результат в `proxmox_status`

### Роль proxmox_network
- Принимает переменные:
  ```yaml
  vmid: "{{ item.item.vmid }}"        # ID контейнера/VM
  container_type: "lxc"               # Тип контейнера
  net_config: "{{ item.item.net }}"   # Конфигурация сети
  ```
- Использует основные переменные аутентификации
- Настраивает сеть и DNS

### Роль proxmox_users
- Принимает переменные:
  ```yaml
  vmid: "{{ item.item.vmid }}"        # ID контейнера
  users: "{{ item.item.users }}"      # Список пользователей
  ```
- Использует основные переменные аутентификации
- Управляет пользователями и их правами

## Приоритеты переменных

1. Переменные в плейбуке (высший приоритет)
2. Переменные в роли (defaults/main.yml)
3. Переменные в группе хостов
4. Переменные в inventory

## Рекомендации по использованию

1. **Безопасность**:
   - Используйте Ansible Vault для хранения паролей
   - Не храните чувствительные данные в plain text

2. **Организация**:
   - Группируйте связанные переменные
   - Используйте осмысленные имена
   - Документируйте нестандартные значения

3. **Переиспользование**:
   - Выносите общие переменные в defaults
   - Используйте шаблоны для повторяющихся значений
   - Применяйте условные конструкции

## Пример использования vault

```bash
# Создание зашифрованной строки
ansible-vault encrypt_string 'your_password' --name 'proxmox_api_password'

# Результат в плейбуке
proxmox_api_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          6638613965386538326430626638646638646638646638646638646638646638
```

## Обработка ошибок

- Проверяйте наличие обязательных переменных
- Используйте значения по умолчанию где возможно
- Добавляйте валидацию входных данных
- Логируйте важные изменения 

## Примеры условных конструкций

### Проверка состояния контейнера
```yaml
- name: Проверка состояния контейнера
  community.general.proxmox_lxc:
    vmid: "{{ vmid }}"
    state: present
  register: container_status

- name: Действия только для запущенных контейнеров
  debug:
    msg: "Контейнер {{ vmid }} запущен"
  when: container_status.status == 'running'
```

### Условное создание пользователей
```yaml
- name: Создание пользователей с разными правами
  community.general.proxmox_lxc:
    vmid: "{{ vmid }}"
    username: "{{ item.username }}"
    groups: "{{ item.groups }}"
  loop: "{{ users }}"
  when: 
    - item.state == 'present'
    - item.groups is defined
```

### Проверка доступности ресурсов
```yaml
- name: Проверка доступного места
  community.general.proxmox_lxc:
    vmid: "{{ vmid }}"
    rootfs: "{{ rootfs }}"
  when: 
    - rootfs is defined
    - rootfs | regex_search('^local-lvm:') is not none
```

### Условная настройка сети
```yaml
- name: Настройка сети с проверкой типа
  community.general.proxmox_lxc:
    vmid: "{{ vmid }}"
    net: "{{ net_config }}"
  when: 
    - net_config is defined
    - container_type in ['lxc', 'kvm']
```

## Сценарии использования переменных

### 1. Создание контейнеров с разными ОС
```yaml
containers:
  - name: "ubuntu-container"
    vmid: 200
    ostype: "ubuntu"
    ostemplate: "local:vztmpl/ubuntu-20.04-standard_20.04-1_amd64.tar.gz"
    users:
      - username: "ubuntu_admin"
        groups: "sudo"
  - name: "debian-container"
    vmid: 201
    ostype: "debian"
    ostemplate: "local:vztmpl/debian-11-standard_11.3-1_amd64.tar.gz"
    users:
      - username: "debian_admin"
        groups: "sudo"
```

### 2. Настройка сети с разными конфигурациями
```yaml
containers:
  - name: "dhcp-container"
    vmid: 200
    net:
      net0: "name=eth0,bridge=vmbr0,ip=dhcp"
  - name: "static-ip-container"
    vmid: 201
    net:
      net0: "name=eth0,bridge=vmbr0,ip=192.168.1.100/24,gw=192.168.1.1"
```

### 3. Управление пользователями с разными правами
```yaml
containers:
  - name: "app-container"
    vmid: 200
    users:
      - username: "app_user"
        groups: "www-data"
        state: present
      - username: "admin"
        groups: "sudo,adm"
        state: present
      - username: "old_user"
        state: absent
```

### 4. Настройка контейнеров с разными ресурсами
```yaml
containers:
  - name: "high-resource"
    vmid: 200
    cores: 4
    memory: 4096
    swap: 2048
    rootfs: "local-lvm:32"
  - name: "low-resource"
    vmid: 201
    cores: 1
    memory: 512
    swap: 256
    rootfs: "local-lvm:8"
```

### 5. Использование шаблонов для повторяющихся значений
```yaml
defaults:
  container_defaults:
    onboot: 1
    unprivileged: 0
    features:
      nesting: 1
      keyctl: 1

containers:
  - name: "container1"
    vmid: 200
    "{{ container_defaults | combine }}"
  - name: "container2"
    vmid: 201
    "{{ container_defaults | combine }}"
```

### 6. Условная настройка безопасности
```yaml
containers:
  - name: "secure-container"
    vmid: 200
    unprivileged: 1
    features:
      nesting: 0
      keyctl: 0
    users:
      - username: "secure_user"
        groups: "sudo"
        ssh_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
```

### 7. Динамическое управление ресурсами
```yaml
containers:
  - name: "dynamic-container"
    vmid: 200
    cores: "{{ ansible_processor_count }}"
    memory: "{{ ansible_memtotal_mb | int * 0.5 }}"
    swap: "{{ ansible_memtotal_mb | int * 0.25 }}"
```

### 8. Использование переменных окружения
```yaml
containers:
  - name: "env-container"
    vmid: 200
    users:
      - username: "{{ lookup('env', 'PROXMOX_USER') }}"
        password: "{{ lookup('env', 'PROXMOX_PASS') }}"
```

## Рекомендации по отладке

1. **Проверка значений переменных**:
```yaml
- name: Отладка переменных
  debug:
    msg: 
      - "Container ID: {{ vmid }}"
      - "Container Name: {{ name }}"
      - "Network Config: {{ net }}"
```

2. **Проверка условий**:
```yaml
- name: Проверка условий
  debug:
    msg: "Условие выполнено: {{ item }}"
  when: item is defined
  loop: "{{ containers }}"
```

3. **Логирование изменений**:
```yaml
- name: Логирование изменений
  debug:
    msg: "Изменения в контейнере {{ vmid }}: {{ item.changed }}"
  register: result
  when: item.changed
```

4. **Проверка ошибок**:
```yaml
- name: Обработка ошибок
  fail:
    msg: "Ошибка: {{ error_msg }}"
  when: error_msg is defined
``` 