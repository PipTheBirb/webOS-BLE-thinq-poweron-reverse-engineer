# LECCP BLE Wake-Up Advertisement Specification (Draft)

**Status:** Draft specification derived from read-only static analysis of the LG ThinQ APK and local BlueZ documentation/binary inspection.

**Scope:** This document describes the LECCP BLE wake-up advertisement that the Android app constructs when invoking `powerOnBLEWakeup`. It covers the exact manufacturer payload, advertising configuration, scan response, phone-side GATT server behaviour, BlueZ mapping, and the boundary between confirmed APK behaviour and unproven assumptions.

**No implementation or transmission is included here.** No advertisements were sent, no Bluetooth/D-Bus/HCI commands were run, no TV connection was attempted, and no system settings were changed.

---

## 1. Executive Summary

The app constructs a legacy BLE advertising set with:

- A 16-bit service UUID: `0000feba-0000-1000-8000-00805f9b34fb` (`UUID_LECCP_CONTROL`).
- Manufacturer Specific Data using company ID `0x00C4`.
- A 14-byte manufacturer payload:

```text
phoneBtAddress[0..5]  07 00  tvRegisteredAddress[0..5]
```

The payload consists of:

1. Six bytes of the app-persisted phone Bluetooth identity, in presentation order.
2. A ControlData length byte `0x07`.
3. A control ID byte `0x00` (wake control).
4. Six bytes of the TV address from `registered_bt_address`, in presentation order.

No address reversal is applied to this payload. The reversal seen in the TV registration advertisement's AD type `0x1B` applies to that advertisement's parsing, not to this phone control payload.

The app also opens a GATT server and adds a primary service with UUID `feba` and zero characteristics. The wake-success path in the app is driven by advertisement matching, not GATT traffic. Whether the TV firmware requires the GATT server or connection remains unproven.

---

## 2. Source Basis and Terminology

### 2.1 APK and decompiled source

- APK: `base.apk`
- Decompiled tree: `/tmp/opencode/lg/base`
- Smali class paths referenced below are under `/tmp/opencode/lg/base/`.

### 2.2 Terms

- **APK-set value:** Explicitly set by the app through Android APIs.
- **Android stack value:** Supplied by Android framework/Bluetooth stack defaults or generated on air.
- **BlueZ value:** A D-Bus property or BlueZ/kernel-generated field on Linux.
- **Presentation order:** The byte order written in a colon-separated MAC string, e.g. `dc:03:98:26:83:41` -> `DC 03 98 26 83 41`.
- **On-air AD structure:** Length byte + AD type + AD data, as specified by Bluetooth GAP.

---

## 3. Confirmed Exact Manufacturer Payload

### 3.1 Construction path

The wake control object is constructed in `ConnectSdkHandler.powerOnBLEWakeup`:

1. A new `LeccpControlAdvertisingInfo` is created with the phone Bluetooth identity bytes:

   - `ConnectSdkHandler.smali:5985-5987`
   - Constructor parameter is `targetAddress` (`LeccpControlAdvertisingInfo.smali:42-57`).

2. The TV address from `registered_bt_address` is converted to six bytes and added as one `ControlData`:

   - `ConnectSdkHandler.smali:5989-5993`
   - The control ID passed at this call site is `0`. The local used for the call is set to `0` at `ConnectSdkHandler.smali:5854`.
   - The field name `CONTROL_WAKEUP` exists at `LeccpControlAdvertisingInfo.smali:24`, but the field is not directly read in this path; the call site passes `0`.

3. `getBytes()` produces the final manufacturer payload:

   - `ConnectSdkHandler.smali:6001`
   - `LeccpControlAdvertisingInfo.smali:118-176`

### 3.2 Byte layout

