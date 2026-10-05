# Стенд

Две виртуальные машины в VirtualBox на ноутбуке с Windows.

| | CauseYou | StepSon |
|---|---|---|
| Роль | Prometheus, Alertmanager, node_exporter | Grafana, node_exporter |
| ОС | Ubuntu Server 25.10 | Ubuntu Desktop 26.04 LTS |
| Внутренняя сеть (`enp0s8`) | 10.10.10.11/24 | 10.10.10.12/24 |
| NAT (`enp0s3`) | 10.0.2.15/24 | 10.0.2.15/24 |
| RAM | 1.6 ГБ | 3.3 ГБ |
| Swap | нет | нет |
| Диск `/` | 25 ГБ | 25 ГБ |

Машины общаются между собой по сети `10.10.10.0/24`. В интернет (Telegram API) выходят через NAT VirtualBox.

## Проброс портов (VirtualBox, NAT)

Все правила привязаны к `127.0.0.1`, чтобы сервисы не были видны из локальной сети.

| Сервис | Адрес на Windows | ВМ | Хост → гость |
|---|---|---|---|
| SSH | `ssh -p 9999 <user>@localhost` | CauseYou | 9999 → 22 |
| Prometheus | `http://localhost:9090` | CauseYou | 9090 → 9090 |
| Alertmanager | `http://localhost:9093` | CauseYou | 9093 → 9093 |
| SSH | `ssh -p 2222 <user>@localhost` | StepSon | 2222 → 22 |
| RDP | `localhost:33890` | StepSon | 33890 → 3389 |
| Grafana | `http://localhost:3000` | StepSon | 3000 → 3000 |

## Проверки стенда

Дата проверки: 2026-10-05.

| Проверка | CauseYou | StepSon |
|---|---|---|
| `ping` соседней ВМ по `enp0s8` | 3/3, 0% потерь, ~2.3 мс | 3/3, 0% потерь, ~2.0 мс |
| `ufw status` | inactive | inactive |
| Часовой пояс | Etc/UTC | Etc/UTC |
| Синхронизация времени | была `no`, включена (`systemd-timesyncd`, `timedatectl set-ntp true`), стало `yes` | была `no`, включена, стало `yes` |
| Доступ к Telegram API напрямую (VPN выкл. и вкл.) | таймаут, `000` | — |
| Доступ к Telegram API, Happ в TUN-режиме | DNS не резолвится, `000` | — |
| Доступ к Telegram API через прокси Happ `socks5h://10.0.2.2:10808` | `302`, доступен | — |


