# Restoring the WireGuard and AmneziaWG handshake on Keenetic routers

Скрипт восстанавливает "lost" рукопожатие WireGuard (в том числе AmneziaWG).

---

## 📌 Requirements

- KeenOS версии 4.x
- Установленная среда [Entware](https://help.keenetic.com/hc/ru/articles/360021214160)
- Установленный пакет `curl`:
  ```sh
  opkg install curl
  ```

---
## 💾 Installation:

### ⚙️ Automatic mode

Отличия от варианта с `cron`:
- Срабатывает при изменении состояния любого интерфейса (например, при подключении/отключении интернета/WireGuard).
- Установка одной командой.
  ```sh
  curl -Ls "https://github.com/Ground-Zerro/Wireguard-DPI-blocking-bypass/raw/refs/heads/main/install.sh" | sh
  ```

---

### ⏱Working via Cron

1. Установите необходимые пакеты:
  ```sh
  opkg update && opkg upgrade
  opkg install cron nping
  ```

2. Отредактируйте файл `/opt/etc/crontab`. Пример:
  ```cron
  SHELL=/bin/sh
  PATH=/sbin:/bin:/usr/sbin:/usr/bin:/opt/bin:/opt/sbin
  MAILTO=""
  HOME=/
  */1 * * * * root /opt/bin/run-parts /opt/etc/cron.1min
  ```

3. Создайте скрипт `/opt/etc/cron.1min/pinger` со следующим содержимым:
  ```sh
  #!/bin/sh
  PATH=/opt/sbin:/opt/bin:/opt/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

  if ps | grep -v grep | grep "wgpass.sh" | grep -qv "$$"; then
      exit 0
  fi

  gnip()
  {
  ! ping -I nwg$1 -s0 -qc1 -W1 1.1.1.1 >/dev/null 2>&1
  }

  for i in $(ip a | sed -n 's/.*nwg\(.*\): <.*UP.*/\1/p'); do
    rem=$(echo $(ndmc -c "show interface Wireguard$i" | sed -n 's/.*remote.*: \(.*\)/\1/p'))
    echo $rem | grep -q '^0\| 0' && continue
    if gnip $i && gnip $i && gnip $i && gnip $i; then
      echo "START bypass for Wireguard$i."
      port=$(hexdump -n2 -e '1/2 "%u\n"' /dev/urandom | awk '{print ($1 % 63000) + 2000}')
      while netstat -nlu | grep -qw $port; do
          port=$(hexdump -n2 -e '1/2 "%u\n"' /dev/urandom | awk '{print ($1 % 63000) + 2000}')
      done >/dev/null 2>&1
      count=$(hexdump -n1 -e '1/1 "%u\n"' /dev/urandom | awk '{print ($1 % 5) + 6}')
      length=$(hexdump -n2 -e '1/2 "%u\n"' /dev/urandom | awk '{print ($1 % 65) + 64}')
      nping --udp --count $count --source-port $port --data-length $length --dest-port $(echo $rem | cut -f2 -d' ') ${rem%% *} >/dev/null 2>&1
      ndmc -c "interface Wireguard$i wireguard listen-port $port" >/dev/null 2>&1
      echo "Bypass for Wireguard$i - COMPLETE."
    fi
  done
  ```

4. Сделайте скрипт исполняемым:
  ```sh
  chmod 755 /opt/etc/cron.1min/pinger
  ```

5. Запустите cron:
  ```sh
  /opt/etc/init.d/S10cron start
  ```

<details>
  <summary>📖 Команды cron</summary>

  ```sh
  /opt/etc/init.d/S10cron start|stop|restart|check|status|kill|reconfigure
  ```
</details>

---

## 🔍 Logic of the script

- Проверяет доступность `1.1.1.1` через каждый включенный (поднятый/UP) интерфейс WireGuard (`nwgX`)
- Если пинг не проходит **4 раза подряд**:
  - Генерируется случайный UDP-порт (2000–65000)
  - Проверяется его незанятость
  - Выполняется случайное число UDP-запросов со случайным размером данных в пакетах (`nping`) с новым портом на текущий WG-пир
  - Меняется локальный listen-порт WireGuard интерфейса

> Пропускаются интерфейсы с `remote: 0.0.0.0 0` — это, как правило, серверные WG-интерфейсы без настроенного пира

---

## 👤 Author of the idea

**Frans**  
Источник: [Форум Keenetic](https://forum.keenetic.ru/topic/19389-обход-блокировки-протокола-wireguard-в-тч-amneziawg/?do=findComment&comment=193941)
