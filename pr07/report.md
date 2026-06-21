# ПР №7. AppArmor, Capabilities и Docker

## 1. Linux Capabilities

### Разбор cap_net_raw=ep

- cap_net_raw — право на raw-сокеты (нужен для ping)

- e (effective) — capability активна, ядро её проверяет

- p (permitted) — процесс может использовать эту capability

В Debian 12 стандартные утилиты (ping) не имеют capabilities, используется setuid-бит.

### Поиск файлов с capabilities

/usr/bin/dumpcap cap_net_admin,cap_net_raw=ep

/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper cap_net_bind_service,cap_net_admin,cap_sys_nice=ep

### CapPrm / CapEff / CapBnd (у обычного пользователя)

CapInh: 0000000000000000

CapPrm: 0000000000000000

CapEff: 0000000000000000

CapBnd: 000001fffffffffff

CapAmb: 0000000000000000

### Демонстрация setcap

До setcap: python3 не может привязаться к порту 80 (Permission denied)

После setcap cap_net_bind_service=ep: python3 успешно привязывается к порту 80 без sudo

Почему лучше чем sudo: даём только конкретное право, а не полный root

### Флаги e, i, p

- e (effective) — активна

- i (inheritable) — может передаваться дочерним процессам

- p (permitted) — разрешена к использованию

## 2. AppArmor

Количество профилей: enforce — 22, complain — 46

### Результаты pr07-reader

| Действие | Без профиля | complain | enforce |

|----------|-------------|----------|---------|

| Читать /tmp/pr07-allowed.txt | ✅ | ✅ | ✅ |

| Читать /etc/shadow | ✅ | ✅ | ❌ DENIED |

| Писать в /tmp/pr07-output.txt | ✅ | ✅ | ❌ DENIED |

| Писать в /etc/ | ❌ (ОС) | ❌ (ОС) | ❌ (ОС) |

### Логи AppArmor

В логах отсутствуют записи DENIED, так как доступ к /etc/shadow был заблокирован на уровне DAC (ОС), а не AppArmor.

## 3. Docker

### Сравнение хост vs контейнер

| Ресурс | Хост | Контейнер |

|--------|------|-----------|

| Корневая ФС | полная (/bin, /etc, /home, /root) | своя изолированная |

| Количество процессов | 263 | 1 |

| Сетевые интерфейсы | lo, ens33, docker0 | только lo |

| /etc/shadow хоста | доступен | не виден |

| Монтирование | разрешено | запрещено (без --privileged) |

### Volumes

- Монтирование папки /tmp/pr07-data в контейнер работает

- Контейнер не видит файлы хоста вне смонтированной папки

- Контейнер может записывать данные в смонтированную папку

### Сравнение capabilities

| Тип контейнера | CapEff | Что означает |

|----------------|--------|--------------|

| Обычный | 00000000a80425fb | 13 capabilities |

| --privileged | 000001ffffff | 35+ capabilities |

Расшифровка обычного контейнера:

cap_chown, cap_dac_override, cap_fowner, cap_fsetid, cap_kill, cap_setgid, cap_setuid, cap_net_bind_service, cap_net_raw, cap_sys_chroot, cap_mknod, cap_audit_write, cap_setfcap

Расшифровка привилегированного контейнера:

cap_chown, cap_dac_override, cap_dac_read_search, cap_fowner, cap_fsetid, cap_kill, cap_setgid, cap_setuid, cap_sys_module, cap_linux_immutable, cap_net_bind_service, cap_net_broadcast, cap_net_admin, cap_net_raw, cap_ipc_lock, cap_ipc_owner, cap_sys_admin, cap_sys_boot, cap_sys_nice, cap_sys_resource, cap_sys_time, cap_sys_tty_config, cap_mknod, cap_lease, cap_audit_write, cap_audit_control, cap_setfcap, cap_mac_override, cap_mac_admin, cap_syslog, cap_wake_alarm, cap_block_suspend, cap_audit_read, cap_perfmon, cap_bpf, cap_checkpoint_restore

### Запуск от непривилегированного пользователя

docker run --rm ubuntu:22.04 whoami → root

docker run --rm --user 1001:1002 ubuntu:22.04 id → uid=1001 gid=1002

apt-get install от не-root → Permission denied

### Итоговый nginx

Запуск:

docker run -d --name pr07-nginx --cap-drop ALL --cap-add NET_BIND_SERVICE --cap-add CHOWN --cap-add DAC_OVERRIDE --cap-add SETGID --cap-add SETUID -p 8080:80 nginx:alpine

Проверка: curl http://localhost:8080 → страница приветствия nginx

Capabilities процесса nginx:

CapEff: 0000000000000043

Расшифровка:

cap_chown, cap_dac_override, cap_setgid, cap_setuid, cap_net_bind_service

| Capability | Зачем нужна nginx |

|------------|-------------------|

| cap_net_bind_service | Привязка к порту 80 |

| cap_chown | Изменение владельца файлов логов |

| cap_dac_override | Обход прав
доступа к файлам |

| cap_setgid / cap_setuid | Смена группы/пользователя |

## 4. Эшелонированная защита

| Слой | Инструмент | Что ограничивает |

|------|-----------|------------------|

| DAC | chmod/chown | права доступа к файлам |

| Capabilities | --cap-drop ALL + cap-add | права привилегий процесса |

| MAC | AppArmor | доступ к файлам даже для root |

| Изоляция | Docker namespaces | видимость процессов, сети, ФС |

## Выводы

Настроена многоуровневая защита:

- Capabilities ограничивают права процесса (демонстрация через python3)

- AppArmor запрещает доступ к /etc/shadow

- Docker изолирует процесс в контейнере с минимальными capabilities

- Итоговый nginx запущен с 5 необходимыми capabilities вместо 35+
