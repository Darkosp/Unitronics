# 001 — Modbus TCP врската паѓа без најава

**Статус:** ПРИЧИНА ПОТВРДЕНА · **Заведено:** 2026-08-24
**Метод:** UniLogic `.ulpr` декодиран и прочитан од внатрешната SQL база + ladder
screenshot-и од двете апликации (UniLogic master и VisiLogic slave).

## Симптоми (пријавени)

1. Modbus TCP врската UniStream (Master) ↔ Vision (Slave) прекинува без најава.
2. Физички рестарт на контролерите **не** ја враќа.
3. **Само** повторен download на UniLogic апликацијата ја враќа.

## Потврден доказ (screenshot-и, 2026-08-24)

### Master (UniLogic) — рутина `30_AQUIZACIJA_TCP_MODBUS_MASTER`

MODBUS Master функциски блок, стартуван на rising edge (P) од `Mdb_TCP_start`,
излез во `Status_MDB`. Вредности на Status:

```
 0 = No Error         1 = Function start (UniCom sent)   2 = Function in progress
-1 = Modbus error    -2 = Init error   -4 = Invalid group ID   ...
```

**Набљудувано при пад на линкот:** `Status_MDB` оди `0 → 2` и **останува 2**
(Function in progress) сè додека линкот е изгубен. Се враќа на `0` **само** по
повторен download на целата апликација.

### Slave (VisiLogic V130) — Main Routine

- Rung 1–2 (на `SB 2` power-up): Close Socket 2 → Card Init → Sock Init 1/2/3 →
  **MODBUS IP CONFIG на Socket 2**, TCP 502, Network ID 255, Timeout D#100 (1s),
  Retries D#3.
- Rung 3 (на `SB 2`): SET на бит за **активирање „Link lost"** (auto-detekcija
  и ослободување на socket-от при пад).
- Rung 4 (без услов): `MODBUS IP SCAN_EX` — служи барања постојано.

## ПРИЧИНА (потврдена)

**MODBUS Master блокот на UniLogic виси во недовршена трансакција (Status 2)
засекогаш, бидејќи TCP keepalive е исклучен на портот преку кој оди.**

Механика, чекор по чекор:
1. Блокот праќа Modbus барање преку TCP socket (Panel Ethernet, `10.30.202.1`).
2. Линкот паѓа. Барањето седи во TCP буфер и чека ACK што не доаѓа.
3. **Keepalive = 0/0/0 на Panel Ethernet** → OS TCP стекот никогаш не прогласува
   дека socket-от е мртов → нема грешка, нема timeout на ниво на врска.
4. Затоа FB вечно останува на `2` — ни одговор, ни грешка (`-1`).
5. Заглавен FB на `2` е зафатен → не прифаќа нова трансакција → целиот master
   е блокиран засекогаш.
6. Само download брише сè (FB состојба + сите сокети) и прави свеж старт → `0`.

Vision страната **е добро конфигурирана** (link-lost активиран → се враќа во
listen и чека нова врска). Проблемот е целосно на master страната.

### Потпорни фактори од конфигурацијата (од `.ulpr`)

- **F2 (примарна):** `CommunicationsConfiguration` → Panel Ethernet
  `TCPKeepaliveTime/Interval/Retry = 0/0/0` (исклучен). CPU Ethernet го има
  вклучен (7200/75/10). Modbus master оди преку Panel Ethernet.
- **F3 (окидач):** `MODBUSSlaveAddressType` → 30 трансакции кон Vision, сите на
  `EveryPeriod = 100 ms` (~300/сек) кон еден V130 — го предизвикува падот.
- **F1 (ризик):** двата UniStream порта (`.20` и `.1`) на иста подмрежа
  `10.30.202.0/24` (ARP/routing двосмисленост).
- **F4 (хигиена):** мртов „Remote Slave1" на `0.0.0.0` — да се исчисти.

## Поправка — фазно (сите измени се само во UniLogic; Vision не се дира)

### Фаза 1 — Keepalive (само параметар, БЕЗ ladder) ⭐ прво

Вклучи TCP Keepalive на **Panel Ethernet**. Тогаш мртвиот socket ќе се затвори
за ~15–20s, FB ќе врати `-1` наместо да виси на `2`, и нормалната trigger
логика повторно ќе ја активира трансакцијата на свеж socket. **Ова само по себе
може да го реши проблемот.**

Предложени вредности: Time = 20 s, Interval = 5 s, Retry = 3.

### Фаза 2 — Растоварување (само параметри, БЕЗ ladder)

- F3: смени `EveryPeriod` од 100 ms на 500 ms–1 s за Modbus scan-от кон Vision.
- F4: избриши го „Remote Slave1" (0.0.0.0).

### Фаза 3 — Watchdog во ladder (само ако Фаза 1–2 не е доволна)

Осигурување — дури и ако FB виси:
- TON тајмер `T_MdbStuck`, PT ≈ 5 s, IN = (`Status_MDB` == 2). Кога Status != 2,
  тајмерот се ресетира.
- На `T_MdbStuck.Q`:
  1. Изврши Modbus reconnect (Disconnect → Connect) на таа TCP врска, и
  2. ресетирај го trigger-от (`Mdb_TCP_start`) за нова трансакција.

Точниот reconnect механизам зависи од достапните MODBUS FB-ови во проектот — да
се дефинира заедно откако ќе се види connect логиката.

## Проверка (loop со докази)

По секоја фаза: Save во UniLogic → испрати го новиот `.ulpr` → се пре-декодира и
се потврдува дека промената навистина влегла (keepalive != 0, EveryPeriod
сменет) → Download → тест на терен со намерен прекин на линкот (извади кабел
5–10 s, врати го) → врската треба да се врати **сама** за < 30 s без download.

## Сè уште отворено

- [ ] Потврди го точниот SB број за „link lost" на Vision (hover во rung 3).
- [ ] Провери ја UniLogic рутината `MODBUS_RTU_TCP_Read_Write` / connect логиката
      за Фаза 3.
- [ ] F1: физичка топологија — колку кабли, ист switch?
