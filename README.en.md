# libnfc-odrfid — libnfc fork for UnitX ODRFID readers

**English** | [Русский](README.md)

This is a fork of [libnfc](https://github.com/nfc-tools/libnfc) that adds
support for **UnitX ODRFID** readers (ODRFID3-M and ODRFID3-N) working in their CCID USB profile.
Everything else is upstream libnfc; the original upstream README is kept at the
end of the Russian [README.md](README.md#upstream-readme-libnfc).

## What the reader is

The readers enumerate as `0483:A26A`, manufacturer "Open Development",
product `ODRFID3-M` / `ODRFID3-N`.

## Changes relative to upstream

| File | Change |
|------|--------|
| `libnfc/drivers/acr122_pcsc.c` | accepts PC/SC reader names `ODRFID` and `Open Development`; accepts the firmware string prefix `Open Development RFID Reader `; firmware string buffer enlarged to 128 bytes and NUL-terminated (was 11); Direct Transmit limited to 254 bytes (one-byte Lc) |
| `libnfc/drivers/acr122_usb.c` | `0483:A26A` added to the supported device list (CCID must be interface 0, which both reader profiles guarantee) |
| `libnfc/drivers/pcsc.c` | `ODRFID` and `Open Development` added to the exclusion list so the generic `pcsc` driver does not claim the reader a second time |
| `contrib/udev/93-pn53x.rules` | rule for direct USB access to `0483:A26A` |

## Choosing a driver

`acr122_pcsc` and `acr122_usb` reach the same device and **cannot be used at
the same time**: `acr122_usb` resets the whole USB device when it opens it,
which breaks the PC/SC daemon session. Build one of them:

| Situation | Build |
|-----------|-------|
| `pcscd` (or Windows PC/SC) is running | `acr122_pcsc` only |
| Linux, direct USB, no `pcscd` | `acr122_usb` only |

Autotools: `./configure --with-drivers=acr122_pcsc` (or `acr122_usb`).
CMake: `-DLIBNFC_DRIVER_ACR122_PCSC=ON` and switch the other drivers off, e.g.

    cmake -S . -B build -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
      -DLIBNFC_DRIVER_ACR122_PCSC=ON -DLIBNFC_DRIVER_ACR122_USB=OFF \
      -DLIBNFC_DRIVER_PCSC=OFF -DLIBNFC_DRIVER_PN53X_USB=OFF \
      -DLIBNFC_DRIVER_ARYGON=OFF -DLIBNFC_DRIVER_PN532_UART=OFF \
      -DLIBNFC_DRIVER_PN532_SPI=OFF -DLIBNFC_DRIVER_PN532_I2C=OFF
    cmake --build build

Note: `nfc-list` opens every device found by every built driver in turn, so a
build with both drivers will disturb its own PC/SC session.

## Setting up the reader on the PC

1. Put the reader into mode 1 or 2 (see the firmware documentation), e.g.
   `python3 tools/odrfid_usbmode.py serial /dev/ttyACM0 --set 1` from the
   firmware repository.
2. **`acr122_pcsc`:** the CCID driver must know the device.
   - Linux (`pcsc-lite` + `libccid`): add `0x0483`, `0xA26A`, `ODRFID CCID` to
     `ifdVendorID`, `ifdProductID`, `ifdFriendlyName` in the libccid
     `Info.plist`, restart `pcscd`. The reader name must start with `ODRFID`
     or `Open Development` for this fork to accept it.
   - Windows: the built-in `usbccid` driver is used; enable Escape commands
     for the device in the registry (`EscapeCommandEnable`, see the firmware
     documentation).
   Check with `pcsc_scan`, then run `nfc-list -v`.
3. **`acr122_usb`:** stop `pcscd`, install the udev rule
   (`sudo cp contrib/udev/93-pn53x.rules /lib/udev/rules.d/`), then run
   `nfc-list -v`.


## License

libnfc is licensed under the GNU Lesser General Public License v3.0 or later;
the changes in this fork are under the same license. Original copyright
notices are kept.
