# ПР №9. Следы вредоносного ПО в Linux

## 1. Что было посажено

| Механизм | Место | Команда/файл |

|----------|-------|-------------|

| Cron | crontab пользователя | /tmp/.hidden_malware/backdoor.sh |

| Systemd | ~/.config/systemd/user/ | system-helper.service |

| Shell profile | ~/.bashrc | /tmp/.hidden_malware/backdoor.sh |

| Процесс | ручной запуск | listener.sh на порту 4444 |

## 2. Что нашли — процессы

pohoje screen poteryal

## 3. Что нашли — сетевые соединения

Команда: ss -tulnp | grep 4444

Результат:

tcp   LISTEN  0  1  0.0.0.0:4444  0.0.0.0:*  users:(("nc",pid=4106,fd=3))

tcp   LISTEN  0  1  0.0.0.0:4444  0.0.0.0:*  users:(("nc",pid=3694,fd=3))

Подозрительный порт: 4444

Процесс: nc (netcat) с PID 4106 и 3694

Команда: sudo lsof -i :4444

Результат:

COMMAND  PID USER FD   TYPE DEVICE SIZE/OFF NODE NAME

nc      3694 user 3u  IPv4  54155   0t0  TCP *:4444 (LISTEN)

nc      4106 user 3u  IPv4  56119   0t0  TCP *:4444 (LISTEN)

Как lsof помогает связать порт с процессом: показывает имя процесса (nc), PID, пользователя, файловый дескриптор, порт и состояние.

## 4. Что нашли — автозапуск

### Cron

Команда: crontab -l

Результат:

@reboot /tmp/.hidden_malware/backdoor.sh &

@reboot /tmp/.hidden_malware/backdoor.sh &

*/5 * * * * /tmp/.hidden_malware/backdoor.sh &

Что подозрительно: скрипт из /tmp/.hidden_malware/, @reboot и */5

### Systemd

Команда: systemctl --user list-units --type=service --state=enabled

Найденный сервис: system-helper.service (enabled)

Содержимое unit-файла:

[UNIT]

Description=System Helper Service

After=default.target

[Service]

ExecStart=/tmp/.hidden_malware/backdoor.sh

Restart=always

[Install]

WantedBy=default.target

Что подозрительно: имя маскируется под системное, ExecStart из /tmp/.hidden_malware/, Restart=always

### ~/.bashrc

Команда: grep -n "hidden\|backdoor" ~/.bashrc

Результат:

/home/user/.bashrc:116:/tmp/.hidden_malware/backdoor.sh &

Где именно нашли: файл /home/user/.bashrc, строка 116

## 5. Итоговая таблица следов

| Место | Инструмент обнаружения | Что нашли |

|-------|----------------------|-----------|

| Процессы | ps aux | backdoor.sh, listener.sh из /tmp |

| Порт 4444 | ss -tulnp | nc (netcat) |

| Файлы процесса | lsof -p PID | (deleted) listener.sh |

| Cron | crontab -l | @reboot и */5 |

| Systemd | systemctl --user | system-helper.service |

| Bashrc | grep | строка 116 с backdoor.sh |

## 6. Связь с нормативкой (ФСТЭК №17)

| Мера | Как реализована |

|------|-----------------|

| АНЗ.2 | Обнаружение вредоносного кода в процессах и автозапуске |

| АУД.4 | Анализ безопасности — проверка процессов, портов, файлов |

| ЗИС.17 | Управление сетевыми соединениями — обнаружение порта 4444 |

## Выводы

В ходе работы были посажены и обнаружены следы «вредоносного» ПО: процессы из /tmp/.hidden_malware/, сетевой backdoor на порту 4444 (netcat), автозапуск в cron (@reboot, */5), пользовательский systemd-сервис (system-helper.service), модифицированный .bashrc (строка 116), техника маскировки через (deleted).

Самым неочевидным местом для нахождения вредоноса оказался пользовательский systemd-сервис в ~/.config/systemd/user/, так как он не требует root и многие администраторы его не проверяют.

## Контрольные вопросы

### 1. Назовите пять мест в Linux где вредонос может прописать автозапуск. Чем они отличаются по уровню привилегий?

Пять мест автозапуска:

- Cron (@reboot, */5) — уровень пользователя или root

- Systemd user-сервисы (~/.config/systemd/user/) — только пользователь

- Systemd system-сервисы (/etc/systemd/system/) — root

- ~/.bashrc, ~/.profile — уровень пользователя

- /etc/rc.local — root

Отличие по уровню привилегий: пользовательские места не требуют root и действуют только для конкретного пользователя. Системные места требуют root и действуют для всей системы.

### 2. Чем lsof отличается от ss? Когда использовать каждый?

- lsof (list open files) показывает все открытые файлы, включая сокеты. Позволяет увидеть, какие файлы держит процесс, включая удалённые (deleted).

- ss (socket statistics) показывает только сетевые соединения и порты. Быстрее работает.

Когда использовать: ss — для быстрого поиска открытых портов. lsof — для детального анализа процесса (какие файлы открыл, откуда запущен).

### 3. Что означает строка (deleted) в выводе lsof? Почему вредоносы используют этот трюк?

Строка (deleted) означает, что файл был удалён с диска, но процесс продолжает держать его открытым. Вредоносы используют этот трюк, чтобы скрыть своё присутствие — файла нет в файловой системе, его не найдут ls или find, но процесс продолжает работать.

### 4. Пользовательский systemd-сервис в ~/.config/systemd/user/ — нужен ли root чтобы его создать? Почему это опасно?

Нет, root не нужен. Любой пользователь может создать свой systemd-сервис. Это опасно, потому что вредонос может прописаться в автозапуск без прав root, и администратор может не проверить пользовательские сервисы.

### 5. Вредонос переименовал себя в systemd-journald. Как отличить настоящий системный процесс от поддельного?

- Посмотреть путь: настоящий systemd-journald находится в /usr/lib/systemd/systemd-journald, поддельный может быть в /tmp или /home

- Проверить цифровую подпись пакета

- Сравнить с оригиналом: dpkg -S /usr/lib/systemd/systemd-journald

- Посмотреть дерево процессов (pstree): настоящий процесс обычно запущен systemd (PID 1)

### 6. Напишите однострочник который находит все исполняемые файлы в /tmp и /var/tmp.

```bash

find /tmp /var/tmp -type f -executable 2>/dev/null
