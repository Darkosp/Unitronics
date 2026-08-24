# UniLogic апликација — Modbus TCP **Master**

## Идентификација (потврдено од проектот, 2026-08-24)

| Ставка                     | Вредност |
|----------------------------|----------|
| PLC модел                  | **USC-B10-TA30** (UniStream Compact) |
| UniLogic верзија (PC)      | **1.32.98** — проект зачуван со понова верзија нема да се отвори со постара |
| IP — CPU Ethernet          | 10.30.202.20 (keepalive 7200/75/10) |
| IP — Panel Ethernet        | 10.30.202.1  (keepalive 0/0/0 — исклучен) |
| Subnet / Gateway           | 255.255.255.0 / 10.30.202.100 |
| Проектен фајл              | `Proekt_Bazen_PPZ_Temp_Dno_Baz_1_32_98_ 24_08_2026.ulpr` |
| Обем                       | 28 ladder функции, 23 HMI екрани |

## Улога во системот

Овој контролер е **иницијаторот** на Modbus TCP комуникацијата — Master ID кон
Vision-от (SlaveID 255) на `10.30.202.15:502`, преку **Panel Ethernet** портот.
Логиката за поврзување живее тука.

## Читливи извадоци (version-controlled)

Проектот `.ulpr` е бинарен, но е декодиран и извезен во читлива форма:

- [`exports/network-config.md`](exports/network-config.md) — мрежа, портови, Modbus master/slave
- [`exports/modbus-register-map.md`](exports/modbus-register-map.md) — сите 40 Modbus трансакции

> Како е декодирано: `.ulpr` = 8-бајтен header + мал SOAP XML + вграден ZIP што
> содржи SQL Server backup (`.bak`) на целиот проект (325 табели). Backup-от се
> враќа во LocalDB и се чита со SQL. Постапката е повторлива — види
> `docs/architecture.md`.

## Отворен проблем — веќе има конкретни наоди

Modbus линкот со Vision паѓа без најава; се враќа само по повторен upload.
**Три конкретни причини најдени во конфигурацијата** (двоен NIC на иста
подмрежа, исклучен TCP keepalive, пол-ирање на 100 ms):
[`docs/issues/001-modbus-tcp-link-drop.md`](../docs/issues/001-modbus-tcp-link-drop.md)
