# Backup-SQL Infrastructure Summary

## 📂 Systemd Mounts & Automounts
Мы отказались от `/etc/fstab` и перешли на systemd units, чтобы избежать проблем с «отвалами» при недоступности серверов. Теперь все CIFS и NFS ресурсы подключаются автоматически при обращении:

- `mnt-server.mount` / `mnt-server.automount` → CIFS `//172.16.0.9/R` → `/mnt/server`
- `mnt-sqlserver.mount` / `mnt-sqlserver.automount` → CIFS `//172.16.0.7/R` → `/mnt/sqlserver`
- `mnt-hotserver.mount` / `mnt-hotserver.automount` → CIFS `//172.16.0.6/R` → `/mnt/hotserver`
- `mnt-wd.mount` / `mnt-wd.automount` → NFS `172.16.0.3:/shares/backup` → `/mnt/wd`
- `mnt-synology.mount` / `mnt-synology.automount` → NFS `172.16.0.2:/volume1/backup` → `/mnt/synology`

> Все `.automount` юниты обеспечивают «ленивое» подключение: монтирование происходит только при обращении к папке.

---

## ⚙️ Backup Units

### backup-sql.service
Одноразовый сервис, запускающий наш скрипт резервного копирования.

```ini
[Unit]
Description=SQL Backup Archiver
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup_sql.sh
SyslogIdentifier=backup-sql
```

### backup-sql.timer
Таймер для ежедневного запуска:

```ini
[Unit]
Description=Run SQL Backup Archiver Daily

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

# /etc/systemd/system/mnt.target
[Unit]
Description=All mnt mounts

Wants=mnt-hotserver.mount
Wants=mnt-server.mount
Wants=mnt-sqlserver.mount
Wants=mnt-synology.mount
Wants=mnt-wd.mount

---

## 🚀 Активация юнитов

1. Перечитать конфигурацию:
   ```bash
   sudo systemctl daemon-reexec
   ```

2. Включить автомаунты:
   ```bash
   sudo systemctl enable --now mnt-{server,sqlserver,hotserver,wd,synology}.automount
   ```

3. Включить таймер:
   ```bash
   sudo systemctl enable --now backup-sql.timer
   ```

Проверить работу:
```bash
systemctl status backup-sql.service
journalctl -t backup-sql -r
```

---
## 📝 Зачем мы всё это сделали
- **NFS/CIFS автомаунты** через systemd → отказ от нестабильного `/etc/fstab`. Точки подключаются только при обращении.
- **backup-sql.service + timer** → ежедневное резервное копирование без ручного запуска.
- **backup-sql.sh** → потоковое сжатие `.bak` файлов в `.zip` (кроссплатформенно для Windows), логирование в journald.
- **root:root владельцы архивов** → гарантирует, что пользователи по сети не смогут случайно удалить копии.
- **Логи с меткой backup-sql** → удобная интеграция в систему мониторинга и уведомлений.

#SQL INSTRUCTION
EXEC master..xp_cmdshell 'R:\backup.bat'
#R:\backup.bat
set _in=R:\Backup\*.bak
set _out=R:\Backup
for %%i in (%_in%) do "C:\Program Files\WinRAR\rar.exe" a -ep -dw -m3 "%_out%\%%~ni.rar" "%%i"


set _in1=R:\Backup\*.rar
SET _out1=R:\"%date:~6,4%-%date:~3,2%-%date:~0,2%"
IF NOT EXIST "%_out1%" (mkdir %_out1%)
FOR %%i IN ("%_in1%") DO MOVE "%%i" "%_out1%"
