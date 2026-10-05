# Prometheus-стек

## 2.1 node_exporter (обе ВМ)

Ставится одинаково на CauseYou и StepSon. Архитектура ВМ: `x86_64` (проверка: `uname -m`).

### Установка

```bash
# последняя версия с GitHub
VER=$(curl -s https://api.github.com/repos/prometheus/node_exporter/releases/latest | grep -oP '"tag_name": "v\K[^"]+')
echo $VER

cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v$VER/node_exporter-$VER.linux-amd64.tar.gz
tar xzf node_exporter-$VER.linux-amd64.tar.gz
sudo cp node_exporter-$VER.linux-amd64/node_exporter /usr/local/bin/
sudo chown root:root /usr/local/bin/node_exporter

# системный пользователь без домашнего каталога и шелла
sudo useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter
```

### systemd-юнит

Файл `node_exporter/node_exporter.service` из репозитория копируется в `/etc/systemd/system/`:

```bash
# CauseYou (из репозитория)
sudo cp ~/Prometheus_Zabbix/node_exporter/node_exporter.service /etc/systemd/system/

# StepSon (забираем с CauseYou)
scp -P 22 <user>@10.10.10.11:Prometheus_Zabbix/node_exporter/node_exporter.service /tmp/
sudo cp /tmp/node_exporter.service /etc/systemd/system/

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

### Проверка

```bash
systemctl is-active node_exporter          # active
ss -tlnp | grep 9100                        # слушает *:9100
curl -s localhost:9100/metrics | grep ^node_load1
```

Связность между ВМ:

```bash
# с CauseYou
curl -s 10.10.10.12:9100/metrics | grep ^node_load1
# со StepSon
curl -s 10.10.10.11:9100/metrics | grep ^node_load1
```

Результат: версия node_exporter `<VER>`, сервис active на обеих ВМ, метрики доступны по сети `10.10.10.0/24`.

## 2.2 Prometheus (CauseYou)

### Установка

```bash
VER=$(curl -s https://api.github.com/repos/prometheus/prometheus/releases/latest | grep -oP '"tag_name": "v\K[^"]+')
echo $VER

cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v$VER/prometheus-$VER.linux-amd64.tar.gz
tar xzf prometheus-$VER.linux-amd64.tar.gz
sudo cp prometheus-$VER.linux-amd64/{prometheus,promtool} /usr/local/bin/

sudo useradd --system --no-create-home --shell /usr/sbin/nologin prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

### Конфиг и юнит

В `prometheus.yml` каждой цели задана метка `instance_name` (CauseYou / StepSon): её используют правила алертов и запросы `avg by(instance_name)`.

```bash
cd ~/Prometheus_Zabbix
sudo cp prometheus/prometheus.yml prometheus/alert.rules.yml /etc/prometheus/
sudo chown -R root:prometheus /etc/prometheus
promtool check config /etc/prometheus/prometheus.yml

sudo cp prometheus/prometheus.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
systemctl is-active prometheus
```

### 2.3 Проверка

В браузере на Windows: `http://localhost:9090/targets`. Цели `node` (2 шт.) и `prometheus` в состоянии **UP**. Цель `alertmanager` будет DOWN до шага 2.5.

Первые запросы (вкладка Query):

```
up
node_load1
100 - avg by(instance_name)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100
```

## 2.4 Правила алертов

Файл `prometheus/alert.rules.yml`, правила из таблицы в плане. Проверка и применение:

```bash
promtool check rules /etc/prometheus/alert.rules.yml
sudo systemctl reload prometheus
```

В браузере: `http://localhost:9090/alerts`, все 5 правил в состоянии Inactive.

## 2.5 Alertmanager с Telegram (CauseYou)

Telegram API с ВМ напрямую недоступен, Alertmanager ходит через SOCKS-прокси Happ (`socks5://10.0.2.2:10808`, см. `docs/01-stand.md`). Happ на Windows должен быть запущен с включённым Allow LAN.

### Установка

```bash
VER=$(curl -s https://api.github.com/repos/prometheus/alertmanager/releases/latest | grep -oP '"tag_name": "v\K[^"]+')
echo $VER

cd /tmp
curl -LO https://github.com/prometheus/alertmanager/releases/download/v$VER/alertmanager-$VER.linux-amd64.tar.gz
tar xzf alertmanager-$VER.linux-amd64.tar.gz
sudo cp alertmanager-$VER.linux-amd64/{alertmanager,amtool} /usr/local/bin/

sudo useradd --system --no-create-home --shell /usr/sbin/nologin alertmanager
sudo mkdir -p /var/lib/alertmanager
sudo chown alertmanager:alertmanager /var/lib/alertmanager
```

### Токен, конфиг и юнит

Токен уже лежит в `/etc/alertmanager/telegram_token` (права 600). Делаем владельцем пользователя alertmanager:

```bash
sudo chown alertmanager:alertmanager /etc/alertmanager/telegram_token

cd ~/Prometheus_Zabbix
sudo cp alertmanager/alertmanager.yml /etc/alertmanager/
sudo amtool check-config /etc/alertmanager/alertmanager.yml

sudo cp alertmanager/alertmanager.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now alertmanager
systemctl is-active alertmanager
```

### Проверка

- `http://localhost:9093`: веб-интерфейс Alertmanager открывается.
- `http://localhost:9090/targets`: цель `alertmanager` теперь UP.
- Тестовый алерт напрямую в Alertmanager (должен прийти в Telegram примерно через 30 секунд):

```bash
amtool alert add TestAlert instance_name=CauseYou severity=info \
  --annotation=summary="Тест Alertmanager" \
  --alertmanager.url=http://localhost:9093
```

Если сообщение не пришло: `journalctl -u alertmanager -n 30`.
