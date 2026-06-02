# TBMFotaService Module Analysis

## Overview
The `TBMFotaService` module is split into two Gradle subprojects:
- `app`: Android library/service implementation for TBM firmware update orchestration.
- `tbmfotaserviceinterface`: A lightweight IPC interface module exposing update status callbacks.

The service coordinates the following responsibilities:
- Bind to extended framework services (ESM)
- Communicate with the CPU command service to query/update TBM update state
- Bind to Infotainment State Manager (ISM) and subscribe to system state and update events
- Coordinate OTA update approval with an arbiter service
- Send TBM update notifications and actions to the TBM HMI
- Notify app drawer/UI of ongoing update status via IPC callback

## Module Structure

### `settings.gradle`
- Includes `:app` and `:tbmfotaserviceinterface`.

### `tbmfotaserviceinterface`
- `src/main/java/com/mitsubishielectric/ahu/lib/tbmfotaserviceinterface/TbmFotaActualUpdateSts.java`
- `src/main/aidl/com/mitsubishielectric/ahu/lib/tbmfotaserviceinterface/ITbmFotaActualUpdateStsManager.aidl`
- `src/main/aidl/com/mitsubishielectric/ahu/lib/tbmfotaserviceinterface/ITbmFotaActualUpdateStsCallback.aidl`

### `app`
- Main service class: `TBMFotaService.java`
- Broadcast receiver: `TbmFotaUpdateActionReceiver.java`
- Supporting managers:
  - `TbmFotaCpuComManager.java`
  - `VehiclePowerManager.java`
  - `TbmFotaArbiterManager.java`
  - `IsmManager.java`
  - `VehicleCfgManager.java`
- Command wrappers: `TbmUpdateStatusRequest.java`, `TbmUpdateStatusNotification.java`, `TbmActionUpdateNotification.java`
- CPU command constants: `TbmFotaCpuCommands.java`
- Service constants: `TbmFotaServiceConstants.java`
- Callback interfaces: `ITbmFotaCpuComManager`, `ITbmFotaCpuComServiceListener`, `IIsmServiceListener`, `ITbmUpdateStatusChangeListener`

## `tbmfotaserviceinterface` Behavior

### `TbmFotaActualUpdateSts`
- Parcelable object containing a single boolean `available`.
- Used to propagate whether TBM update is currently active or not.

### AIDL Interfaces
- `ITbmFotaActualUpdateStsManager`
  - `registerAsyncConnection(ITbmFotaActualUpdateStsCallback tbmFotaCallback)`
  - `unRegisterAsyncConnection(ITbmFotaActualUpdateStsCallback tbmFotaCallback)`
- `ITbmFotaActualUpdateStsCallback`
  - `onTbmFotaActualUpdateStsChange(TbmFotaActualUpdateSts state)`

This interface module defines the IPC contract through which external clients can receive TBM update status changes.

## `app` Module Behavior

### `TBMFotaService`
This is the core Android `Service` for TBM OTA update support.

#### Lifecycle and initialization
- `onCreate()` initializes `VehicleCfgManager`.
- `onBind()` and `onStartCommand()` call `bindExtendedService(intent)` if not already bound.
- `bindExtendedService()` obtains the ESM binder from the start intent or binds to the ESM service directly.
- `registerToExtendedServiceManager(IBinder esmBinder)`:
  - Sets the ESM binder via `ExtSrvManager.setBinder(...)`
  - Retrieves vehicle config reader binder and initializes vehicle config access
  - Checks if the vehicle line config is applicable for TBM updates
  - If applicable, calls `init()`, `bindToIsm()`, `subscribeToVps()`

#### `init()`
Creates instances of:
- `VehiclePowerManager`
- `TbmFotaCpuComManager`
- `TbmFotaArbiterManager`
- `IsmManager`

Also binds to TBM HMI service and registers broadcast receivers:
- ESM reboot receiver
- FW service reboot receiver
- TBM update action receiver from HMI

