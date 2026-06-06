# portstat

Мониторинг подключений и трафика по портам в реальном времени для VPN/прокси-серверов и ни только. Считает по каждому сетевому порту:
число подключений, число уникальных IP, накопленный трафик и текущую скорость — отдельно
для входящего и исходящего, плюс общие итоги.

Работает поверх **conntrack**, поэтому видит и TCP, и UDP-сервисы (обычный `ss` UDP-клиентов
не показывает).

Вывод разбит по **процессу** (`NAME`) и **состоянию TCP-соединения**
(`STATUS`: `ESTAB`, `TIME-WAIT`, `FIN-WAIT-2` и т.д.), а скорость показывается
раздельно вверх/вниз (`↑ / ↓`).

<img width="997" height="1091" alt="image" src="https://github.com/user-attachments/assets/4cb670c0-422f-43a4-8fa1-68c8fa3308eb" />

---

## Требования

| Компонент | Пакет | Назначение |
|-----------|-------|------------|
| `conntrack` | `conntrack-tools` (бинарь `conntrack`) | учёт TCP+UDP потоков и байт |
| `ss`, `ip` | `iproute2` | список портов и IP сервера |
| `watch` | `procps` / `procps-ng` | автообновление вывода |
| `awk`, `sed`, `grep`, `sort`, `paste` | coreutils + awk | обработка (есть почти всегда) |
| `bash` | bash | сам скрипт |
| ядро Linux | модуль `nf_conntrack` | отслеживание соединений |

Скрипт нужно запускать **от root** (`conntrack -L` требует прав, `ss -p` — для имён процессов).

---

## Установка

### 1. Поставить зависимости

**Debian / Ubuntu:**
```bash
sudo apt update
sudo apt install -y conntrack iproute2 procps
```

**RHEL / CentOS / Rocky / Alma / Fedora:**
```bash
sudo dnf install -y conntrack-tools iproute procps-ng
```

**Alpine:**
```bash
sudo apk add conntrack-tools iproute2 procps bash
```

**Arch:**
```bash
sudo pacman -S --needed conntrack-tools iproute2 procps-ng
```

### 2. Включить учёт байт в conntrack

Без этого колонки `TRAFFIC` и `SPEED` будут нулевыми (подключения и IP считаются и так).

```bash
# загрузить модуль (и сделать автозагрузку при ребуте)
sudo modprobe nf_conntrack
echo nf_conntrack | sudo tee /etc/modules-load.d/conntrack.conf

# включить учёт байт/пакетов и сохранить после перезагрузки
echo 'net.netfilter.nf_conntrack_acct=1' | sudo tee /etc/sysctl.d/99-conntrack-acct.conf
sudo sysctl --system
```

Проверка, что учёт включён (должно быть `= 1`):
```bash
sysctl net.netfilter.nf_conntrack_acct
```

### 3. Установить скрипт

```bash
sudo curl -fsSL https://raw.githubusercontent.com/CABi-Code/ServerTrafficMonitor/main/portstat -o /usr/local/bin/portstat
sudo chmod +x /usr/local/bin/portstat
```

### 4. Добавить алиас

Под root:
```bash
echo "alias portstat='watch -n 2 /usr/local/bin/portstat'" >> ~/.bashrc && source ~/.bashrc
```

Под обычным пользователем (conntrack требует root):
```bash
echo "alias portstat='watch -n 2 sudo /usr/local/bin/portstat'" >> ~/.bashrc && source ~/.bashrc
```

---

## Запуск

```bash
portstat
```
или разово, без автообновления:
```bash
sudo /usr/local/bin/portstat
```

Выход — `Ctrl+C`.

---

## Команды одним блоком УСТАНОВКА (Debian/Ubuntu, под root)

```bash
apt update && apt install -y conntrack iproute2 procps
modprobe nf_conntrack
echo nf_conntrack > /etc/modules-load.d/conntrack.conf
echo 'net.netfilter.nf_conntrack_acct=1' > /etc/sysctl.d/99-conntrack-acct.conf
sysctl --system
curl -fsSL https://raw.githubusercontent.com/CABi-Code/ServerTrafficMonitor/main/portstat -o /usr/local/bin/portstat
chmod +x /usr/local/bin/portstat
echo "alias portstat='watch -n 2 /usr/local/bin/portstat'" >> ~/.bashrc && source ~/.bashrc
portstat
```

