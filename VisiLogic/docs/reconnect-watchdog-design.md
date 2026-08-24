# VisiLogic — watchdog за автоматско опоравување на Modbus TCP линкот

> Поправка **само на Vision страна** (slave). Master-от (UniLogic) не се дира.
> Сите потребни FB-ови постојат и се потврдени во палетата (2026-08-24):
> `Com → TCP/IP`: Socket Init, Connect: TCP, **Close: TCP**, **PING**.
> `FB's → MODBUS IP`: **Configuration**, ScanEX.

## Идеја

Master-от виси на `Status 2` со half-open socket и не се реконектира сам (нема
keepalive). Ако **Vision го затвори и повторно го отвори Socket 2**, тогаш при
следниот пакет од master-от Vision враќа TCP RST → master-от го добива грешниот
socket → неговиот FB излегува од `Status 2` → се реконектира на свежиот listener
на Vision. Врската се враќа **без re-upload и без keepalive**.

## Што сè уште треба да се потврди — тригерот

Треба **битот што покажува дека Socket 2 / линкот е паднат**. Vision веќе
„активира link lost" (Main rung 3, `SB 168`). Побарај го статусниот SB:
`Operands → SB` → барај „Socket 2 … Connected" / „Link" / „Socket Status".
Означи го тука: `SB ___` = Socket 2 disconnected/link-lost.

Ако не се најде сигурен статус-бит што фаќа и „меко" замрзнување (линк физички
горе, но сесијата мртва), користи ја **PING-варијантата** подолу.

## Дизајн A — тригер преку статус-бит (препорачано, ако постои)

Нека `MB_LinkLost` = статусниот SB (или негова копија).

**Рунг R1 — при пад, затвори:**
```
[ MB_LinkLost  (P) ] ── Close: TCP  (Socket 2)
                     ── (S) MB_ReinitPending
                     ── TON  T_Reinit  (PT = 1 s)
```

**Рунг R2 — по 1 s, повторно отвори и конфигурирај:**
```
[ MB_ReinitPending ][ T_Reinit.Q ] ── TCP/IP Socket Init (Socket 2)
                                    ── MODBUS IP Configuration
                                         (Socket 2, TCP 502, Network ID 255,
                                          Timeout D#100, Retries D#3)
                                    ── (R) MB_ReinitPending
                                    ── (R) MB_LinkLost
```

Тоа е истата низа како power-up rung 2 (само за Socket 2), но тригерирана на
пад наместо на `SB 2`.

## Дизајн B — PING-варијанта (ако нема сигурен статус-бит)

Vision пингува кон Panel Ethernet на master-от (`10.30.202.1`) и, кога линкот
ќе се врати по прекин, го рефрешира Socket 2.

**Рунг P1 — пинг на секои ~2 s:**
```
[ T_PingIval.Q (NC) ] ── TON  T_PingIval (PT = 2 s)
```

**Рунг P2 — изврши PING:**
```
[ T_PingIval.Q (P) ] ── PING  (IP 10.30.202.1)  → PING_Success / PING_Fail
```

**Рунг P3 — заклучи дека паднал:**
```
[ PING_Fail ] ── (S) MB_LinkWasDown
```

**Рунг P4 — при враќање, рефрешрирај го Socket 2:**
```
[ PING_Success ][ MB_LinkWasDown ] ── Close: TCP (Socket 2)
                                    ── (S) MB_ReinitPending
                                    ── TON T_Reinit (PT = 1 s)
                                    ── (R) MB_LinkWasDown
```
(потоа истиот R2 рунг за Socket Init + MODBUS IP Configuration)

> Ограничување: PING-варијантата фаќа **физички** прекин. Ако линкот е физички
> горе но сесијата е мртва („меко" замрзнување), користи го Дизајн A со статус-бит,
> или додај Modbus **heartbeat** (бара мала измена и на master-от).

## Операнди што треба да ги одбереш (слободни во проектот)
- `MB_LinkLost` (или статус SB), `MB_ReinitPending`, `MB_LinkWasDown`
- `T_Reinit` (TON 1 s), `T_PingIval` (TON 2 s)
- PING резултат битови (`PING_Success` / `PING_Fail`)

## Тест
Извади го кабелот кон Vision 5–10 s, врати го. Врската да се врати **сама** за
< 30 s; на master-от `Status_MDB` да оди `2 → -1 → 0`, не да виси на `2`.