| Payload offset | Length | Value | Meaning / source |
|---|---:|---|---|
| `0..5` | 6 | Phone identity bytes, presentation order | From `BtNetwork.getAddress` / `NetworkUtil.getOrCreateRandomizedMacAddress`, converted by `CUtil.getBytesFromMacAddress` |
| `6` | 1 | `0x07` | ControlData length byte = `2 + 6 - 1` |
| `7` | 1 | `0x00` | Control ID passed at the wake call site |
| `8..13` | 6 | TV address bytes, presentation order | From lowercased `registered_bt_address`, converted by `CUtil.getBytesFromMacAddress` |

Total: **14 bytes**.

### 3.3 ControlData encoding

`ControlData.getByteSize()` returns `2 + controlData.length`.

For a six-byte TV address:

```text
getByteSize() = 2 + 6 = 8
```

`ControlData.getBytes()` emits:

```text
[getByteSize() - 1] [controlId] [controlData...]
```

Therefore:

```text
[0x07] [0x00] [tvAddress[0..5]]
```

— `ControlData.smali:42-113`

`LeccpControlAdvertisingInfo.getBytes()` concatenates:

```text
targetAddress + each ControlData.getBytes()
```

— `LeccpControlAdvertisingInfo.smali:118-176`

### 3.4 Phone identity source and byte order

The phone identity is not necessarily the Bluetooth adapter's hardware address.

- `BtNetwork.getAddress` reads the app setting `BtNetwork.macAddress` from `SettingsDb`.
- If absent, it calls `NetworkUtil.getOrCreateRandomizedMacAddress()` and persists the generated value.
- `NetworkUtil.createRandomUnicastAddress()` generates a random locally administered unicast EUI-48-style address.
- `NetworkUtil.stringAddrFromLongAddr()` formats it as lowercase `%02x:%02x:%02x:%02x:%02x:%02x`.

Evidence:

- `BtNetwork.smali:204-228`
- `NetworkUtil.smali:238-264`
- `NetworkUtil.smali:365-421`
- `NetworkUtil.smali:1663-1735`
- `BtNetwork.smali:31-43` (`BtNetwork.macAddress` key construction)

`CUtil.getBytesFromMacAddress` splits the string on `:`, parses each component as hexadecimal, and stores bytes in the same order. It does **not** reverse them.

— `CUtil.smali:365-413`

### 3.5 TV identity source and byte order

The TV identity is `registered_bt_address` from the TV's `ThingsDevice` data.

- `ConnectSdkHandler.powerOnBLEWakeup` reads `registered_bt_address` — `ConnectSdkHandler.smali:5819-5823`.
- It lowercases the value — `:5844`.
- It converts the string to bytes with `CUtil.getBytesFromMacAddress` — `:5989`.
- Those bytes are passed as the ControlData payload — `:5993`.

The bytes are in presentation order. No reversal is applied here.

The AD type `0x1B` reversal in `LeccpAdvertisingInfo.getAddress` is separate and applies when parsing the TV's registration advertisement, not when constructing the phone's control payload.

### 3.6 Illustrative example

The following uses a synthetic phone identity and the captured TV address:

```text
Phone identity:        02:11:22:33:44:55
TV registered address: dc:03:98:26:83:41
```

Manufacturer payload:

```text
02 11 22 33 44 55 07 00 DC 03 98 26 83 41
```

Breakdown:

```text
02 11 22 33 44 55   phone identity, 6 bytes
07                  ControlData length byte
00                  wake control ID
DC 03 98 26 83 41   TV address, 6 bytes
```

The phone identity above is illustrative only. The TV address is the captured registration-advertisement decode.

---

## 4. Manufacturer AD Structure and Service UUID

### 4.1 Manufacturer Specific Data AD

The app passes:

```text
manufacturer ID = 0xC4
manufacturer data = 14-byte payload
```

— `BleAdvertiser.smali:513-525`, `:201`

On air, Android encodes this as:

```text
11 FF C4 00 [14-byte payload]
```

| Field | Bytes | Notes |
|---|---|---|
| AD length | `0x11` | 17 bytes after the length byte: 1 type + 2 company ID + 14 payload |
| AD type | `0xFF` | Manufacturer Specific Data |
| Company ID | `C4 00` | `0x00C4` little-endian on air |
| Payload | 14 bytes | Phone identity + `07 00` + TV address |

