# Архитектура на системот

> Мрежните вредности се **потврдени** од UniLogic проектот (2026-08-24).
> Vision страната сè уште чека потврда (`.vlp` е заштитен со лозинка).

## Топологија

```
              Ethernet 10.30.202.0/24  ·  gateway .100
   ┌───────────────────────┴───────────────────────┐
   │                                                │
┌──┴──────────────────────┐               ┌─────────┴────────┐
│ UniStream USC-B10-TA30   │               │  Vision V130     │
│ UniLogic 1.32.98         │               │  VisiLogic       │
│ Modbus MASTER            │               │  Modbus SLAVE    │
│  CPU Eth   .20  (ka on)  │               │  SlaveID 255     │
│  Panel Eth .1   (ka off) │──502─────────▶│  .15             │
└──────────────────────────┘               └──────────────────┘
```

Стрелката = кој ја иницира врската. Modbus Master оди преку **Panel Ethernet**.

## Мрежни параметри (потврдени)

| Параметар        | CPU Ethernet | Panel Ethernet | Vision |
|------------------|--------------|----------------|--------|
| IP               | 10.30.202.20 | 10.30.202.1    | 10.30.202.15 |
| Subnet           | 255.255.255.0| 255.255.255.0  | _?_ |
| Gateway          | 10.30.202.100| 10.30.202.100  | _?_ |
| TCP Keepalive    | 7200/75/10   | **0/0/0 off**  | _?_ |
| Modbus порта     | —            | 502 (master)   | 502 (slave) |

⚠ **Двата UniStream порта се во истата подмрежа** — види F1 во
[`issues/001-modbus-tcp-link-drop.md`](issues/001-modbus-tcp-link-drop.md).

## Modbus регистарска мапа

Извезена од проектот →
[`UniLogic/exports/modbus-register-map.md`](../UniLogic/exports/modbus-register-map.md).
Ова е договорот меѓу двете апликации; секоја промена оди во двете, во ист commit.

## Како да се прочита `.ulpr` (повторливо)

```
.ulpr = [8B header][SOAP XML метадата][вграден ZIP]
вграден ZIP → NGP_*.bak  (SQL Server backup, „TAPE" формат)
```

1. Извади го ZIP-от од offset 1776 (или барај `PK\x03\x04`).
2. `RESTORE DATABASE ... FROM DISK='NGP_*.bak'` во `(localdb)\MSSQLLocalDB`.
3. Проектот е 325 SQL табели. Клучни: `CommunicationsConfiguration`,
   `MODBUSMasterConfiguration`, `MODBUSRemoteSlaveConfiguration`,
   `MODBUSSlaveAddressType`, `MODBUSOperands`.

`.vlp` = ZIP што содржи `Current_OPLC.udb` = MS Access (Jet) база, заштитена со
лозинка (стандардна VisiLogic лозинка — потребна за читање).

## Git LFS

Сè уште не е вклучен. `.ulpr` е ~6 MB, `.vlp` ~170 KB — во ред за сега. Ако
`app/` надмине ~20 MB, вклучи LFS (`git lfs track "*.ulpr" "*.vlp"`) **пред**
следниот голем commit.
