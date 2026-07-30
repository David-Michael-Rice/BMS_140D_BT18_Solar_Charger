D140 / BT18 Solar Charge Controller — Bluetooth Protocol
Reverse-engineered notes for the generic D140 MPPT solar charge controller
and its "BT18" Bluetooth module — the one the SController / Scontroller 2
app (Sunthysis; hardware attributed to Wuhan Hanfei Science & Technology) talks
to. Published so nobody else has to work it out from scratch.
Everything here was observed by capturing the app's own Bluetooth traffic. It is
a description of an interface, written from scratch. No vendor code is included.
> \\\\\\\*\\\\\\\*The one sentence that saves you days:\\\\\\\*\\\\\\\* the D140 speaks \\\\\\\*\\\\\\\*Modbus TCP\\\\\\\*\\\\\\\*
> (an MBAP header, \\\\\\\*\\\\\\\*no CRC\\\\\\\*\\\\\\\*) tunneled over a BLE characteristic — \\\\\\\*not\\\\\\\*
> Modbus RTU. RTU frames (with a CRC) get no reply at all.
Bluetooth transport
	
Advertised name	`BT18`
Data service	`0000ffe0-0000-1000-8000-00805f9b34fb`
Data characteristic	`0000ffe1-...` — used for both writing requests and receiving notifications
Unused	`0000ffe2-...` (present, not used by the app)
Manufacturer ID (adverts)	`0x050E`
Connections	one at a time (typical HM-10-style bridge)
Connect, enable notifications on `ffe1`, write a request to `ffe1`, and the
reply arrives as a notification on `ffe1`.
Framing — Modbus TCP (no CRC)
Every request and reply carries a 7-byte MBAP header followed by the PDU:
```
\\\\\\\[transaction id : 2]\\\\\\\[protocol id : 2 = 0x0000]\\\\\\\[length : 2]\\\\\\\[unit id : 1]\\\\\\\[PDU...]
```
`transaction id` increments per request; the reply echoes it.
`protocol id` is always `0x0000`.
`length` = number of bytes that follow (unit id + PDU).
`unit id` = `0x01`.
No CRC / checksum.
Reading (function 0x03, read holding registers)
The app reads one register at a time. Request is 12 bytes:
```
00 01  00 00  00 06  01  03  <reg hi> <reg lo>  00 01
└TID┘ └PID┘ └LEN┘ UNIT FN  └─ register ─┘   └qty=1┘
```
Reply:
```
00 01  00 00  00 05  01  03  02  <val hi> <val lo>
└TID┘ └PID┘ └LEN┘ UNIT FN  BC  └── value (1 reg) ──┘
```
Worked example — read battery voltage (register `0x0005`):
```
→ 00 01 00 00 00 06 01 03 00 05 00 01
← 00 01 00 00 00 05 01 03 02 05 35        0x0535 = 1333 → 13.33 V (×0.01)
```
Register map (function 0x03 holding registers)
The app polls these 15 registers each cycle, in this order:
`0x000B 0x0001 0x0002 0x0004 0x0006 0x0005 0x0007 0x000A 0x0027 0x0008 0x0036 0x0010 0x000D 0x0014 0x0011`.
Register	Meaning	Scale / encoding	Confidence
`0x0001`	PV (solar) voltage	×0.01 V	confirmed
`0x0002`	PV current	×0.01 A	confirmed (display 0.8–1.1 A)
`0x0004`	status flag	`1` observed	unknown
`0x0005`	Battery voltage	×0.01 V	confirmed
`0x0006`	Temperature	°C (likely)	likely
`0x0007`	PV current (2nd)	×0.01 A	mirrors `0x0002`
`0x0008`	Battery current	×0.01 A	confirmed
`0x000A`	State of charge	%	confirmed
`0x000B`	Charge state / mode	`0` = idle, `2`/`3`/`4` = charge stages	partial
`0x0036`	Load output enable (commanded)	`0` = off, `8` = on	confirmed
`0x000D`, `0x0010`, `0x0011`, `0x0014`, `0x0027`	setpoints echoed on the main screen (`0` / `247` / `0` / `85` / `0`)	—	part of the config block — see below
Notes:
`0x0008` is signed battery current. It reads charge current when the
panel is feeding in and discharge (load) current when it isn't. With the PV
panel unplugged and a ~0.94 A load, it read `94` (0.94 A ×0.01).
`0x0002` / `0x0007` are PV current (×0.01 A). Both drop to a flat
baseline the instant the PV panel is disconnected, which is how they were
identified, and read 0.8–1.1 A against the controller's own display.
Load output control (write, function 0x10)
The D140 has a switchable load output (a device you turn on/off remotely). The
app controls it by writing register `0x0036` with function `0x10`
(write single register):
```
Load ON  : 00 01 00 00 00 09 01 10 00 36 00 01 02 00 08
Load OFF : 00 01 00 00 00 09 01 10 00 36 00 01 02 00 00
                                                    └ 0x0008 = on / 0x0000 = off
```
Reading `0x0036` back reports the commanded state (`8` = on, `0` = off). Note
it reflects the switch position, which can also be changed by the physical
button on the controller — it is not the same signal as the actual current in
`0x0008`, so don't infer load draw from it directly.
Configuration registers (battery type & charge parameters)
Beyond the live-monitoring registers above, the controller stores a
configuration block: a battery type selector and, per type, a set of
default charge parameters. Known parameters (from the app's settings screen):
Battery type — an enum: Flooded, LiFePO4, Lithium, and others.
Float voltage
Absorption time
Overcharge alarm voltage
Charge shut-down (cut-off) voltage
Charge restart voltage (the level, after a shut-down, at which charging resumes)
Each battery type carries its own default values for these. Selecting
LiFePO4 locks the parameters — they become read-only, because the battery's
own BMS manages charge cut-off and balancing, so the controller defers to it.
These live at register addresses not in the 15-register monitoring poll —
they are read when the app opens its settings/parameters screen. To map them,
capture the BLE traffic while opening that screen (the app reads every config
register to populate it) and note the displayed value beside each, exactly as
the live registers were calibrated. Not yet captured here.
Still open
Exact stage names behind `0x000B` = 2 / 3 / 4 (bulk / absorption / float?).
The specific register address and scale for each configuration parameter
(capture the settings screen to fill these in).
Whether the static registers (`0x0010 = 247`, `0x0014 = 85`, …) are config
setpoints or firmware/model identifiers.
Why PV current appears in two registers (`0x0002` and `0x0007`).


use and build on this. This document describes an observed wire protocol; write
your own implementation rather than redistributing the vendor app or any
decompiled parts.
