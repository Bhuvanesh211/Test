# FOTA Package Unpacker — Debug Trigger: Work Report

**Author:** Bhuvanesh
**Module:** `vendor/melco/appservice/middle/FOTA`
**Component touched:** `UsbUpdateService` (front-end) + shared `Common/library` unpacker (logging only)
**Environment:** Android emulator (x86_64), no real USB media, no VCPU, no boot-control HAL.

---

## 1. Objective (what I was asked to do / what I set out to do)

Exercise and trace the **FOTA package unpacker** (`PackageUnpacker`) on the
emulator, where the normal production trigger can never fire, and produce log
evidence of the unpack flow:

- The real unpacker is only reached after a long chain of gates: USB media
  mounted → Arbiter validates file name → VCPU put into engineering/SWDL mode →
  boot-control HAL queried. On the emulator **none of these pass** (no USB, no
  VCPU — `ENGINEERING MODE REQUEST` times out; no boot HAL — `boot control hal
  service not found`).
- So we needed a **developer-only trigger** to call `PackageUnpacker.unpack()`
  directly, on demand, without changing any production behaviour.

---

## 2. Background — how the unpacker is normally triggered

The unpacker library (`com.mitsubishielectric.ahu.appservice.ota.common.unpacker`)
is front-end agnostic. Each OTA service has a thin `Unpacker` (a `Runnable`)
that wraps the shared `PackageUnpacker`.

| Front-end | Trigger chain | Uses `PackageUnpacker`? |
|-----------|---------------|--------------------------|
| **USB** (`UsbUpdateService`) | USB mounted → `USBUpdateStateMachine` → `UnpackingState.enter()` → `getUnpacker().unpack()` → `google/unpack/Unpacker` → `PackageUnpacker.unpack(MCPU_VCPU)` | Yes |
| **WiFi** (`WiFiUpdateService`) | RB agent finishes → `FOTAStateMachine` `UnpackingState` → `rb/unpack/Unpacker` → `PackageUnpacker.unpack(MCPU_VCPU)` | Yes |
| **SimpleUpdate** (`SimpleUpdateService`) | media mounted → plain-copies raw artifacts → drives updaters directly | **No** (bypasses the unpacker entirely) |

Both real callers always enter through the same method: `PackageUnpacker.unpack(int type)`.
That is the single point I targeted for the debug trigger.

---

## 3. What I changed, and why

All code changes are confined to **one production file**
(`UsbUpdateService.java`) and are **additive** — they do not alter the existing
USB/VCPU/install flow. The shared `PackageUnpacker` was changed for **logging only**.

### 3.1 Trace logging (`BHUVANESH_UNPACK` markers)
Added step-by-step `CommonLog.i(...)` markers along the whole pipeline so the
flow is visible in `logcat`:
- USB adapter `Unpacker.startUnpackThread()` and WiFi adapter `Unpacker.startUnpackThread()` — show **which front-end** entered.
- `PackageUnpacker.unpack()` — `ENTRY`, `STEP1 loadJsonData`, `STEP2 loadPackageHeader`, `STEP3/4 MCPU/VCPU present`, `STEP5 extract`, `STEP6 result`, `EXIT`.
- `decrypt() / decryptMCPU() / decryptVCPU() / extract()` — entry + the native `SecurePackageHandler` status.

**Why:** the lead asked to *see* where the unpacker is called from and how the
flow proceeds. These markers make the entire path greppable with a single
filter: `adb logcat | grep BHUVANESH_UNPACK`.

### 3.2 Debug trigger in `UsbUpdateService`
- A `BroadcastReceiver` (`mDebugUnpackReceiver`) registered in `onCreate()` for a
  custom action `…ota.google.DEBUG_UNPACK`, exported so `adb` can reach it.
- A shared private method `runDebugUnpack(file, type)` that builds a
  `PackageUnpacker` and calls `unpack(type)` on a worker thread — the **same
  entry point the production USB flow uses**.
- A branch in `onStartCommand()` that handles the `DEBUG_UNPACK` action and then
  returns, so it never falls through into the real `initUsbUpdateService()`.
- A registration-confirmation log line in `onCreate()`.

**Why each piece:**
- *Receiver* → lets us trigger via `am broadcast`.
- *`onStartCommand` branch* → lets us trigger via `am start-service`, which
  **also starts the service if it is not running** (the broadcast path cannot).
- *Shared `runDebugUnpack`* → no duplicated logic between the two entry points.
- *Registration log* → a litmus test: if it is absent after a flash, the running
  APK is stale; if present but the broadcast still does nothing, it is a delivery
  problem, not a code problem.

---

## 4. How it is triggered

### 4.1 Manual trigger — broadcast (works only if the service is already alive)
```
adb shell am broadcast \
    -a com.mitsubishielectric.ahu.appservice.ota.google.DEBUG_UNPACK \
    --es file /data/vendor/ota/usbupdateservice/test.swpkg --ei type 4
```

### 4.2 Manual trigger — start-service (also auto-starts the service)
```
adb root
adb shell am start-service \
    -n com.mitsubishielectric.ahu.appservice.ota.google/.UsbUpdateService \
    -a com.mitsubishielectric.ahu.appservice.ota.google.DEBUG_UNPACK \
    --es file /data/vendor/ota/usbupdateservice/test.swpkg --ei type 4
```
- `--es file` = package path (defaults to `UAHelper.getPathToTemporaryDwlFile()` if omitted).
- `--ei type` = unpack type: `1`=MCPU, `2`=VCPU, `4`=MCPU_VCPU.

