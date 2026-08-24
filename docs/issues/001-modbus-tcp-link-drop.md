# 001 — Modbus TCP врската паѓа без најава

**Статус:** ПРИЧИНА ПОТВРДЕНА · **Заведено:** 2026-08-24
**Метод:** UniLogic `.ulpr` декодиран од внатрешната SQL база + ladder и
конфигурациски screenshot-и од двете апликации.

## Симптоми (пријавени)

1. Modbus TCP врската UniStream (Master) ↔ Vision (Slave) прекинува без најава.
2. Физички рестарт на контролерите **не** ја враќа.
3. **Само** повторен download на UniLogic апликацијата ја враќа.

## Потврден доказ (screenshot-и, 2026-08-24)

### Master (UniLogic) — `30_AQUIZACIJA_TCP_MODBUS_MASTER`
MODBUS Master FB, стартуван на rising edge (P) од `Mdb_TCP_start`, излез
`Status_MDB`. Вредности: `0`=No Error, `1`=Function start, `2`=Function in
progress, `-1`=Modbus error, ...

**Набљудувано при пад:** `Status_MDB` оди `0 → 2` и **останува 2 засекогаш**;
се враќа на `0` само по повторен download на целата апликација.

### Master — конфигурација на слејвот `Modbus_TCP_V_130`
- Com: `10.30.202.15 : 502`, Slave ID `255`, Response Timeout `1000` ms,
  Optimal queue `15`.
- **Сите операции се Aperiodic:** Registers Aperiodic (21) + Coils Aperiodic (9)
  = 30. Periodic табовите се 0. → трансакциите се **тригерирани од ladder**, не
  автоматски полирани.
- Постои **дијагностичка структура по слејв:** `Status, Sessions, Success, Fail,
  IP, Dropped, Remote Slave ID`.

### Slave (VisiLogic V130)
- Socket init + MODBUS IP Config само на `SB 2` (power-up); „Link lost"
  активиран (rung 3) → Vision се враќа во listen и чека нова врска. **Vision е
  добро конфигуриран.**

## ПРИЧИНА (потврдена)

**MODBUS Master FB виси на Status 2 засекогаш, бидејќи TCP keepalive е исклучен
на Panel Ethernet — мртвиот socket никогаш не се прогласува за неуспешен.**

1. FB праќа барање преку TCP (Panel Ethernet, `10.30.202.1`).
2. Линкот паѓа; барањето седи во TCP буфер, чека ACK што не доаѓа.
3. **Keepalive 0/0/0 на Panel Ethernet** → OS никогаш не го прогласува socket-от
   мртов → Response Timeout (1000ms) не се активира бидејќи FB виси во фазата на
   праќање, не на чекање одговор.
4. FB останува на `2`, зафатен → блокира сите нови трансакции.
5. Само download брише сè → `0`.

## Наоди

- **F2 (примарна):** Panel Ethernet keepalive `0/0/0` (исклучен). CPU Ethernet
  `7200/75/10`. Modbus master оди преку Panel Ethernet. **Keepalive НЕ Е
  достапен во UniLogic GUI** (потврдено 2026-08-24) — не може да се смени, па
  поправката мора да е во ladder.
- **F1 (потврдено визуелно):** Panel `10.30.202.1/24` и CPU `10.30.202.20/24` —
  двата на иста подмрежа `10.30.202.0/24`. ARP/routing двосмисленост.
- ~~F3 полирање 100ms~~ **ПОВЛЕЧЕНО:** операциите се Aperiodic (ladder-тригерирани),
  не автоматски полирани. Не е причина.
- **F4 (хигиена):** мртов „Remote Slave1" на `0.0.0.0` — да се исчисти.

## Поправка — ladder watchdog (единствен изводлив пат)

TCP Keepalive не постои во UniLogic GUI, па socket-от не може да се натера да
падне на ниво на OS. Затоа опоравувањето мора да го направи **ladder-от** на
UniStream. Имаме сè што треба: операциите се веќе ladder-тригерирани (aperiodic)
и постои дијагностичка структура по слејв (`Status, Sessions, Success, Fail,
Dropped`).

### Чекор 1 — Детекција (сигурна, веќе можна)
- TON `T_MdbStuck`, PT ≈ 5 s, IN = (`Status_MDB` == 2). Кога != 2 → ресет.
- Алтернативно/дополнително: следи пораст на `Modbus_TCP_V_130.Dropped`/`.Fail`
  или падот на `Modbus_TCP_V_130.Status`.

### Чекор 2 — Опоравување (механизмот треба да се потврди)
На `T_MdbStuck.Q`, треба да се изврши едно од следниве — **кое е достапно се
гледа од MODBUS палетата/рутините во проектот**:
- Ако постои MODBUS **Disconnect/Connect** (или socket close/open) за таа врска —
  повикај го, па ресетирај го `Mdb_TCP_start`.
- Ако нема explicit disconnect — превентивна стратегија: **PING gating**. Пред
  секое `Mdb_TCP_start`, провери со PING FB дека `10.30.202.15` одговара; ако не,
  не праќај трансакција (за да FB не се заглави воопшто), и продолжи штом ping
  се врати.

> За да ги дефинирам ТОЧНИТЕ блокови ми требаат screenshot-и (види „Сè уште
> отворено"). Без нив не можам да измислувам FB што можеби не постои.

### Чекор 3 — хигиена и дијагностика
- F4: избриши го „Remote Slave1" (0.0.0.0).
- F1: ако двата порта се физички на иста мрежа, раздели подмрежи.
- HMI екран со `Status, Success, Fail, Dropped, Sessions`.

## Проверка (loop со докази)
По секоја фаза: Save → испрати `.ulpr` → пре-декодирам и потврдувам дека
промената влегла → Download → тест: извади кабел 5–10 s, врати; врската да се
врати сама за < 30 s без download.

## Сè уште отворено
- [x] TCP Keepalive НЕ постои во UniLogic GUI → ladder е единствениот пат.
- [ ] Connect/reconnect механизам за Фаза 2 (види `MODBUS_RTU_TCP_Read_Write` /
      како се сетира `Mdb_TCP_start`).
- [ ] Точниот SB број за „link lost" на Vision (потврда).
