# Modbus регистарска мапа — UniLogic Master → Vision Slave

> Извадок од `MODBUSSlaveAddressType` во проектот, 2026-08-24. Читлив и diff-абилен.
> FC = Modbus function code. Сите трансакции се пол-ираат на **секои 100 ms**.

## Линк што паѓа: `Modbus_TCP_V_130` @ 10.30.202.15:502 (SlaveID 255)

**30 трансакции, сите на 100 ms** → ~300 барања/сек кон еден Vision V130.
Види наод F3 во [001-modbus-tcp-link-drop.md](../../docs/issues/001-modbus-tcp-link-drop.md).

### Читања — Holding Registers (FC03)

Адреси: 3, 15, 16, 17, 31, 32, 33, 39, 40, 47, 48, 55, 57, 113, 114, 116, 117, 121, 122

### Читања — Coils (FC01)

Адреси: 17, 18, 64

### Запишувања — Multiple Coils (FC15)

Адреси: 8, 9, 23, 24, 25, 26

### Запишувања — Multiple Registers (FC16)

Адреси: 4, 5

## Локален slave `Vodomeri` @ 10.30.202.1:502 — 6 читања FC03 (адреси 1–6)

## `Remote Slave1` @ 0.0.0.0 — 4 трансакции, IP не е поставен (мртов/остаток → да се исчисти)

---

## Да се потврди на Vision страната (VisiLogic)

Овие адреси мора да одговараат **точно** на Modbus мапата дефинирана во
VisiLogic (MODBUS Configuration / Scan во ladder-от). Vision користи 1-базирано
адресирање во некои прикази — провери offset. Кога ќе се отвори `.vlp`,
пополни ја колоната „Vision операнд" за секоја адреса горе.
