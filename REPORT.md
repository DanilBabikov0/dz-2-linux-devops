# Отчёт по домашней работе

Бабиков Данил Александрович

## Проблема 1: сервис падает с status=203/EXEC неверный ExecStart

### Что было обнаружено

Первая команда диагностики `systemctl status homework-app.service` показала:
```bash
danilbabikov@DEll-Danil:~/dz-2$ systemctl status homework-app.service
● homework-app.service - Linux DevOps Homework Service
     Loaded: loaded (/etc/systemd/system/homework-app.service; enabled; preset: enabled)
     Active: activating (auto-restart) (Result: exit-code) since Wed 2026-10-07 20:15:55 +05; 587ms ago
    Process: 4209 ExecStart=/opt/linux-devops-homework/app.py (code=exited, status=203/EXEC)
   Main PID: 4209 (code=exited, status=203/EXEC)
        CPU: 749us
```

Сервис в бесконечном цикле перезапуска, код 203/EXEC означает, что systemd не смог выполнить команду из ExecStart.

### Как проводилась диагностика

Посмотрел, что именно запускает systemd, и что реально лежит в каталоге приложения.

```bash
systemctl cat homework-app.service
ls -la /opt/linux-devops-homework/
ls -la app/
head -5 /opt/linux-devops-homework/server.py
```


### В чём была причина

В unit-файле в ExecStart указано неверное имя файла: app.py вместо реально существующего server.py. systemd пытался запустить несуществующий файл и падал с 203/EXEC.


### Что было изменено

Файл systemd/homework-app.service: заменил строку

```bash
ExecStart=/opt/linux-devops-homework/app.py
```

на

```bash
ExecStart=/opt/linux-devops-homework/server.py
```


### Почему было выбрано это решение

systemctl cat и ls показали расхождение между тем, что запускается, и тем, что есть на диске. Единственное обоснованное изменение привести ExecStart в соответствие с реальным именем файла. Никакие права или содержимое приложения менять не требовалось.


### Как проверялся результат

```bash
systemctl status homework-app.service
```

Код 203/EXEC пропал, сервис пошёл дальше по запуску и упал уже с другой ошибкой

---

## Проблема 2: PermissionError на /var/lib/linux-devops-homework/startup.log

### Что было обнаружено

В логе сервиса после исправления 1 ошибки:

```bash
PermissionError: [Errno 13] Permission denied: '/var/lib/linux-devops-homework/startup.log'
```

Приложение не могло открыть файл для записи в каталоге состояния.


### Как проводилась диагностика

```bash
ls -ld /var/lib/linux-devops-homework
```
```bash
drwx------ 2 root root ... /var/lib/linux-devops-homework
```

Каталог принадлежит root:root с правами 0700. Приложение работает от пользователя homework, а значит, не имеет доступа ни на чтение, ни на запись.


### В чём была причина

Каталог состояния был создан setup.sh намеренно с владельцем root:root и правами 0700. Сервис работает от homework, и не мог писать в этот каталог.


### Что было изменено

reset-failed понадобился, потому что systemd заблокировал сервис из-за частых падений

```bash
sudo chown homework:homework /var/lib/linux-devops-homework
sudo chmod 0750 /var/lib/linux-devops-homework
sudo chown -R homework:homework /var/lib/linux-devops-homework
sudo systemctl reset-failed homework-app.service
sudo systemctl restart homework-app.service
```


### Почему было выбрано это решение

Владелец и права каталога то, чего требует задание. chown + chmod — минимально необходимое изменение.


### Как проверялся результат

```bash
ls -ld /var/lib/linux-devops-homework
# drwxr-x--- 2 homework homework ...
systemctl status homework-app.service
# Active: active (running)
```

Сервис перешёл в состояние active (running), PermissionError исчез из логов.

---


## Проблема 3: приложение слушает только 127.0.0.1:8080

### Что было обнаружено

```bash
sudo ss -tlnp | grep 8080
# LISTEN 0 5 127.0.0.1:8080 0.0.0.0:* users:(("python3",pid=5467,fd=3))
```

Сервис работал, но слушал только 127.0.0.1:8080. По заданию требуется 0.0.0.0:8080


### Как проводилась диагностика

```bash
grep -n -i 'config\|app.conf' app/server.py
# 7:CONFIG = Path('/etc/linux-devops-homework/app.conf')
cat /etc/linux-devops-homework/app.conf
# [server]
# host = 127.0.0.1
# port = 8080
```

Приложение читает конфиг из /etc/linux-devops-homework/app.conf, а не из репозитория


### В чём была причина

Значение host = 127.0.0.1 не соответствует конечному состоянию, где требуется 0.0.0.0.


### Что было изменено

В config/app.conf: заменил host = 127.0.0.1 на host = 0.0.0.0.

Скопировал исправленный конфиг в реальный путь, откуда его читает приложение

```bash
sudo cp config/app.conf /etc/linux-devops-homework/app.conf
sudo chown root:root /etc/linux-devops-homework/app.conf
sudo chmod 0644 /etc/linux-devops-homework/app.conf
sudo systemctl restart homework-app.service
```