## Команды для ОБНОВЛЕНИЯ (если уже установили)

```bash
sudo curl -fsSL https://raw.githubusercontent.com/CABi-Code/ServerTrafficMonitor/main/portstat -o /usr/local/bin/portstat
sudo chmod +x /usr/local/bin/portstat
sudo rm -f /tmp/portstat.state
portstat
```

---

## Важные замечания

- **Скорость появляется со второго обновления.** Она считается как разница байт между
  двумя соседними тиками `watch`, делённая на прошедшее время. Поэтому при самом первом
  выводе `SPEED` = `0.0B/s`. Состояние хранится в `/tmp/portstat.state`.

- **Скорость без всплесков, но слегка занижена.** В расчёт берутся только потоки, которые
  были в обоих замерах. Новые и закрывшиеся соединения в скорость не попадают — это убирает
  ложные скачки в десятки GiB/s, но немного занижает значение на очень бурном трафике.

- **`CONN / IP`** — число соединений (потоков) и число уникальных IP на порту. Один клиент
  обычно держит несколько соединений (Telegram, браузер, NAT на стороне клиента), поэтому
  `CONN` всегда заметно больше `IP` — это нормально.

- **Показываются только сетевые порты.** Сервисы, привязанные к `127.0.0.1`/`[::1]`,
  и приватные/Docker-адреса (`10.x`, `172.16–31.x`, `192.168.x`) отфильтрованы. Порт
  появляется в таблице, только если по нему есть активные соединения.

- **Сервер за NAT (только приватный IP на интерфейсе).** Если на сетевом интерфейсе нет
  публичного адреса (бывает на некоторых облаках, где публичный IP — это NAT провайдера),
  фильтр приватных IP оставит список пустым. В этом случае уберите в скрипте строку
  `grep -vE '^(10\.|172\.(1[6-9]...` в блоке `SRV`.

- **`conntrack -L` пустой?** Значит conntrack не отслеживает трафик. На серверах с Docker,
  UFW, firewalld или nftables он включается сам. На «голом» сервере без файрвола может
  потребоваться задействовать трекинг (например, добавить в nftables правило с `ct state`,
  либо включить любой файрвол).

- **UDP-потоки протухают по таймауту** (~30–120 с простоя), поэтому счётчик по UDP-портам
  отражает недавно активных клиентов, а не «всех за всё время».

- **Трафик/скорость считаются по conntrack** и точны, пока поток отслеживается. Для точного
  суммарного учёта за всё время (например, по WireGuard) лучше дополнительно смотреть родные
  счётчики: `wg show <iface> transfer` или per-port счётчики в nftables.

  - **`STATUS` и `NAME`.** Состояние соединения и имя процесса берутся из `ss` и
  накладываются на потоки `conntrack` по 4-кортежу (`src:sport`/`dst:dport`).
  Благодаря `ss` различаются `FIN-WAIT-1` и `FIN-WAIT-2` — в самом `conntrack`
  оба свёрнуты в общий `FIN_WAIT`. Имена процессов (`ss -p`) требуют root.

- **`SPEED` раздельно `↑ / ↓`.** `↑` — трафик в сторону сервиса (клиент → сервер,
  upload клиента), `↓` — обратно (сервер → клиент, download клиента). Для прокси/VPN
  `↓` обычно заметно больше `↑`.

- **Уникальные IP — только публичные.** В счётчик `IP` попадают лишь внешние адреса.
  Loopback, приватные (`10.x`, `172.16–31.x`, `192.168.x`) и собственные IP сервера
  отфильтрованы как контрагент, поэтому локальные healthcheck'и и обращения с
  `127.0.0.1` больше не накручивают уникальные IP.

- **`NAME` для `TIME-WAIT` / `FIN-WAIT`.** У таких сокетов процесс уже отсутствует,
  поэтому имя подтягивается по локальному сокету `ip:port` от живого процесса на
  том же порту.
