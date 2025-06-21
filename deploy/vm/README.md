# Установка виртуальной машины
Процесс установки среды виртуализации не рассматривается в данной статье. Перед продолжением убедитесь, что:
1. на компьютере установлена подходящая среда виртуализации (VirtualBox, Qemu, UTM, etc.)
2. в среде виртуализации установлена Unix-based ОС (Ubuntu, CentOS, etc.)
3. на компьютере установлен SSH-клиент и сгенерирован SSH-ключ.

## Настройка локального ~/.ssh/config
Для упрощения подключения к ВМ добавьте в файл `~/.ssh/config` запись ниже, заменив данные на свои:
```
Host lh-ubuntu
   HostName 192.168.64.3
   User wdl
```

# Настройка окружения Ansible
Для работы с Ansible необходимо чтобы на компьютере была установлена версия Python указанная в [pyproject.toml](../../pyproject.toml), 
а так же менеджер зависимостей Poetry. Установите их доступным для вашей ОС способом прежде, чем продолжить.

## Создание окружения .venv
Перейдите в каталог проекта и выполните команду
```shell
> poetry install --no-root
```

## Установка SSH-ключа
```shell
> cd deploy/vm/
> ansible-playbook playbooks/config_authorized_key.yml
```
Если всё прошло успешно - то вы сможете выполнять команды ansible без запроса пароля. Пример:
```shell
> ssh-add
Enter passphrase for /Users/irc/.ssh/id_rsa: 
Identity added: /Users/irc/.ssh/id_rsa (irc@Kirills-MacBook-Pro.local)

> ansible db -m ping
lh-ubuntu | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.12"
    },
    "changed": false,
    "ping": "pong"
}
```

# Установка PostgreSQL
```shell
> cd deploy/vm/
> ansible-playbook playbooks/install_db.yml
```