### Broadcast and Messenger communication
- `sendBroadcastToTbmFotaHmi(int tbmUpdateStausCode)` builds and sends an intent to the HMI app:
  - Category: `CATEGORY_TBM_UPDATE_NOTIFICATION`
  - Action: `ACTION_TBM_UPDATE_NOTIFICATION`
  - Extras: `TBM_UPDATE_STATUS_CODE`, `TBM_ECALL_BUTTON_VALUE`, `TBM_UPDATE_USER_ON_OFF_STATE`
- When bound to TBM HMI, it also sends Messenger messages for system state, power button, and user on/off state.
- `TbmFotaUpdateActionReceiver` listens for HMI actions and forwards them to the CPU communication manager.

### Update status flow
- `notifyTbmData(int tbmUpdateStausCode)` is called when a TBM update status is received from the CPU command service.
- It performs three actions:
  1. `mIsmManager.setTbmStatusCode(...)`
  2. `checkTbmStatusForChangingAppDrawerIcons(...)`
  3. `sendUpdateStatusToArbiter(...)`

### App drawer / UI callback
- `checkTbmStatusForChangingAppDrawerIcons(int tbmUpdateStausCode)` maps status codes:
  - `TBM_UPDATE_START` → send `true` to callback
  - `TBM_UPDATE_END`, `TBM_UPDATE_FAIL`, `TBM_UPDATE_NOT_AVAIL` → send `false`
- `notifyTbmUpdateStsToAppDrawer(boolean sts)` invokes `mTbmFotaActualUpdateStsCallback.onTbmFotaActualUpdateStsChange(...)`
- This is the only place where the interface module callback is used to provide ongoing status to clients.

### Vehicle configuration checks
- `checkVehicleLineConfigIsApplicableForTbm()` reads vehicle configuration values via `VehicleCfgManager`:
  - `PROXI_VEHICLE_LINE_CONFIGURATION`
  - `VEHICLE_MODEL`
  - `PROXY_MODEL_YEAR`
- It compares these against external `VsmServiceInterfaceConstants` values to decide if TBM updates should run on this vehicle.
- If not applicable, the service unbinds from ESM and stops initialization.

### Power management integration
- `VehiclePowerManager` binds to the vehicle power service and subscribes an `IVehiclePowerServiceListener`.
- The listener responds to lifecycle events:
  - `onAppStart()` → initial CPU request, notify start complete, start ISM subscriptions
  - `onAppRestart()` → same as start
  - `onAppResume()` → same as start, with resume complete
  - `onAppStop()` → stop processing complete

### Infotainment state management via `IsmManager`
`IsmManager` binds to the infotainment state manager and subscribes to:
- availability changes (`IAvailabilityChangeListener`)
- system state transitions for TBM FOTA update (`ISystemStateTransitionTbmFotaUpdateListener`)
- TBM update status updates (`ITbmFotaUpdateStatusListener`)
- system state changes (`ISystemStateListener`, retried until known)

Important behavior in `IsmManager`:
- On `ESystemState.FULL`, it triggers a CPU update request: `mTbmFotaCpuComManager.initialRequest()`
- On `ESystemState.TIMED`, it calls back into `TBMFotaService.onIsmSystemStateTimed(...)`
- Handles user on/off availability changes to show/hide popup or notify status changes
- Starts a 30-minute timer for TIMED states and invokes `onTimeOutTimerExpired()` if it expires
- For one-minute user interaction expiry, it tells ISM service the status changed via `tbmFotaUpdateStatusChanged(...)`

### Arbiter coordination via `TbmFotaArbiterManager`
This manager binds to `ArbiterServiceManager` and requests update approval.

Key methods:
- `requestInstallation()` calls `installationRequest(UPDATE_TYPE.OTA_TBM, listener)`
- `requestInstallationFinished()` calls `installationFinished(UPDATE_TYPE.OTA_TBM)`
- Tracks approval state via `onRequestGranted()` / `onRequestDenied()` callbacks

`TBMFotaService` uses the arbiter for these cases:
- If TBM update status is `TBM_UPDATE_AVAIL` or `TBM_UPDATE_FORCED`, request installation approval
- If TBM update status is `TBM_UPDATE_SILENT`, call `requestInstallationFinished()`
- On timeout or HMI user action, call `requestInstallationFinished()`

