# Сессионный проект по курсу «Безопасность GNU/Linux»: Настройка базовых средств защиты ОС GNU/Linux

Студент: Никита Костяев, номер **M265876**

ОС: РЕД ОС 8.0.3 (Standard Server Minimal), ядро 6.12.92, без графического интерфейса, виртуальная машина Oracle VirtualBox.

Все команды выполнялись в командной строке от имени root (`sudo -i`). Все настройки сохраняются после перезагрузки: это проверено перезагрузкой системы перед сбором артефактов.

## Выполнение заданий

| Раздел | Задание | Как выполнено | Подтверждение |
|---|---|---|---|
| 1.1 | Динамическая настройка сети (DHCP) | Интерфейс `enp0s3` настроен на DHCP в установщике; проверка: `nmcli -g ipv4.method con show enp0s3` → `auto`; `ip -4 addr` → адрес с пометкой `dynamic` | `history.out` |
| 1.2 | Имя хоста `mephi-2026.domain.local` | Задано в установщике; проверка: `hostnamectl` | `history.out`, `journalctl.out` |
| 1.3 | Проверка сетевой связности | `ping -c 4 8.8.8.8 > ping.out` | `ping.out`: 0% packet loss |
| 2.1 | Обновление дистрибутива | `dnf upgrade -y` | `dnf.out` |
| 2.2 | Установка nginx и libcap-ng-utils | `dnf install -y nginx libcap-ng-utils` | `dnf.out` |
| 2.3 | Установка локального RPM-пакета tcpdump | `dnf download tcpdump --destdir /tmp`<br>`rpm -ivh /tmp/tcpdump-*.x86_64.rpm` | `history.out` |
| 3.1 | Раздел на втором диске, ext4, метка `MEPHI_WEB` | `parted -s /dev/sdb mklabel gpt mkpart primary ext4 0% 100%`<br>`mkfs.ext4 -L MEPHI_WEB /dev/sdb1` | `history.out` |
| 3.2 | Монтирование в `/mephi-web` по метке | `mkdir /mephi-web`; запись в `/etc/fstab`:<br>`LABEL=MEPHI_WEB /mephi-web ext4 defaults 0 2`<br>`mount /mephi-web` | `fstab`, `stat.out` |
| 4.1 | Запуск nginx и автозапуск | `systemctl enable --now nginx` | `journalctl.out` |
| 4.2 | Журнал nginx за текущую загрузку | `journalctl -u nginx -b > journalctl.out` | `journalctl.out` |
| 5.1 | DAC: каталог проекта, разработчики и кураторы | Группа `developers` (GID 5500): `user1`, `user2`, `user3` (UID 5501–5503).<br>Группа `curators` (GID 4444): `curator1`, `curator2`.<br>`/data/mephi-2026`: владелец `root:developers`, права `2770` (бит set-GID: новые файлы получают группу `developers`).<br>ACL: `setfacl -m g:curators:rx`<br>ACL по умолчанию: `setfacl -d -m u::rwx,g::rwx,g:curators:rx,o::---`<br>При ACL по умолчанию `umask` не участвует в правах новых файлов, поэтому любой файл разработчика доступен всем разработчикам на чтение и запись, кураторам — только на чтение, остальным — недоступен | `stat.out`, `getfacl.out`, `passwd`, `group` |
| 5.2 | tcpdump без set-UID, запуск обычным пользователем | `chmod u-s /usr/sbin/tcpdump`<br>`setcap cap_net_raw,cap_net_admin=ep /usr/sbin/tcpdump`<br>Проверка: `sudo -u user1 tcpdump -c 5 -i any` — 5 пакетов захвачено | `getcap.out` |
| 5.3 | SELinux в режиме Enforcing, nginx отдаёт файлы из `/mephi-web` | `getenforce` → `Enforcing`, `SELINUX=enforcing` в `/etc/selinux/config`.<br>`semanage fcontext -a -t httpd_sys_content_t "/mephi-web(/.*)?"`<br>`restorecon -Rv /mephi-web`<br>В `/etc/nginx/nginx.conf`: `root /mephi-web;` | `getenforce.out`, `stat.out` (контекст `httpd_sys_content_t`), `curl.out` |
| 6.1 | Запрет локального входа кураторам | В `/etc/pam.d/login` добавлено `account required pam_access.so`.<br>В `/etc/security/access.conf`: `-:(curators):LOCAL`.<br>Проверка на tty2: вход `curator1` отклонён (`pam_access(login:account): access denied for user 'curator1' from 'tty2'`), вход `user1` успешен | `history.out` |
| 6.2 | Смена пароля каждые 90 дней, минимальная длина 12 символов | `/etc/login.defs`: `PASS_MAX_DAYS 90`; для существующего пользователя `chage -M 90`.<br>`/etc/security/pwquality.conf`: `minlen = 12` | `shadow` (максимальный срок 90), `pwquality.conf` |
| 7.1 | Веб-страница | `echo 'Hello from Student: M265876' > /mephi-web/index.html` | `curl.out` |
| 7.2 | Проверка доступности | `curl http://localhost/` → `Hello from Student: M265876` | `curl.out`, `mephi-screenshot.png` |
| 8 | Публикация результатов | Публичный репозиторий `mephi-session-project-2026` | Этот репозиторий |

## Состав репозитория

| Файл | Что проверяется |
|---|---|
| `mephi-screenshot.png` | Визуальное подтверждение: вывод `curl http://localhost/` |
| `history.out` | Выполненные команды |
| `ping.out` | Работоспособность сети |
| `dnf.out` | Управление пакетами |
| `stat.out` | Права доступа `/data/mephi-2026` и контекст SELinux `/mephi-web` |
| `getfacl.out` | ACL каталога проекта (дополнительно) |
| `journalctl.out` | Запуск веб-сервера |
| `getcap.out` | Настройка привилегий tcpdump |
| `getenforce.out` | Режим SELinux |
| `curl.out` | Результат тестирования |
| `fstab` | Монтирование файловых систем (`/etc/fstab`) |
| `passwd` | Пользователи (`/etc/passwd`) |
| `shadow` | Пароли пользователей (`/etc/shadow`) |
| `group` | Группы (`/etc/group`) |
| `pwquality.conf` | Настройка пароля (`/etc/security/pwquality.conf`) |

## Особенности выполнения

- В РЕД ОС 8 исполняемый файл tcpdump изначально не имел бита set-UID. Команда `chmod u-s` выполнена для явной гарантии его отсутствия, а запуск обычным пользователем обеспечен привилегиями `cap_net_raw` и `cap_net_admin`.
- `dnf download tcpdump` скачал пакеты для двух архитектур (i686 и x86_64). Установлен пакет x86_64, файл для i686 удалён.
- Установщик и система загружаются с параметром ядра `nomodeset` (закреплён через `grubby`): без него в VirtualBox экран машины не обновлялся после старта ядра.