### 4.3 Why `start-service` was needed (the key finding)
The `UsbUpdateService` is `android:singleUser="true"` and guarded by
`android:permission="…ACCESS_MELCO_APPS"`. The `BroadcastReceiver` is
**context-registered**, so it only exists *after* `onCreate()` runs. Early
`am broadcast` attempts were enqueued **before the service process existed**
(`Attempt to launch receivers … before boot completion`, and broadcasts
enqueued before the process started), so they were silently dropped
(`Enqueued broadcast … : 0` ≠ delivered). The `am start-service` path solves
this by **creating the service**, then routing through `onStartCommand`.

Note: `am start-service` requires the `ACCESS_MELCO_APPS` permission; from shell
this needs `adb root` (we saw `Permission Denial … uid=2000 requires
…ACCESS_MELCO_APPS` until root was used).

---

## 5. Step-by-step procedure we followed

1. Add `BHUVANESH_UNPACK` trace logs to `PackageUnpacker` and both `Unpacker` adapters.
2. Add the debug receiver + `runDebugUnpack()` + registration log to `UsbUpdateService`.
3. Add the `onStartCommand` `DEBUG_UNPACK` branch (start-service path).
4. Rebuild the `UsbUpdateService` APK, push to `/product/priv-app/UsbUpdateService/`, reboot.
5. Confirm `BHUVANESH_UNPACK DEBUG receiver registered …` appears in logcat (proves fresh APK).
6. `adb root`; push a package file to `/data/vendor/ota/usbupdateservice/test.swpkg`.
7. Trigger via `am start-service` (Section 4.2).
8. Watch `adb logcat | grep BHUVANESH_UNPACK`.

---

## 6. Results — how far we got (achievement level)

Across iterations the trace advanced exactly as the trigger and input were fixed:

| Iteration | Trigger reached? | Unpacker reached? | Stopped at | Cause |
|-----------|------------------|-------------------|------------|-------|
| 1 | No | No | broadcast dropped | service not alive / runtime receiver timing |
| 2 | **Yes** (`onStartCommand`) | **Yes** (`unpack() ENTRY`) | `loadJsonData` "failed to open file" | input file did not exist |
| 3 | Yes | Yes | `STEP1` done, then `NumberFormatException` | bad `Package_Version` in synthetic file |
| 4 (current build) | Yes | Yes | native `SecurePackageHandler.decryptPackage` → `KEY_NOT_INSTALLED` | SWDL key not provisioned on emulator |

**Final confirmed trace (iteration 4):** trigger → `unpack() ENTRY` → `STEP1 loadJsonData (738B)`
→ `STEP2 loadPackageHeader=true` → `STEP3 MCPU present` → `STEP4 VCPU present` →
`STEP5 extract()` → `extract() ENTRY` → `decryptMCPU() ENTRY` → offsets computed
(FWEK@194/16, payload@210/256) → `decrypt() ecu=MCPU` → KeyMaster reports
"SWDL key was not installed" → `SecurePackageHandler status=KEY_NOT_INSTALLED` →
`STEP6 extract()=false` → `unpack() EXIT, packageError=131073` (`0x020001`
= `ERROR_SECURITY_MCPU_DECRYPTION`). The native crypto path is fully reached and
executing; it stops only because the emulator KeyMaster has no SWDL key. On real
hardware with the key provisioned + a genuine signed package, the same call
produces `MCPU.dec` and continues to signature verification and `delta.mld` extraction.

**Achieved:** the debug trigger reliably starts the service and drives the
**genuine production entry point** `PackageUnpacker.unpack()`. We can now observe
the full pipeline in logcat: trigger → `unpack() ENTRY` → `loadJsonData` →
`loadPackageHeader` → MCPU/VCPU member detection → `extract()` → `decryptMCPU()`
→ offset computation → the native decrypt call.

**Hard stop (by design):** the final step,
`SecurePackageHandler.decryptPackage(...)` in `PackageUnpacker.decrypt()`, is the
native crypto boundary. It requires a **production-signed, FWEK-encrypted
package + real key material + secure element**, none of which exist on the
emulator. No synthetic/test package can pass it — this is the intended
signature-bypass protection, not a defect. To complete a real
decrypt+verify+unpack we need a genuine signed `.swpkg` on real/representative
hardware.

---

## 7. Risk / impact statement

- Production flow untouched: the `DEBUG_UNPACK` branch in `onStartCommand`
  returns before `initUsbUpdateService()`; the receiver is additive; the rest is
  logging.
- The trigger is developer-only (custom action, requires `adb root` due to the
  service permission).
- **Before release:** the `BHUVANESH_UNPACK` debug receiver, `onStartCommand`
  branch, and verbose logs should be removed or gated behind a debug build flag.

---

## 8. Files changed

- `UsbUpdateService/app/src/main/java/.../google/UsbUpdateService.java`
  — debug receiver, `runDebugUnpack()`, `onStartCommand` branch, registration log.
- `Common/library/src/main/java/.../unpacker/PackageUnpacker.java`
  — `BHUVANESH_UNPACK` trace logs (no logic change).
- `UsbUpdateService/.../google/unpack/Unpacker.java`,
  `WiFiUpdateService/.../rb/unpack/Unpacker.java`
  — one trace log each.
- Test artifact (not committed): synthetic `test.swpkg` (valid header, dummy payload)
  used only to advance the trace to the native decrypt boundary.