Full synthetic example:

```text
11 FF C4 00
02 11 22 33 44 55 07 00 DC 03 98 26 83 41
```

The manufacturer AD is **18 bytes total on air**, including its length byte.

### 4.2 Service UUID

The advertised service UUID is:

```text
0000feba-0000-1000-8000-00805f9b34fb
```

— `BleModel$BleService.smali:35-41` (`UUID_LECCP_CONTROL`)

The app adds it to the primary advertising data with `addServiceUuid` — `BleAdvertiser.smali:203`.

Expected legacy on-air encoding for the 16-bit form:

```text
03 03 BA FE
```

| Field | Bytes | Notes |
|---|---|---|
| AD length | `0x03` | 3 bytes after the length byte |
| AD type | `0x03` | Complete List of 16-bit Service Class UUIDs |
| UUID | `BA FE` | `0xFEBA` little-endian |

This specific encoding is an inference from standard Bluetooth/Android/BlueZ behaviour; the APK supplies the UUID through the Android API rather than raw AD bytes.

### 4.3 Other service UUID in the APK

`UUID_LECCP_SERVICE` is a different UUID:

```text
0000feb9-0000-1000-8000-00805f9b34fb
```

— `BleModel$BleService.smali:27-33`

- The phone's own scanner filters for `feb9` when looking for TV registration advertisements.
- The phone's wake control advertisement uses `feba`.
- The phone-side GATT server also adds a `feba` service.
- The `onServiceAdded` callback checks for `feb9`, not `feba` (see GATT section).

### 4.4 Primary packet size

Without a Flags AD:

```text
Service UUID AD:        4 bytes
Manufacturer AD:       18 bytes
                       --------
                       22 bytes
```

With a 3-byte Flags AD:

```text
Flags AD:               3 bytes
                       --------
                       25 bytes
```

Both fit the legacy 31-byte advertising data limit. The APK defines a legacy limit constant of `0x1f` (31) at `BleAdvertiser.smali:15`.

AD structure order is not semantically significant and may differ between Android and BlueZ.

---

## 5. Android Advertising Configuration

### 5.1 Settings explicitly set by the APK

`BleAdvertiser.startAdvertising` creates `AdvertiseSettings.Builder` and calls only:

```text
setAdvertiseMode(2)
```

— `BleAdvertiser.smali:187-195`

Android constant `2` is:

```text
ADVERTISE_MODE_LOW_LATENCY
```

### 5.2 Settings not explicitly set

The APK does **not** call:

- `setConnectable(...)`
- `setTimeout(...)`
- `setTxPowerLevel(...)`

Therefore the following are Android framework defaults, not APK-defined values:

| Parameter | Expected Android default | Confidence |
|---|---|---|
| Connectable | `true` | Framework default; not APK-set |
| Timeout | `0` (no timeout) | Framework default; not APK-set |
| TX power level | Medium | Framework default; not APK-set |

These defaults must not be presented as APK constants.

### 5.3 Primary advertising data

The APK builds primary `AdvertiseData` with:

- Manufacturer data `0xC4` + 14-byte payload — `BleAdvertiser.smali:201`
- Service UUID `feba` — `:203`

It does not add:

- Device name to primary data
- TX power
- Service data
- Solicit UUIDs
- Appearance
- Explicit Flags AD

### 5.4 Scan response

The APK builds a separate scan-response `AdvertiseData` with only:

```text
setIncludeDeviceName(includeDeviceName)
```

— `BleAdvertiser.smali:249-259`

The flag is normally `1`, but is changed to `0` when the device name's UTF-8 byte length exceeds 31 bytes — `:209-253`.

Device name source:

- `DeviceNameManager.getDeviceName()` — `DeviceNameManager.smali:391`
- LG build: `lg_device_name` setting.
- Non-LG build: `BluetoothAdapter.getName()`.
- Default internal value: `"Unknown"`.
- If no context is available, returns empty string.

Implications:

