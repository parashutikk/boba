# ПР №3. Права доступа Linux и управление пользователями

## 1. Пользователи и группы

**Созданные пользователи и их роли:**

| Пользователь | Группа | Роль |
|---|---|---|
| alice | developers | Администратор проекта |
| bob | developers | Разработчик |
| carol | auditors | Аудитор |

**Поля /etc/passwd (разбор строки alice):**

`alice:x:1001:1003::/home/alice:/bin/bash`

- поле 1 `alice` — имя пользователя
- поле 2 `x` — пароль в /etc/shadow
- поле 3 `1001` — UID
- поле 4 `1003` — GID
- поле 5 (пусто) — комментарий
- поле 6 `/home/alice` — домашняя директория
- поле 7 `/bin/bash` — оболочка

## 2. Права доступа chmod/chown

**Итоговые права /srv/project/code:** `chmod 770`, владелец `alice:developers`

**Почему bob не смог создать файл при правах 750:**

При правах 750 (`rwxr-x---`) группа developers имеет `r-x` (чтение и выполнение, без записи). Bob в группе developers, поэтому не может создавать файлы — нет права `w`.

**Что нужно изменить:** `sudo chmod 770 /srv/project/code` или `sudo chmod g+w /srv/project/code`

## 3. ACL

**Задача:** дать carol доступ на чтение без изменения группы.

**Команда:** `setfacl -m u:carol:r-x /srv/project/code`

**Вывод getfacl:**

`file: home/user/srv/project/code`
`owner: alice`
`group: developers`
`user::rwx`
`user:carol:r-x`
`group::rwx`
`mask::rwx`
`other:---`

**Результаты:**
- carol читает (`ls`) — успешно
- carol создаёт (`touch`) — Permission denied

**Чем ACL лучше:** позволяет дать доступ конкретному пользователю без добавления в группу developers.

## 4. sudo-политики

**Результаты sudo -l -U (фото 4):**

| Пользователь | Разрешено |
|---|---|
| alice | (ALL) NOPASSWD: ALL |
| bob | (ALL) /usr/bin/apt, /usr/bin/apt-get |
| carol | (ALL) /usr/bin/journalctl, /bin/cat /var/log/* |

**Почему alice NOPASSWD, а bob нет:** alice — администратор, bob — разработчик, требует ввод пароля для безопасности.

**Принцип:** наименьших привилегий.

## 5. PAM

`password requisite pam_pwquality.so retry=3 minlen=12`

`minlen = 12`

`file: home/user/srv/project/code`
`owner: alice`
`group: developers`
`user::rwx`
`user:carol:r-x`
`group::rwx`
`mask::rwx`
`other:---`

**Модуль pam_unix.so в /etc/pam.d/sudo:** `auth required pam_unix.so`

**Что означает required:** модуль должен успешно выполниться, иначе аутентификация не пройдена.

**Как добавить минимальную длину пароля 12 символов:** в `/etc/security/pwquality.conf` добавить `minlen = 12`

## 6. Выводы

Изучены: управление пользователями, chmod/chown, ACL, sudo-политики, PAM. Все задачи выполнены.
