# webOS-BLE-thinq-poweron-reverse-engineer

Reverse engineered case study of sorts using opencode zen and chatgpt to help analyse.

LG ThinQ uses the LECCP protocol to power on BLE tv.

Below is chatgpt created summary, less because I'm lazy, more because I'm actively trying to publish this so anyone else might be able to integrate into a Bluetooth proxy for Home Assistnat or whatever. Tested using an MSI VenturePro 15 AI with an intel AX211 - which uses the CNvIO2 protocol or whatever its called. putting this up here for anyone in future to be able to reimplement

# LG webOS ThinQ BLE Power-On Reverse Engineering

Reverse-engineering the Bluetooth Low Energy (BLE) wake-up mechanism used by the LG ThinQ Android application to turn on an LG webOS television from standby.

This project investigates whether the same wake-up mechanism can be reproduced from Linux, potentially allowing Home Assistant to wake the television without relying on the ThinQ application.

## How it works

The LG ThinQ application uses a BLE advertising and scanning mechanism called LECCP for television wake-up and discovery.

Rather than simply sending a conventional Bluetooth command, the phone broadcasts a specially constructed BLE advertisement containing:

- A persistent, application-generated phone Bluetooth identity.
- A control identifier (`07 00`).
- The target television's registered Bluetooth address.

The manufacturer-specific advertisement uses company ID `0x00C4` (196) and the service UUID:

`0000feba-0000-1000-8000-00805f9b34fb`

The 14-byte manufacturer payload follows this structure:

```text
PHONE_MAC[6] 07 00 TV_MAC[6]
```

The application simultaneously scans for the television using the service UUID:

`0000feb9-0000-1000-8000-00805f9b34fb`

When the television is detected, the application can identify it and complete its wake-up process.

## Reverse-engineering findings

The ThinQ Android APK (`com.lgeha.nuts`, version `5.2.14331`) was examined to identify the television power-control call chain and reconstruct the BLE wake advertisement.

The investigation established that:

- BLE wake-up advertising is separate from the application's `fd31` GATT remote-control protocol.
- The phone identity is generated and stored by ThinQ rather than obtained directly from Android's Bluetooth adapter address.
- The wake advertisement contains the phone identity, control identifier and television address.
- The application advertises the `feba` service and scans for `feb9`.
- The application uses low-latency BLE advertising and maintains the operation for up to 60 seconds.

## Linux implementation investigation

The intended Linux implementation uses:

- **Bleak** for BLE scanning.
- **BlueZ D-Bus advertising APIs** for BLE advertising.
- **dbus-fast** for D-Bus integration.

The initial hardware investigation was performed on a Fedora 44 laptop with an Intel AX211 Bluetooth adapter and BlueZ 5.87.

The adapter supports both central and peripheral roles.

A concurrency test was successfully completed, demonstrating that the adapter can advertise and scan simultaneously.

The test ran for 60 seconds, detected an existing `feb9` advertiser, and successfully cleaned up after both normal completion and SIGINT cancellation.

Only dummy manufacturer data (`0xFFFF`, `DEADBEEF`) was used during this test. No LG wake command was transmitted.

## Real television observations

#first stage:

A passive BLE scan was performed while an LG television named `[TV] Gym Room` was in standby.

The following advertisement was observed:

| Field | Observed value |
|---|---|
| Device name | `[TV] Gym Room` |
| Bluetooth address | `7E:__:__:__:__:__` | #redacted for privacy.
| RSSI | `-91 dBm` |
| Service UUID | `0000feb9-0000-1000-8000-00805f9b34fb` |
| Manufacturer ID | `0x00C4` |
| Manufacturer data | `09 33 14 12 16 80` |

This is a strong match for the LECCP protocol identified in the ThinQ APK.

Importantly, the television was advertising while in standby.

The observed Bluetooth address may be a private/random address rather than the television's permanent hardware address.

The six manufacturer-data bytes have not yet been decoded.

