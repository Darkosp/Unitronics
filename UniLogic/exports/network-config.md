# UniLogic — мрежна и Modbus конфигурација (извадок од проектот)

> Автоматски извлечено од `Proekt_Bazen_PPZ_Temp_Dno_Baz_1_32_98_ 24_08_2026.ulpr`
> на 2026-08-24. Изворот е бинарен; овој фајл е читливиот, diff-абилен извадок.
> Ако проектот се измени, регенерирај го овој извадок во ист commit.

## PLC / софтвер

| | |
|---|---|
| Модел | **USC-B10-TA30** (UniStream Compact) |
| UniLogic верзија | **1.32.98** |
| Проект создаден | 2021-01-30 |
| Последно зачувано | 2026-08-24 (SavedBy: `SRVAITPPZVOD\alkppz`) |
| Ladder функции / HMI екрани | 28 / 23 |

## Ethernet портови (`CommunicationsConfiguration`)

UniStream-от има **два** Ethernet порта, и двата на **истиот** сегмент:

| Порт | EthernetType | IP | Subnet | Gateway | TCP Keepalive (Time/Interval/Retry) |
|------|-------------|----|--------|---------|-------------------------------------|
| **CPU Ethernet**   | 1 | `10.30.202.20` | 255.255.255.0 | 10.30.202.100 | **7200 / 75 / 10** (вклучен) |
| **Panel Ethernet** | 0 | `10.30.202.1`  | 255.255.255.0 | 10.30.202.100 | **0 / 0 / 0** (ИСКЛУЧЕН) |

⚠ Двата порта се во истата подмрежа `10.30.202.0/24`. Види наод F1 во
[001-modbus-tcp-link-drop.md](../../docs/issues/001-modbus-tcp-link-drop.md).

UDP Socket1 на порта 555 (`UDPSockets`) — не е поврзан со Modbus.

## Modbus Master-и (`MODBUSMasterConfiguration`)

| Име | Преку порт | Периодичен режим |
|-----|-----------|------------------|
| Panel Ethernet | Panel Ethernet (`10.30.202.1`, keepalive OFF) | Не |
| UAC-CB-01RS4_0 | сериски | Не |

## Modbus Slave-ови кон кои чита Master-от (`MODBUSRemoteSlaveConfiguration`)

| Име | IP:Port | SlaveID | ResponseTimeout | Master |
|-----|---------|---------|-----------------|--------|
| **Modbus_TCP_V_130** | **10.30.202.15:502** | 255 | **1000 ms** | Panel Ethernet |
| Remote Slave1 | 0.0.0.0:502 | 1 | 500 ms | UAC-CB-01RS4_0 (недоконфигуриран остаток) |

`Modbus_TCP_V_130` = **Vision V130 PLC-то** (VisiLogic Slave-от). Ова е линкот што паѓа.

## UniStream како локален Slave (`MODBUSLocalSlavesConfiguration` / `MODBUSSlaveConfiguration`)

| Име | IP:Port | Забелешка |
|-----|---------|-----------|
| Vodomeri | 10.30.202.1:502 | читање водомери, 6 трансакции |