### Почему было выбрано это решение

127.0.0.1 — это адрес, доступный только с самого хоста. 0.0.0.0 означает «слушать на всех IPv4-интерфейсах», и тогда сервис доступен извне, что и требуется.


### Как проверялся результат

```bash
sudo ss -tlnp | grep 8080
# LISTEN 0 5 0.0.0.0:8080 0.0.0.0:* users:(("python3",pid=...,fd=...))
curl -v http://127.0.0.1:8080/health
# HTTP/1.0 200 OK ... {"status": "ok"}
```

Приложение перешло на 0.0.0.0:8080, health-эндпоинт отвечает как ожидается.

---


## Проблема 4: ExecStart не соответствует ожидаемому пути /opt/linux-devops-homework/app/server.py

### Что было обнаружено

После первых исправлений check.sh показал:

```bash
[FAIL] systemd ExecStart is incorrect
       Next: Inspect: systemctl status homework-app; systemctl cat homework-app; ls -l /opt/linux-devops-homework /opt/linux-devops-homework/app
[FAIL] repository still contains the original broken unit
```


### Как проводилась диагностика

Заглянул в check.sh, чтобы понять, какой именно ExecStart ожидается:

```bash
grep -n -i 'execstart\|/opt/linux' scripts/check.sh
```

Checker ожидает ExecStart=/opt/linux-devops-homework/app/server.py то есть файл приложения должен лежать в подкаталоге app/.

Затем посмотрел scripts/setup.sh, чтобы понять, как он устанавливает приложение:

```bash
grep -n 'install\|server.py\|APP_DIR' scripts/setup.sh
```
setup.sh кладёт файл по пути ${APP_DIR}/server.py то есть в корень /opt/linux-devops-homework/, а не в app/. Значит, потребуется создать подкаталог и скопировать туда файл.


### В чём была причина

Путь в ExecStart не совпадали с требуемым конечным состоянием: файл должен находиться в /opt/linux-devops-homework/app/server.py, а setup.sh создаёт его в /opt/linux-devops-homework/server.py.


### Что было изменено

В systemd/homework-app.service: ExecStart=/opt/linux-devops-homework/app/server.py

```bash
sudo mkdir -p /opt/linux-devops-homework/app
sudo cp /opt/linux-devops-homework/server.py /opt/linux-devops-homework/app/server.py
sudo chown -R root:root /opt/linux-devops-homework/app
sudo chmod 0755 /opt/linux-devops-homework/app
sudo chmod 0755 /opt/linux-devops-homework/app/server.py
sudo cp systemd/homework-app.service /etc/systemd/system/homework-app.service
sudo systemctl daemon-reload
sudo systemctl restart homework-app.service
```


### Почему было выбрано это решение

check.sh ищет в ExecStart подстроку /opt/linux-devops-homework/app/server.py. Значит, конечное состояние требует именно такого пути. Файл создаётся копированием в нужный каталог с правами 0755 — этого достаточно, потому что запускается он из-под systemd от пользователя homework.


### Как проверялся результат

```bash
sudo bash scripts/check.sh
# Result: 10 passed, 0 failed
```

Все 10 проверок пройдены.

---
<!-- Скопируйте секцию «Проблема 1» для каждой следующей обнаруженной проблемы. -->

## Дополнительные наблюдения

### От какого пользователя фактически работает процесс:

```bash
ps -o user,pid,ppid,cmd -p $(pgrep -f 'server.py' | head -1)
# USER=homework  PID=...  PPID=1  CMD=python3 /opt/linux-devops-homework/app/server.py
```
Подтверждение через /proc:

/proc/<PID>/status — UID процесса 999, GID 989 (соответствуют пользователю и группе homework).

/proc/<PID>/cmdline — точная команда запуска, начинается с python3 /opt/linux-devops-homework/app/server.py.

/proc/<PID>/fd/ — открытые файлы и сокеты.


## Итоговый ход диагностики

1. systemctl status показал 203/EXEC → проблема в ExecStart → исправил путь в unit-файле.

2. journalctl показал PermissionError на /var/lib/linux-devops-homework/startup.log → исправил владельца и права каталога состояния.

3. ss -tlnp показал 127.0.0.1:8080 вместо 0.0.0.0:8080 → нашёл конфиг через grep по коду и заменил host в app.conf.

4. check.sh указал на несоответствие ExecStart ожидаемому пути → перенёс файл приложения в подпапку app/ и обновил unit.

Каждая проверка проводилась после перезапуска сервиса; следующая ошибка появлялась только когда предыдущая была действительно устранена.


## Итоговая проверка

```bash
sudo ./scripts/check.sh
```

```text
Linux DevOps Homework Checker

[PASS] systemd ExecStart points to the application
[PASS] service runs as homework:homework
[PASS] state directory ownership and permissions are correct
[PASS] service is enabled
[PASS] service is active
[PASS] running process has the expected UID
[PASS] TCP/8080 listens on 0.0.0.0
[PASS] GET /health returns the expected response
[PASS] fixed systemd unit is saved in the repository
[PASS] fixed application configuration is saved in the repository

Result: 10 passed, 0 failed
```