- If the encoded device name is <=31 bytes, Android includes it in the scan response.
- If the encoded device name is >31 bytes, `includeDeviceName` is set to `false`, producing an empty scan response.
- No service UUID, manufacturer data, or TX power is placed in the scan response.

The scan response is not part of the wake payload.

### 5.5 Stack-generated fields

The APK does not construct raw advertising bytes. Android's Bluetooth stack may add or control:

- Flags AD
- Advertising interval for the selected mode
- Advertising address/privacy handling
- Any required formatting of service UUID and manufacturer AD structures
- TX power behaviour

The exact on-air Flags byte and interval are **not** established by the APK alone.

---

## 6. GATT Server Behaviour

### 6.1 When the server is opened

After advertising starts successfully, `BleAdvertiser$2.onSuccess()` opens a GATT server if one is not already open:

```text
BluetoothManager.openGattServer(context, serverCallback)
```

— `BleAdvertiser$2.smali:107-139`

It then adds the static LECCP service if `mIsServiceAdded` is false — `:162-180`.

`onStartFailure` treats error codes `0` and `3` as success and invokes the same path — `BleAdvertiser$2.smali:253-269`.

### 6.2 Service UUID and characteristics

The static service is created as:

```text
new BluetoothGattService(UUID_LECCP_CONTROL, 0)
```

— `BleAdvertiser.smali:46-54`

Therefore:

| Property | Value |
|---|---|
| Service UUID | `0000feba-0000-1000-8000-00805f9b34fb` |
| Service type | `0` = primary |
| Characteristics | None |

There are no `addCharacteristic` calls in `BleAdvertiser`, and no characteristic UUIDs are added to this service.

### 6.3 GATT callbacks

`BleAdvertiser$1` is a `BluetoothGattServerCallback` with only:

- `onConnectionStateChange`
- `onServiceAdded`

— `BleAdvertiser$1.smali:34-169`

There are no:

- Characteristic read request handlers
- Characteristic write request handlers
- Descriptor handlers
- Notification/indication handlers

The GATT server therefore exposes no usable characteristic data.

### 6.4 The `feba`/`feb9` inconsistency

The service added to the GATT server uses `UUID_LECCP_CONTROL` = `feba`:

- `BleAdvertiser.smali:48`

However, `onServiceAdded` only sets `mIsServiceAdded = true` when the added service UUID equals `UUID_LECCP_SERVICE` = `feb9`:

- `BleAdvertiser$1.smali:150-166`

Result:

- Adding the `feba` service does not satisfy the `feb9` check.
- `mIsServiceAdded` remains false.
- `onConnectionStateChange` may add the `feba` service again on subsequent connection-state changes, because it checks `!mIsServiceAdded` before adding — `BleAdvertiser$1.smali:74-100`.

This is observed APK behaviour. It may be an app bug or an intentional mismatch; the investigation did not establish intent.

### 6.5 Is the GATT server required?

APK-side facts:

- The app opens a GATT server and attempts to expose a `feba` service.
- The service has zero characteristics.
- The app's wake-success path is driven by advertisement matching:
  - `BleScanner` parses the TV registration advertisement.
  - `LBSManager` updates the device.
  - `ConnectSdkHandler$45.notifyBleDevice` is invoked on address match.
- The app does not require GATT traffic to declare its own wake attempt successful.

APK-side conclusion:

- The GATT server is **not required by the app's own success path**.
- A minimal candidate for wake-up is advertisement-only.

TV-side unknown:

- Whether the TV firmware requires a connectable advertiser or a GATT connection cannot be proven from the APK.
- If the TV requires a connection, the APK-shaped mirror would be connectable advertising plus an empty primary `feba` GATT service.
- Whether the TV accepts an empty `feba` service, or requires a characteristic, is unknown.

### 6.6 TV-side service distinction

The APK also contains characteristic UUIDs such as:

```text
2ab3, 2ab4, 2ab5, 2ab8, 2ab9, 2aba, 2abb, 2abd, 2abe, 2abf
```

