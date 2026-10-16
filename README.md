# Signed A/B Firmware Update with Trial Boot and Rollback (ESP32)

An ESP32 sketch (Arduino IDE) and a Python host tool that update the firmware over USB serial. The device writes the new image to the inactive OTA slot, accepts it only if its ECDSA P-256 signature and SHA-256 hash check out and its version is newer, boots it on trial, runs a health check, and rolls back to the previous image if the check fails. The rollback count is stored in flash.

## How it works

```text
 PC (fota_tool.py)                           ESP32
 ─────────────────                           ──────────────────────────────────────
 sign(version | size | SHA-256)  ─ header ─▶ 1. version newer than running image?
 with the private key                        2. ECDSA signature valid? (public key in firmware)
                                             3. erase inactive slot
                                 ◀── 'K' ──
 1 KB chunks                     ─ data ───▶ 4. write to flash + running SHA-256
                                 ◀── 'A' ──     (one ACK per chunk)
                                             5. SHA-256 matches the signed hash?
                                             6. validate image, set boot slot, reboot
                                 ◀── 'D' ──
                                             7. TRIAL BOOT: health check
                                                pass -> confirmed
                                                fail / crash loop -> switch back to old slot
```

| Field in the update header | Size |
|---|---|
| Magic `FOTA` | 4 B |
| Version (little-endian) | 4 B |
| Image size (little-endian) | 4 B |
| SHA-256 of the image | 32 B |
| Signature length | 2 B |
| Signature (DER, ECDSA P-256) | up to 80 B |

The signature covers `version | size | SHA-256`, so the version and size cannot be changed without invalidating it.

**Trial boot and rollback.** Before rebooting, the old image stores the candidate's version in non-volatile storage (NVS). The candidate sees that its own version is "on trial", counts the attempt, and runs the health check (free heap above 80 KB, plus a SHA-256 known-answer self-test). If it passes, the trial flag is cleared and the image is confirmed. If it fails, or the candidate fails to confirm within 3 attempts (for example a crash loop), it switches the boot partition back to the previous slot. When the old image boots again, it sees a leftover trial flag for a different version, reports `ROLLBACK DETECTED` and increments a persistent rollback counter. The IDF rollback API (`esp_ota_mark_app_invalid_rollback_and_reboot`) is called first, and the manual boot-partition switch is the fallback.

## Test results

Run on an ESP32 WROOM-32 (ESP32-D0WD-V3), Arduino-ESP32 core 3.3.12, default 4 MB partition scheme (`app0` at `0x010000`, `app1` at `0x150000`). The images are 341 KB.

| # | What was sent | What the device did |
|---|---|---|
| 1 | v2, correctly signed | `Image v2 verified (signature + SHA-256) and written to app1`, rebooted, `[TRIAL] v2 CONFIRMED`, running `app1`, IDF image state 2 (valid) |
| 2 | v3, correctly signed, built so its health check fails | Written to `app0`, booted on trial, `simulated peripheral bus timeout`, `[ROLLBACK] health check FAILED`, rebooted into v2, `[ROLLBACK DETECTED] ... Total rollbacks: 1` |
| 3 | v3 with one byte corrupted after signing | `REJECTED: image hash MISMATCH - data corrupted or tampered` (the boot slot is only switched after this check passes) |
| 4 | v3 signed with a different key | `REJECTED: signature INVALID - update not trusted` (rejected from the header, before any data was transferred) |
| 5 | v2 again while v2 is running | `REJECTED: version is not newer than the running firmware (anti-rollback)` |
| 6 | v3, USB cable unplugged at about 50% of the transfer | The host tool stopped with a serial write timeout (the cable was gone). After replugging, the board booted v2 from `app1`, IDF image state 2 (valid), rollback count still 1 (no false trial), and a re-sent v2 was still rejected as "not newer" |

**Not tested yet:** a candidate that crashes during the health check (`FW_FAIL_MODE 2`), and completing a normal update after an interrupted transfer.

## Security scope

| Covered | Not covered |
|---|---|
| Corrupted or truncated transfers (SHA-256) | Anyone with physical access can re-flash over USB: **secure boot and flash encryption are not enabled** |
| Unsigned or wrongly signed images (ECDSA) | The version check compares against the running image's compile-time constant, **not a hardware monotonic counter** |
| Re-sending the same or an older version through the update channel | No key rotation or revocation; the private key sits on the host PC |
| A bad candidate image (rollback) | A candidate that crashes before `setup()` runs cannot trigger the app-level rollback |

Other limitations:

- The health check is simple (heap and a SHA-256 self-test). The failure in test 2 is simulated with a compile-time flag in a test build.
- If power is cut in the instant between writing the trial flag and switching the boot slot, the old image can report a rollback that did not happen.
- Transfer is over 115200-baud serial, not Wi-Fi.
- While idle, the sketch treats a received `s` as a status request, so stray bytes on the serial line can print an extra status block.

## Build and run

**Requirements:** Arduino IDE with the *esp32 by Espressif Systems* package, Python 3, and `pip install pyserial cryptography`.

1. `python fota_tool.py keygen` creates `fota_private.pem` (**keep secret**) and `public_key.h`. Keep `public_key.h` next to `FOTA_Node.ino`.
2. In `FOTA_Node.ino` set `FW_VERSION 1` and `FW_FAIL_MODE 0`, then upload over USB.
3. Set `FW_VERSION 2`, choose Sketch > Export Compiled Binary, and copy `FOTA_Node.ino.bin` to `fw_v2_good.bin`. Set `FW_VERSION 3` and `FW_FAIL_MODE 1`, export again, and copy it to `fw_v3_bad.bin`.
4. Close the Serial Monitor, then send the updates:

```bash
python fota_tool.py send fw_v2_good.bin --version 2 --port COM5
python fota_tool.py send fw_v3_bad.bin  --version 3 --port COM5
python fota_tool.py send fw_v3_bad.bin  --version 3 --port COM5 --tamper
python fota_tool.py send fw_v3_bad.bin  --version 3 --port COM5 --badsig
```

```text
├── FOTA_Node/
│   ├── FOTA_Node.ino
│   └── public_key.h
├── fota_tool.py
└── README.md
```

## Roadmap

- [ ] The crash-during-health-check build (`FW_FAIL_MODE 2`) and a retry after an interrupted transfer
- [ ] Enable secure boot and flash encryption
- [ ] Use a hardware monotonic counter (eFuse) for anti-rollback
- [ ] Over-the-air transfer over Wi-Fi (HTTPS)
- [ ] A health check that exercises a real peripheral