## Current status

The following have been established:

- ThinQ's app-side BLE wake advertisement has been reconstructed.
- The relevant LECCP service UUIDs and manufacturer identifier have been identified.
- A real LG television has been observed advertising the expected `feb9` service and `0x00C4` manufacturer identifier in standby.
- The Linux Bluetooth adapter can advertise and scan simultaneously.
- Advertising and scanning cleanup has been tested.

The actual LG wake advertisement has **not yet been transmitted from Linux**, and successful television wake-up has not been demonstrated.

## Outstanding questions

- What do the six bytes in the television's manufacturer data represent?
- Can the raw advertisement expose the separate AD type `0x1b` field used by ThinQ?
- Does the television validate the phone identity, advertiser address, or both?
- Does the television require the empty GATT server used by the ThinQ application?
- Will BlueZ's advertising implementation be accepted by the television?
- Can the reconstructed advertisement successfully wake the television?

## Disclaimer

This is an independent reverse-engineering and interoperability research project. It is not affiliated with, endorsed by, or supported by LG Electronics.

The findings describe observed application behaviour and Bluetooth advertisements. Some aspects of the television's firmware-side implementation remain unknown.

## Stage 2 — Passive HCI Advertisement Capture

**Status: Successful**

A passive HCI capture was made using `btmon` while an ordinary Bleak scan was running on the Intel AX211 Bluetooth adapter (`hci0`).

No HCI commands were issued manually, and no adapter settings or BlueZ configuration were changed.

### Target device

The LG TV `[TV] Gym Room` was observed advertising with:

- Bluetooth address: `7E:45:14:82:D1:18`
- Address type: Random (Resolvable)
- Service UUID: `0000feb9-0000-1000-8000-00805f9b34fb`
- Manufacturer company ID: `0x00C4` (LG Electronics)

### Raw advertising data

The relevant HCI Extended Advertising Report contained:

```text
Advertising Data[29]:
02 01 1a
09 ff c4 00 09 33 14 12 16 80
02 0a d4
03 03 b9 fe
08 1b 00 41 83 26 98 03 dc
```

Decoded AD structures:

| AD structure | Meaning |
|---|---|
| `02 01 1a` | Flags |
| `09 ff c4 00 09 33 14 12 16 80` | LG manufacturer-specific data |
| `02 0a d4` | TX Power |
| `03 03 b9 fe` | Complete 16-bit service UUID list containing `feb9` |
| `08 1b 00 41 83 26 98 03 dc` | Unknown AD type `0x1B`, containing the registration-related data |

### AD type `0x1B`

The final structure is particularly important because the ThinQ APK contains a parser for AD type `0x1B`.

Its data is:

```text
00 41 83 26 98 03 dc
```

The APK's parsing logic reverses the six address bytes following the leading `00`:

```text
41 83 26 98 03 dc
        ↓ reverse
dc 03 98 26 83 41
```

This gives the candidate address:

```text
DC:03:98:26:83:41
```

This is an APK-derived interpretation of the field. The meaning of the leading `00` and the candidate address's exact role have not yet been independently verified.

### Additional observations

- The TV's manufacturer data matches the earlier Bleak scan.
- The TV was advertising as a scannable, non-connectable legacy advertising device (`ADV_SCAN_IND`).
- Its scan response contained the complete local name `[TV] Gym Room`.
- The same advertisement was observed while the laptop was simultaneously advertising and scanning during Stage 1.

### What this establishes

The TV's advertisement contains the exact AD type `0x1B` expected by the ThinQ APK parser. We can now inspect the actual bytes rather than relying on an interpretation of the decompiled code alone.

This does not yet establish that the candidate address is accepted by the TV's wake-up protocol, or that a Linux implementation can successfully wake it.

### Outstanding questions

- What does the leading `00` represent?
- Is the derived address the exact address used for TV registration?
- What additional conditions does the TV require before accepting a wake-up advertisement?