— `BleModel$BleCharacteristic.smali:18-131`

These are used by `LBSManager` against the TV-side service `feb9`:

- `LBSManager` uses `UUID_LECCP_SERVICE` = `feb9` — `LBSManager.smali:2557`
- The wake GATT service on the phone is `feba`.

Therefore, the `2ab*` characteristics should **not** be treated as characteristics of the phone's advertised wake service. They appear to belong to the TV-side LECCP service and are used for other GATT interactions.

---

## 7. BlueZ 5.87 Mapping

### 7.1 Local BlueZ facts

Installed BlueZ package:

```text
bluez-5.87+1.git8750129efca8-1.fc44.x86_64
```

The local `/usr/libexec/bluetooth/bluetoothd` binary contains strings for:

```text
org.bluez.LEAdvertisement1
org.bluez.LEAdvertisingManager1
ScanResponseData
ScanResponseManufacturerData
ScanResponseServiceData
ScanResponseServiceUUIDs
ScanResponseSolicitUUIDs
MinInterval
MaxInterval
SupportedCapabilities
SupportedFeatures
```

This establishes that the local binary includes the relevant advertising API names. It does not establish that experimental D-Bus properties are enabled.

### 7.2 `LEAdvertisement1` mapping

| APK/Android behaviour | BlueZ `LEAdvertisement1` mapping | Notes |
|---|---|---|
| Advertising mode `2` (low latency) | No direct mode property | Use `MinInterval`/`MaxInterval` only if an exact interval is established; the APK does not specify one |
| Default connectable `true` | `Type="peripheral"` | BlueZ sets the connectable advertising flag for `peripheral` |
| No timeout | Omit `Duration` and `Timeout` | APK does not set a timeout |
| Manufacturer ID `0xC4` + 14 bytes | `ManufacturerData={0x00C4: payload}` | Do not include `11 FF C4 00` in the D-Bus value |
| Service UUID `feba` | `ServiceUUIDs=["0000feba-0000-1000-8000-00805f9b34fb"]` | Do not include `03 03 BA FE` in the D-Bus value |
| Scan-response local name | `LocalName` if name <=31 bytes | BlueZ normally places the name in the scan response for legacy peripheral advertising |
| No explicit TX power | Omit `Includes=["tx-power"]` and `TxPower` | APK does not request TX power AD |
| No explicit Flags AD | Let BlueZ/kernel generate | Use `Discoverable=true` only if a capture confirms the Android `0x06` value |
| Empty `feba` GATT service | `GattManager1.RegisterApplication` with a `GattService1` UUID `feba`, `Primary=true`, no characteristics | BlueZ acceptance and TV dependency are unproven |

### 7.3 BlueZ behaviour and limitations

Confirmed from BlueZ source:

- `Type="peripheral"` sets `MGMT_ADV_FLAG_CONNECTABLE` in `get_adv_flags()`.
- `ServiceUUIDs` is parsed into the Service UUID AD data.
- `ManufacturerData` is parsed into Manufacturer Specific Data; the D-Bus key is the numeric company ID.
- `LocalName` is added to the scan response for legacy peripheral advertising when scan response is used.
- `Duration` and `Timeout` are optional.
- `MinInterval` and `MaxInterval` are optional and are converted from milliseconds to 0.625 ms units.
- `ScanResponse*` parsers are experimental and are ignored when experimental D-Bus support is disabled.
- Flags can be generated by the kernel/BlueZ; `Discoverable` and adapter state affect the exact Flags bits.

Limitations and differences:

1. **No direct advertising-mode equivalent**
   - Android's `ADVERTISE_MODE_LOW_LATENCY` is not a BlueZ property.
   - BlueZ's `MinInterval`/`MaxInterval` are explicit intervals, not modes.
   - The APK does not provide an interval value, so any interval chosen on Linux would be an implementation decision, not an APK-derived fact.

2. **Flags are stack-controlled**
   - The APK does not set a Flags AD.
   - BlueZ may emit flags based on `Type`, `Discoverable`, and adapter state.
   - Exact on-air flag bits require capture verification.

