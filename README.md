# SetupVps

Ansible-проект для первоначальной настройки, bootstrap и базового hardening Ubuntu VPS.

Главный принцип проекта — **каждый сервер описывается отдельно через `host_vars`**. Это особенно важно для первоначального подключения: пароль используется только один раз, чтобы Ansible смог подключиться к чистому VPS и выполнить настройку. После применения hardening вход по паролю отключается, и дальнейшая работа выполняется по SSH-ключу.

## Что делает проект

Текущий playbook применяет набор ролей к группе `nodes`:

- `users` — создаёт системных пользователей, устанавливает их SSH public keys и настраивает sudo без пароля;
- `apt_update` — обновляет систему;
- `devsec.hardening.ssh_hardening` — выполняет hardening SSH;
- `swapfile` — настраивает swap;
- `downloads` — выполняет необходимые загрузки.

Основной playbook находится в `site.yml`.

## Структура

```text
SetupVps/
├── inventory.ini
├── host_vars/
│   ├── germanyNode.yml
│   ├── finlandNode.yml
│   └── moscowNode.yml
├── site.yml
├── files/
│   └── ssh/
└── roles/
    ├── users/
    │   ├── defaults/
    │   └── tasks/
    ├── apt_update/
    ├── swapfile/
    └── downloads/
```

## Установка

### 1. Установить Ansible

На машине, с которой будет запускаться Ansible, должен быть установлен Ansible.

Для Ubuntu/Debian:

```bash
sudo apt update
sudo apt install -y ansible
```

Проверить установку:

```bash
ansible --version
```

### 2. Клонировать проект

```bash
git clone https://github.com/PeaceDeath04/SetupVps.git
cd SetupVps
```

### 3. Установить коллекцию devsec.hardening

Проект использует роль `devsec.hardening.ssh_hardening`, поэтому перед первым запуском необходимо установить коллекцию из Ansible Galaxy:

```bash
ansible-galaxy collection install devsec.hardening
```

Проверить установленные коллекции:

```bash
ansible-galaxy collection list
```

### 4. Создать host_vars

Каталог `host_vars/` **не хранится в Git**, потому что в нём находятся индивидуальные настройки серверов и первоначальные пароли.

После клонирования проекта его нужно создать вручную:

```bash
mkdir -p host_vars
```

Для каждого сервера создаётся отдельный файл, имя которого должно совпадать с hostname из `inventory.ini`:

```text
host_vars/
├── germanyNode.yml
├── finlandNode.yml
└── moscowNode.yml
```

Например, для нового VPS:

```yaml
ansible_host: 203.0.113.10
ansible_user: root
ansible_port: 22
ansible_password: "PASSWORD"
```

> **Важно:** этот файл содержит пароль для первоначального подключения и не должен добавляться в Git.

### 5. Подготовить SSH keys

Создайте каталог для public keys, если его ещё нет:

```bash
mkdir -p files/ssh
```

В него нужно поместить **публичные SSH-ключи** пользователей, которые указаны в роли `users`.

Например:

```text
files/ssh/
├── ansible_ed25519.pub
├── peacedeath_ed25519.pub
└── lovushka_ed25519.pub
```

Имена файлов должны соответствовать значениям `ssh_key` в настройках роли `users`.

Приватные SSH-ключи (`id_ed25519`, `ansible_ed25519` и т. п.) **в этот каталог не помещаются** и никогда не должны попадать в GitHub. Они остаются только на компьютере, с которого выполняется Ansible.

### 6. Проверить inventory

После создания `host_vars` убедитесь, что hostname совпадает в обоих местах:

```text
inventory.ini
    ↓
germanyNode
    ↓
host_vars/germanyNode.yml
```

После этого можно выполнять первый запуск.

## Inventory

В `inventory.ini` указываются только хосты и их принадлежность к группе:

```ini
[nodes]
germanyNode
finlandNode
moscowNode
```

Имена хостов должны соответствовать файлам в `host_vars/`.

```text
inventory.ini
    │
    └── [nodes]
          ├── germanyNode ──→ host_vars/germanyNode.yml
          ├── finlandNode ──→ host_vars/finlandNode.yml
          └── moscowNode ──→ host_vars/moscowNode.yml
```

## Host vars

Все параметры, которые могут отличаться между серверами, задаются через `host_vars/<hostname>.yml`.

В том числе параметры первоначального SSH-подключения:

```yaml
ansible_host: 203.0.113.10
ansible_user: root
ansible_port: 22
ansible_password: "PASSWORD"
```

Пароль здесь нужен **только для первого запуска**.

После выполнения playbook SSH hardening отключает вход по паролю. Далее сервер должен быть доступен по SSH public key, поэтому пароль из `host_vars` больше не используется.

После успешной настройки пароль можно удалить из host vars:

```yaml
ansible_host: 203.0.113.10
ansible_user: ansible
ansible_port: 36837
```

Конкретные значения `ansible_user`, `ansible_port` и других параметров зависят от итоговой конфигурации сервера.

### Почему не group_vars

Доступ к каждому VPS может отличаться:

- разные IP;
- разные SSH-порты;
- разные пользователи;
- разные первоначальные пароли;
- разные параметры, которые должны применяться только к одному серверу.

Поэтому настройки подключения и host-specific параметры не должны находиться в общей конфигурации группы.

## Первый запуск

Первый запуск выполняется с учётными данными, которые находятся в `host_vars/<hostname>.yml`.

Проверить доступность серверов:

```bash
ansible nodes -i inventory.ini -m ping
```

Проверить playbook без внесения изменений:

```bash
ansible-playbook -i inventory.ini site.yml --check
```

Применить конфигурацию:

```bash
ansible-playbook -i inventory.ini site.yml
```

Если нужно настроить только один сервер:

```bash
ansible-playbook -i inventory.ini site.yml --limit germanyNode
```

## Что происходит после первого запуска

Первоначальная схема:

```text
VPS
 │
 │ SSH password
 ▼
Ansible
 │
 ├── создаёт пользователей
 ├── устанавливает SSH public keys
 ├── настраивает sudo
 ├── меняет SSH configuration
 └── отключает PasswordAuthentication
          │
          ▼
      пароль больше не нужен
          │
          ▼
      SSH public key
```

То есть пароль не является постоянным способом доступа к серверу. Он нужен только как средство первоначального bootstrap.

После применения hardening дальнейшие запуски Ansible выполняются уже по SSH-ключу.

## SSH hardening

Проект настраивает SSH с использованием переменных роли `devsec.hardening.ssh_hardening`.

Основные настройки:

- SSH port: `36837`;
- разрешён вход только пользователям, указанным в `ssh_allow_users`;
- запрещён прямой SSH-вход под `root`;
- вход по паролю отключён;
- разрешена аутентификация по public key;
- `publickey` используется как основной метод аутентификации;
- ограничены SSH forwarding-возможности;
- отключены GSSAPI и ненужные X11/MOTD-настройки;
- включён более подробный SSH logging.

Перед применением SSH hardening необходимо убедиться, что созданный пользователь имеет рабочий SSH public key и что новый SSH-порт доступен.

**Не закрывай текущую SSH-сессию, пока не проверишь новое подключение.**

## Users role

Пользователи описываются данными, а не отдельными блоками задач:

```yaml
users:
  - name: ansible
    ssh_key: "files/ssh/ansible_ed25519.pub"
    sudo_nopasswd: true

  - name: peacedeath
    ssh_key: "files/ssh/peacedeath_ed25519.pub"
    sudo_nopasswd: true

  - name: lovushka
    ssh_key: "files/ssh/lovushka_ed25519.pub"
    sudo_nopasswd: true
```

Это позволяет добавлять пользователей в список без копирования одних и тех же Ansible tasks.

Важно: `ssh_allow_users` и список `users` решают разные задачи.

- `users` создаёт пользователя и устанавливает его SSH key;
- `ssh_allow_users` определяет, кому SSH разрешено подключаться.

## SSH keys

В репозитории хранятся только public keys:

```text
files/ssh/
├── ansible_ed25519.pub
├── peacedeath_ed25519.pub
└── lovushka_ed25519.pub
```

Приватные SSH-ключи никогда не должны попадать в GitHub.

После первого запуска подключение к серверу должно выполняться с использованием соответствующего приватного ключа.

Например:

```bash
ssh -p 36837 ansible@SERVER_IP
```

## Пароли и секреты

Первоначальный пароль сервера является временным bootstrap-секретом.

Если пароль хранится в:

```text
host_vars/<hostname>.yml
```

этот файл **не должен попадать в публичный Git-репозиторий**.

Рекомендуется добавить реальные host vars с паролями в `.gitignore` и хранить в репозитории только безопасные примеры.

Например:

```text
host_vars/
├── germanyNode.yml
├── finlandNode.yml
└── moscowNode.yml
```

может существовать локально, а в Git храниться только пример:

```text
host_vars/
└── example.yml
```

После первого успешного запуска пароль можно удалить из конкретного host vars, поскольку PasswordAuthentication на сервере уже отключён.

## Повторный запуск

После первоначального bootstrap Ansible больше не должен зависеть от пароля.

Подключение выполняется по SSH public key:

```yaml
ansible_user: ansible
ansible_port: 36837
```

При необходимости можно явно указать приватный ключ:

```yaml
ansible_ssh_private_key_file: ~/.ssh/ansible_ed25519
```

Таким образом, жизненный цикл сервера выглядит так:

```text
1. Новый VPS
      ↓
2. host_vars/<hostname>.yml
      ↓
3. SSH password
      ↓
4. Первый запуск Ansible
      ↓
5. Создание пользователя + SSH key
      ↓
6. SSH hardening
      ↓
7. PasswordAuthentication = no
      ↓
8. Удаление временного пароля из host_vars
      ↓
9. Дальнейшая работа по SSH key
```

## Зависимости

Проект использует коллекцию:

```text
devsec.hardening
```

В частности:

```text
devsec.hardening.ssh_hardening
```

Также используются стандартные коллекции Ansible для работы с пользователями, SSH-ключами и системными настройками.

## План развития

Проект будет постепенно расширяться:

- дополнительные роли для bootstrap VPS;
- настройка Docker;
- дополнительные системные настройки;
- автоматизация установки сервисов;
- проверки конфигурации перед применением;
- дальнейшее разделение инфраструктуры на независимые роли.

## Принцип проекта

Главная идея — описывать **состояние сервера**, а не вручную выполнять последовательность команд на каждом VPS.

Каждый сервер имеет собственные параметры в `host_vars`, а общая логика остаётся в playbook и roles.

```text
inventory
    ↓
host_vars/<hostname>.yml
    ↓
site.yml
    ↓
roles
    ↓
Ubuntu VPS
```

Пароль используется только на этапе первоначального bootstrap. После hardening сервер работает без SSH-доступа по паролю.
