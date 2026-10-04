# PUSR G771 as an NTRIP Client for RTK Corrections

Configuration notes for using a **PUSR G771 / G771-E** serial-to-cellular modem as an
NTRIP client, forwarding RTCM correction data to a serial-connected GNSS receiver
(tested against a CLAAS RTK monitor and the free [Centipede RTK](https://centipede.fr/)
caster network in France).

> ⚠️ The G771 has **no built-in NTRIP client mode**. This setup works around that by
> using **Transparent (NET) mode** plus the **Identity Package** feature to send a
> manually-crafted HTTP GET request once, when the TCP socket connects.

---

## Contents

- [How it works](#how-it-works)
- [Prerequisites](#prerequisites)
- [1. Serial port settings](#1-serial-port-settings)
- [2. Socket A (TCP client) settings](#2-socket-a-tcp-client-settings)
- [3. Identity Package (NTRIP login)](#3-identity-package-ntrip-login)
- [4. Save and restart](#4-save-and-restart)
- [Building the GET request hex string](#building-the-get-request-hex-string)
- [Optional: NEAR mountpoint + periodic GGA (Heartbeat Package)](#optional-near-mountpoint--periodic-gga-heartbeat-package)
- [Troubleshooting](#troubleshooting)
- [Known limitation: OEM lock-in](#known-limitation-oem-lock-in)

---

## How it works

An NTRIP connection is just a plain HTTP GET request over a TCP socket. The caster
replies with a short HTTP header, then streams raw RTCM binary data for as long as the
socket stays open.

Since the G771 doesn't understand NTRIP natively, the trick is:

1. Put the modem in **Transparent mode**, and open a **persistent TCP socket** to the
   NTRIP caster.
2. Use the **Identity Package** feature (normally meant for device-registration
   strings) to send a hex-encoded HTTP GET request **once**, right when the socket
   connects.
3. Everything the caster sends back afterward — the HTTP response header, then the
   RTCM stream — gets passed straight through to the serial port, since Transparent
   mode doesn't touch the data.

---

## Prerequisites

- PUSR G771 / G771-E, connected via its **RS232/RS485 port** (not USB — USB talks to
  the cellular module's diagnostic AT port and will return `+CME ERROR` for these
  commands).
- NTRIP caster address, port, mountpoint, and credentials from your correction
  provider.
- A terminal / the PUSR configuration utility to send AT commands.

---

## 1. Serial port settings

Match the G771's UART settings to your GNSS receiver's correction-input port (baud
rate, data/stop bits, parity). This varies by receiver — check its documentation.

---

## 2. Socket A (TCP client) settings

```
AT+SOCKAEN=ON                              # enable Socket A
AT+SOCKASL=LONG                            # persistent connection (not open-on-data)
AT+SOCKA=TCP,<caster_address>,<port>       # e.g. TCP,caster.centipede.fr,2101
```

---

## 3. Identity Package (NTRIP login)

```
AT+REGEN=ON                                # enable Identity Package
AT+REGTP=USER                              # type = user-defined data
AT+REGDT=<hex-encoded GET request>         # see below
AT+REGSND=LINK                             # send once, only at connection
```

---

## 4. Save and restart

```
AT+S
```

---

## Building the GET request hex string

The `AT+REGDT` value must be the hex encoding of a complete, correctly terminated
HTTP GET request. Every line must end in `\r\n` (bytes `0D 0A`), and the request
needs a **blank line at the end** (`\r\n\r\n`) or the caster will reject it with
`400 Bad Request`.

Template:

```
GET /MOUNTPOINT HTTP/1.1
Host: <caster_address>:<port>
Ntrip-Version: Ntrip/2.0
User-Agent: NTRIP PUSR-G771
Authorization: Basic <base64(user:pass)>

```

Python snippet to generate it:

```python
import base64

mountpoint = "KOLR"
host = "caster.centipede.fr"
port = 2101
user = "centipede"
password = "centipede"

auth = base64.b64encode(f"{user}:{password}".encode()).decode()

request = (
    f"GET /{mountpoint} HTTP/1.1\r\n"
    f"Host: {host}:{port}\r\n"
    f"Ntrip-Version: Ntrip/2.0\r\n"
    f"User-Agent: NTRIP PUSR-G771\r\n"
    f"Authorization: Basic {auth}\r\n"
    f"\r\n"
)

print(request.encode().hex().upper())
```

Paste the resulting hex string into `AT+REGDT=` (or the "User-defined data" field in
the config utility, with **Hex** checked).

### Example (Centipede RTK, mountpoint `KOLR`)

```
Host:     caster.centipede.fr
Port:     2101
Mount:    KOLR
User/Pass: centipede / centipede
```

```
474554202F4B4F4C5220485454502F312E310D0A486F73743A206361737465722E63656E7469706564652E66723A323130310D0A4E747269702D56657273696F6E3A204E747269702F322E300D0A557365722D4167656E743A204E5452495020505553522D473737310D0A417574686F72697A6174696F6E3A204261736963205932567564476C775A57526C4F6D4E6C626E52706347566B5A513D3D0D0A0D0A
```

---

## Optional: NEAR mountpoint + periodic GGA (Heartbeat Package)

Centipede's `NEAR` mountpoint auto-selects the closest base station, but it expects
an **NMEA GGA sentence** sent periodically so the caster knows your position and can
switch base stations as you move. The one-time Identity Package can't do this —
use the **Heartbeat Package** instead, which resends a fixed string on a timer:

```
AT+HEARTEN=ON              # enable heartbeat
AT+HEARTTP=USER            # user-defined data
AT+HEARTDT=<hex-encoded GGA sentence>
AT+HEARTTM=15              # send every 15s (tune as needed)
AT+S
```

⚠️ Not confirmed whether the Identity Package and Heartbeat Package can run at the
same time on this firmware without conflict — test carefully. If it doesn't behave,
use a **fixed mountpoint** (like `KOLR` above) instead, which needs no GGA at all.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `HTTP/1.1 400 Bad Request` from caster | Malformed GET request — missing `Host:`, missing `Ntrip-Version:`, missing trailing blank line, or wrong line endings (`0A` instead of `0D 0A`) in the hex. |
| Modem echoes back something unrelated (e.g. `AT+Z`) as if it were caster data | Wrong/stale value stored in `AT+REGDT`, or a literal AT command got sent while already in transparent/data mode. |
| `+CME ERROR:58` on every AT command | Connected to the wrong physical interface — likely the USB diagnostic port (talks to the cellular module) instead of the RS232/RS485 data port. |
| `+++` escape sequence fails | Must be sent as **two separate transmissions** (`+++`, wait for `a`, then send `a` again within 3s) — not as one combined `+++a` string. |
| Caster connects and streams data, but receiver does nothing | See [Known limitation](#known-limitation-oem-lock-in) below. |

---

## Known limitation: OEM lock-in

Some terminals (e.g. certain CLAAS RTK monitors) only reveal their RTK correction
menu when their **original OEM telematics module** (e.g. a CLAAS TCM II) is
connected — even if a generic modem is correctly streaming RTCM data on the same
serial line. This is because the OEM module may authenticate over CAN bus with a
proprietary handshake, rather than simply passing RTCM bytes over RS232.

If your receiver behaves this way, this NTRIP setup will get valid RTCM to the
serial port, but the terminal may still ignore it. Confirm with your equipment
dealer whether your specific terminal supports a non-OEM correction source before
relying on this setup.

---

## Disclaimer

This is a community workaround, not an officially documented feature of either the
PUSR G771 or any GNSS receiver/terminal. Test thoroughly before relying on it in
the field.