3. **Scan response properties are experimental**
   - BlueZ 5.87 contains `ScanResponse*` support, but those parsers are experimental.
   - Stage 1 found `AdvertisementMonitorManager1` absent, consistent with experimental D-Bus interfaces being disabled.
   - Do not rely on `ScanResponseManufacturerData` or similar for the required wake payload.

4. **Device-name length handling differs**
   - Android disables scan-response device name inclusion if the name exceeds 31 bytes.
   - BlueZ's `LocalName` truncates if it does not fit.
   - To mirror Android, omit `LocalName` when the UTF-8 name exceeds 31 bytes.

5. **Empty GATT service is not proven**
   - The APK creates a service with zero characteristics.
   - BlueZ may accept an empty `GattService1`, but this was not tested.
   - Even if BlueZ accepts it, whether the TV needs it is unknown.

6. **GATT characteristics differ**
   - The APK's phone-side wake service has no characteristics.
   - The `2ab*` UUIDs are associated with the TV-side `feb9` service in `LBSManager`.
   - They should not be added to the phone's `feba` service merely to fill it.

---

## 8. Confirmed Facts, Inferences, and Unknowns

### 8.1 Confirmed facts

1. The manufacturer payload is 14 bytes.
2. The payload layout is `phoneBtAddress[6] + 07 00 + tvRegisteredAddress[6]`.
3. The ControlData length byte is `0x07`.
4. The control ID passed at the wake call site is `0x00`.
5. The phone identity comes from the app-persisted `BtNetwork.macAddress` value, generated as a random locally administered unicast EUI-48-style address if absent.
6. The TV address comes from lowercased `registered_bt_address`.
7. `CUtil.getBytesFromMacAddress` preserves presentation order and does not reverse bytes.
8. The AD `0x1B` reversal used for TV registration parsing does not apply to the phone control payload.
9. The advertised service UUID is `0000feba-0000-1000-8000-00805f9b34fb`.
10. The TV/registration service UUID is `0000feb9-0000-1000-8000-00805f9b34fb`.
11. The manufacturer company ID is `0x00C4`.
12. The APK sets advertising mode `2`.
13. The APK does not explicitly set connectable, timeout, or TX power.
14. The APK's primary advertising data contains the manufacturer data and the `feba` service UUID.
15. The APK's scan response contains only the conditional device name.
16. The APK opens a GATT server after advertising start success.
17. The APK adds a primary GATT service `feba` with zero characteristics.
18. The GATT server callback has no characteristic read/write/notification handlers.
19. `onServiceAdded` checks for `feb9`, not the added `feba` service.
20. The app's wake-success path is based on advertisement address matching, not GATT data exchange.
21. The APK defines a legacy advertising data limit of 31 bytes.
22. The local BlueZ 5.87 binary contains `LEAdvertisement1`, `LEAdvertisingManager1`, scan-response, interval, and capability API strings.
23. BlueZ `Type="peripheral"` maps to connectable advertising.
24. BlueZ scan-response properties are experimental.

### 8.2 Inferences

1. Android's Bluetooth stack inserts the Flags AD; the APK does not construct it.
2. The expected Android/BlueZ on-air service UUID AD for `feba` is `03 03 BA FE`.
3. The expected on-air manufacturer AD for company `0x00C4` and the 14-byte payload is `11 FF C4 00` + payload.
4. The exact Android advertising interval for mode `2` is stack/controller-dependent and was not captured from the APK.
5. Android's default connectable setting for this `AdvertiseSettings` is `true`.
6. Android's default timeout is `0` (no timeout).
7. Android's default TX power level is medium.
8. BlueZ will normally place `LocalName` in the scan response for legacy peripheral advertising.
9. The `2ab*` characteristic UUIDs are TV-side LECCP characteristics, not characteristics of the phone's `feba` wake service.
10. An empty `feba` GATT service may be accepted by BlueZ, but this is not proven.
11. Advertisement-only may be sufficient for TV wake-up, but this is not proven.

