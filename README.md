# portstat

Мониторинг подключений и трафика по портам в реальном времени для VPN/прокси-серверов
(WireGuard, Hysteria 2, MTProto, Shadowsocks и т.п.). Считает по каждому сетевому порту:
число подключений, число уникальных IP, накопленный трафик и текущую скорость — отдельно
для входящего и исходящего, плюс общие итоги.

Работает поверх **conntrack**, поэтому видит и TCP, и UDP-сервисы (обычный `ss` UDP-клиентов
не показывает).

```
=== ВХОДЯЩИЕ ===
PORT    CONN / IP    TRAFFIC   SPEED
------------------------------------------
11478   42 / 18      8.3G      12.4MiB/s
8443    59 / 21      20.6M     310.0KiB/s
------------------------------------------
ИТОГО   101 / 33     8.4G      12.7MiB/s

=== ВСЕГО ===
Уникальных IP: 54
Трафик:        8.42G
Скорость:      13.6MiB/s
```

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

Скрипт нужно запускать **от root** (`conntrack -L` требует прав).

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
(замените `<USER>/<REPO>` на свой; либо просто скопируйте файл `portstat` в `/usr/local/bin/`)

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

## Команды одним блоком (Debian/Ubuntu, под root)

```bash
apt update && apt install -y conntrack iproute2 procps
modprobe nf_conntrack
echo nf_conntrack > /etc/modules-load.d/conntrack.conf
echo 'net.netfilter.nf_conntrack_acct=1' > /etc/sysctl.d/99-conntrack-acct.conf
sysctl --system
chmod +x /usr/local/bin/portstat        # после того как положили файл
echo "alias portstat='watch -n 2 /usr/local/bin/portstat'" >> ~/.bashrc && source ~/.bashrc
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