### CPU communication via `TbmFotaCpuComManager`
This manager wraps `CpuComManager` and handles command subscription and sending.

Important behavior:
- `setCpuComBinder(IBinder service)` binds the CPU service and subscribes to commands and errors
- `initialRequest()` sends a `TbmUpdateStatusRequest` command to query TBM update status
- `sendCommand(CpuCommand cmd)` forwards arbitrary commands through `CpuComManager`
- `onReceiveCmd(CpuCommand cpuCommand)` listens for CPU commands:
  - Only handles `CMD_TBM_UPDATE_STATUS_REQUEST` / `SUBCMD_TBM_FOTA_UPDATE_STATUS_REQUEST_RESULT`
  - Extracts the first byte from `data` and calls `mTbmFotaCpuComServiceListener.notifyTbmData(...)`

Command classes:
- `TbmUpdateStatusRequest` → command `0x40`, subcommand `0x07`
- `TbmUpdateStatusNotification` → command `0x40`, subcommand `0x87`
- `TbmActionUpdateNotification` → command `0x40`, subcommand `0x08`, with a one-byte payload for the user action

### HMI action handling
`TbmFotaUpdateActionReceiver` listens for HMI broadcast actions from the FOTA UI and does one of the following:
- `ACTION_TBM_UPDATE` → send `TbmActionUpdateNotification` to VCPU and call `onTbmUpdateStatusChanged()` on the service
- `ACTION_TBM_UPDATE_NOT_AVAILABLE` → call `onTbmUpdateStatusChanged()`
- `ACTION_TBM_UPDATE_ONGOING` → call `onTbmUpdateStatusStart()`
- `ACTION_TBM_UPDATE_TIMEOUT` → call `onIsmOneMinuteExpiredInTbmUpdate(AVAILABLE_USER_OFF)`

### External services and APIs used
The module does not perform local file I/O. It interacts with platform services and external frameworks:
- `com.mitsubishielectric.ahu.efw.lib.extendedservicemanager.ExtSrvManager`
- `com.mitsubishielectric.ahu.efw.lib.cpucomservice.CpuComManager`
- `com.mitsubishielectric.ahu.efw.lib.vehiclepwrmgr.VehiclePowerServiceManager`
- `com.mitsubishielectric.ahu.efw.lib.vehicleconfigservice.VehicleConfigManager`
- `com.mitsubishielectric.ahu.appservice.infotainmentstatemanager.IInfotainmentStateManagerService`
- `com.mitsubishielectric.ahu.appservice.ota.arbiter.ArbiterServiceManager`
- TBM HMI service: `com.mitsubishielectric.ahu.app.tbmfotahmi.TbmFotaHmiUpdateService`

## Summary of responsibilities

### What the module does
- Orchestrates TBM firmware update state and UI notifications
- Observes system state and vehicle power state
- Requests installation approval through an arbiter
- Sends commands to VCPU and consumes responses
- Broadcasts status updates to the TBM HMI
- Provides a callback interface for app UI components to show update state

### What the module does not do
- It does not read or write ordinary app files from disk.
- It does not implement the actual update delivery mechanism itself.
- It does not directly manage firmware payloads; it only monitors and notifies state.

## Key call graph

1. `onStartCommand` / `onBind` → `bindExtendedService`
2. `bindExtendedService` → `bindEsmService` or `initiateExtSrv`
3. `registerToExtendedServiceManager` → `init()` + `bindToIsm()` + `subscribeToVps()`
4. `TbmFotaCpuComManager.setCpuComBinder` → subscribe to CPU commands
5. `TbmFotaCpuComManager.initialRequest` → send `TbmUpdateStatusRequest`
6. `onReceiveCmd` from CPU socket → `notifyTbmData(int)`
7. `notifyTbmData` → update ISM manager, update app drawer callback, arbiter/HMI flow
8. `TbmFotaUpdateActionReceiver.onReceive` → `sendCommand(new TbmActionUpdateNotification(...))`
9. `IsmManager` callbacks → `TBMFotaService.onIsm*...` methods