### 8.3 Unknowns

1. Whether the TV firmware requires a connectable advertisement for wake-up.
2. Whether the TV firmware requires a GATT connection for wake-up.
3. Whether the TV firmware requires the phone-side `feba` GATT service.
4. Whether the TV firmware accepts an empty `feba` GATT service.
5. Whether the TV validates the full 14-byte payload, only the phone/TV address bytes, the company ID, the service UUID, or a subset.
6. The exact Android on-air Flags byte in the relevant phone state.
7. The exact Android advertising interval/controller timing associated with mode `2`.
8. Whether BlueZ 5.87 exposes the `ScanResponse*` properties when experimental D-Bus is disabled.
9. Whether BlueZ will accept the APK's zero-characteristic GATT service without modification.
10. Whether the TV-side `feb9` GATT service/characteristics play any role in the wake-up sequence after the advertisement match.

---

## 9. Relevant APK References

### 9.1 Payload construction

- `/tmp/opencode/lg/base/smali_classes4/com/lge/lms/model/BleModel$LeccpRegistInfo$LeccpControlAdvertisingInfo.smali`
  - Fields: `:24-38`
  - Constructor: `:42-57`
  - `getByteSize`: `:62-96`
  - `addControlData`: `:100-116`
  - `getBytes`: `:118-176`

- `/tmp/opencode/lg/base/smali_classes4/com/lge/lms/model/BleModel$LeccpRegistInfo$LeccpControlAdvertisingInfo$ControlData.smali`
  - Fields: `:17-21`
  - `getByteSize`: `:42-59`
  - `getBytes`: `:61-113`

- `/tmp/opencode/lg/base/smali_classes12/com/lge/lms/things/service/smarttv/connectsdk/ConnectSdkHandler.smali`
  - `powerOnBLEWakeup`: `:5801`
  - Read `registered_bt_address`: `:5819-5823`
  - Lowercase TV address: `:5844`
  - Phone BT address bytes: `:5858-5862`
  - Control ID local set to `0`: `:5854`
  - Construct control info: `:5985-5987`
  - TV address bytes: `:5989`
  - Add ControlData: `:5993`
  - `getBytes`: `:6001`
  - Start advertise: `:6005`

### 9.2 Address conversion

- `/tmp/opencode/lg/base/smali_classes12/com/lge/lms/common/CUtil.smali`
  - `getBytesFromMacAddress`: `:365-413`
  - No reversal; six colon-separated components parsed in order.

- `/tmp/opencode/lg/base/smali_classes12/com/lge/lms/connectivity/network/BtNetwork.smali`
  - `getAddress`: `:204-228`
  - Key construction: `:31-43`

- `/tmp/opencode/lg/base/smali_classes12/com/lge/lms/connectivity/network/NetworkUtil.smali`
  - `createRandomUnicastAddress`: `:238-264`
  - `getOrCreateRandomizedMacAddress`: `:365-421`
  - `stringAddrFromLongAddr`: `:1663-1735`

### 9.3 Advertising

- `/tmp/opencode/lg/base/smali_classes12/com/lge/lms/model/BleModel$BleService.smali`
  - `UUID_LECCP_SERVICE` = `feb9`: `:27-33`
  - `UUID_LECCP_CONTROL` = `feba`: `:35-41`

- `/tmp/opencode/lg/base/smali_classes12/com/lge/lms/connectivity/advertiser/BleAdvertiser.smali`
  - Max legacy data: `:15`
  - Static `sLeccpService` with `UUID_LECCP_CONTROL`: `:19`, `:46-54`
  - `startAdvertising`: `:181`
  - Mode `2`: `:191`
  - `addManufacturerData(0xC4, payload)`: `:201`
  - `addServiceUuid(feba)`: `:203`
  - Scan-response device-name flag: `:209-253`
  - Android `startAdvertising`: `:265`
  - `start`: `:393`
  - Start arguments (`2`, `0xC4`, `feba`, payload): `:503-525`
  - `stop`: `:545-695`

- `/tmp/opencode/lg/base/smali_classes12/com/lge/lms/connectivity/advertiser/AdvertiseManager.smali`
  - `startAdvertise`: `:243-279`
  - Calls `BleAdvertiser.start`: `:275`

### 9.4 GATT server

- `/tmp/opencode/lg/base/smali_classes12/com/lge/lms/connectivity/advertiser/BleAdvertiser$2.smali`
  - `onSuccess`: `:86-230`
  - `openGattServer`: `:107-139`
  - Add service: `:162-180`
  - Set listen flag: `:193-197`
  - `onStartFailure` treats `0`/`3` as success: `:253-269`

- `/tmp/opencode/lg/base/smali_classes12/com/lge/lms/connectivity/advertiser/BleAdvertiser$1.smali`
  - `onConnectionStateChange`: `:34-114`
  - Add service on connection state change: `:74-100`
  - `onServiceAdded`: `:116-169`
  - Check against `UUID_LECCP_SERVICE` (`feb9`): `:150-166`

### 9.5 TV-side service and characteristics

- `/tmp/opencode/lg/base/smali_classes12/com/lge/lms/model/BleModel$BleCharacteristic.smali`
  - Characteristic UUID fields: `:18-38`
  - Values: `:42-131`

- `/tmp/opencode/lg/base/smali_classes12/com/lge/lms/connectivity/LBSManager.smali`
  - Uses `UUID_LECCP_SERVICE` (`feb9`): `:2557`

- `/tmp/opencode/lg/base/smali_classes12/com/lge/lms/connectivity/LBSManager$LBSGattCallback.smali`
  - Uses `UUID_LECCP_SERVICE`: `:1150`

### 9.6 Device name

- `/tmp/opencode/lg/base/smali_classes12/com/lge/lms/external/util/DeviceNameManager.smali`
  - `getDeviceName`: `:391-509`
  - LG setting `lg_device_name` handling: `:403-476`
  - Non-LG adapter name: `:478-508`

### 9.7 BlueZ documentation/source consulted

- BlueZ `LE Advertising API`: `doc/org.bluez.LEAdvertisement.rst`
- BlueZ `LEAdvertisingManager1`: `doc/org.bluez.LEAdvertisingManager1.rst`
- BlueZ advertising implementation: `src/advertising.c`
  - `parse_type`
  - `parse_service_uuids`
  - `parse_manufacturer_data`
  - `parse_includes`
  - `parse_local_name`
  - `parse_duration`
  - `parse_timeout`
  - `generate_adv_data`
  - `generate_scan_rsp`
  - `get_adv_flags`
  - `parsers[]` experimental flags

---

## 10. Practical Specification Boundary

If the objective is to reproduce the APK's advertising set as closely as possible:

1. Advertise as connectable peripheral.
2. Use service UUID `0000feba-0000-1000-8000-00805f9b34fb`.
3. Use manufacturer company ID `0x00C4`.
4. Use the 14-byte payload `phone[6] + 07 00 + tv[6]`.
5. Use the same phone identity value that the phone-side app would use, not necessarily the Linux adapter's hardware address.
6. Use the TV's `registered_bt_address` in presentation order, without reversal.
7. Preserve the distinction between APK-set values and Android/BlueZ stack-generated fields.
8. Treat the empty `feba` GATT service as APK-observed behaviour, not as a proven TV requirement.

If the objective is to establish the minimum required for wake-up:

- The APK's own success path is advertisement-only.
- Whether the TV requires connectability or GATT remains unknown and must be established experimentally, not assumed from the APK.

---

## 11. Document Limitations

- This document is a static-analysis specification, not a tested interoperability result.
- No advertisement was transmitted.
- No GATT server was created.
- No BlueZ D-Bus introspection was performed in this step.
- No TV connection or wake test was performed.
- The TV-side GATT requirement is explicitly unresolved.
- The exact Android on-air Flags and interval were not measured.
- BlueZ empty-service acceptance was not tested.

**End of draft.**
