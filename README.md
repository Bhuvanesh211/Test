# TBMFotaService Startup Flow and Major API Interactions

## Overview
This document describes the startup flow and the major API calls/interactions in the `TBMFotaService` module.

The service acts as a bridge between the TBM update status from the vehicle CPU/VCPU, the infotainment state manager (ISM), the TBM FOTA HMI, and external clients that need actual update status.

---

## 1. Service Startup Flow

### 1.1 `onCreate()`
- Called when the Android service is created.
- Actions:
  - `VehicleCfgManager.getInstance()` is initialized.
- Note: at this point, framework services are not yet bound.

### 1.2 `onBind(Intent intent)` / `onStartCommand(Intent intent, ...)`
- Both methods call `bindExtendedService(intent)` if `mBindStatus` is false.
- This ensures the service starts correctly whether it is bound or started.

### 1.3 `bindExtendedService(Intent intent)`
- If the incoming intent contains `Const.ESM_SERVICE` binder:
  - the service was started by the Extended Service Manager (ESM)
  - it calls `initiateExtSrv(intent.getExtras().getBinder(ESM_SERVICE))`
- Otherwise:
  - it calls `bindEsmService()` to bind explicitly to ESM.

### 1.4 `bindEsmService()`
- Builds an Intent for `Const.ESM_PACKAGE` / `Const.ESM_SERVICE`.
- Binds using `mSyncConnection` and `Context.BIND_AUTO_CREATE`.

### 1.5 `mSyncConnection.onServiceConnected()`
- When ESM binds, `registerToExtendedServiceManager(service)` is invoked.

### 1.6 `registerToExtendedServiceManager(IBinder esmBinder)`
- Sets up the `ExtSrvManager` singleton.
- Retrieves the vehicle config reader binder and calls `initiateVehicleConfigReaderService(...)`.
- Calls `checkVehicleLineConfigIsApplicableForTbm()`.
- If the vehicle line is compatible:
  - calls `init()`
  - calls `bindToIsm()`
  - calls `subscribeToVps()`
- If not compatible:
  - unbinds from ESM if needed and stops further init.

### 1.7 `init()`
- Instantiates core managers:
  - `VehiclePowerManager`
  - `TbmFotaCpuComManager`
  - `TbmFotaArbiterManager`
  - `IsmManager`
- Binds to the TBM FOTA HMI service via `mTbmFotaHmiServiceConnection`.
- Registers broadcast receivers for:
  - ESM reboot
  - framework service reboot
  - HMI action notifications

### 1.8 `bindToIsm()`
- Calls `mIsmManager.bindService(getApplicationContext())`.
- Connects the service to the infotainment state manager.

### 1.9 `subscribeToVps()`
- Retrieves the CPU COM binder from `mExtSrvManager`.
- Passes that binder to `mFotaCpuComManager`.
- Initializes `mVehiclePowerManager`.
- Subscribes to vehicle power callbacks via `mVehiclePowerServiceListener`.

---

## 2. Major API Calls and Interactions

### 2.1 TBM update status entry point
- `notifyTbmData(int tbmUpdateStausCode)`
- Called when the TBM update status is received from VCPU.
- Actions:
  - `mIsmManager.setTbmStatusCode(tbmUpdateStausCode)`
  - stores `mTbmUpdateStatus`
  - logs the status
  - calls `checkTbmStatusForChangingAppDrawerIcons(tbmUpdateStausCode)`
  - calls `sendUpdateStatusToArbiter(tbmUpdateStausCode)`

### 2.2 Arbiter interaction
- `sendUpdateStatusToArbiter(int tbmUpdateStausCode)` handles update flow decisions.
- Behavior:
  - if status is `TBM_UPDATE_AVAIL` or `TBM_UPDATE_FORCED`:
    - `mTbmFotaArbiterManager.requestInstallation()`
  - else if status is `TBM_UPDATE_SILENT`:
    - `mTbmFotaArbiterManager.requestInstallationFinished()`
  - else:
    - `sendBroadcastToTbmFotaHmi(tbmUpdateStausCode)`

### 2.3 HMI broadcast notification
- `sendBroadcastToTbmFotaHmi(int tbmUpdateStausCode)`
- Constructs an Intent to `TBM_FOTA_HMI_PACKAGE_NAME`.
- Adds category `CATEGORY_TBM_UPDATE_NOTIFICATION` and action `ACTION_TBM_UPDATE_NOTIFICATION`.
- Adds extras:
  - `TBM_UPDATE_STATUS_CODE`
  - `TBM_ECALL_BUTTON_VALUE`
  - `TBM_UPDATE_USER_ON_OFF_STATE`
- If the update is `TBM_UPDATE_NOT_AVAIL`, it calls `onTbmUpdateStatusChanged()`.
- If the user is off and the status is `TBM_UPDATE_END`, it calls `mIsmManager.setTbmFotaDisplayStatus(ETbmFotaPopupStatus.HIDE)`.
- Finally it sends the broadcast.

### 2.4 User On/Off and popup control
- `getUserOnOffStateInTbmFotaService(Intent mIntent)`:
  - calls `mIsmManager.getUserOnOffState()`
  - adds the result to the Intent as `TBM_UPDATE_USER_ON_OFF_STATE`
  - returns the boolean
- `setBackLight(boolean isBackLightOn)`:
  - if `mTbmUpdateStatus` is start/fail/not available,
  - uses `mIsmManager.setTbmFotaDisplayStatus(SHOW/HIDE)`.

### 2.5 External IPC callback
- Service exposes `ITbmFotaActualUpdateStsManager` through `mTbmFotaServiceBinder`.
- External clients can call:
  - `registerAsyncConnection(ITbmFotaActualUpdateStsCallback tbmFotaCallback)`
  - `unRegisterAsyncConnection(ITbmFotaActualUpdateStsCallback tbmFotaCallback)`
- When registered, the service updates app drawer status via `notifyTbmUpdateStsToAppDrawer(boolean sts)`.
- This method calls `mTbmFotaActualUpdateStsCallback.onTbmFotaActualUpdateStsChange(new TbmFotaActualUpdateSts(sts))`.

### 2.6 Vehicle power lifecycle callbacks
- `mVehiclePowerServiceListener` handles:
  - `onAppStart()`
  - `onAppRestart()`
  - `onAppStop()`
  - `onAppResume()`
- For start/restart/resume:
  - calls `mFotaCpuComManager.initialRequest()`
  - notifies start/restart/resume completion through `mVehiclePowerManager`
  - calls `mIsmManager.startSubscribingToIsmSystemState()`
- For stop:
  - calls `mVehiclePowerManager.stopProcessingComplete()`

### 2.7 ISM system state and timed state handling
- `onIsmSystemStateTimed(int tbmUpdateStatus)`:
  - if `TBM_UPDATE_SILENT`:
    - if user off → `requestInstallationFinished()`
    - else → broadcast `TBM_UPDATE_SILENT`
  - otherwise, check arbiter status and potentially broadcast current status.
- `onIsmServiceConnected()`:
  - if current status is `TBM_UPDATE_AVAIL`, `TBM_UPDATE_FORCED`, or `TBM_UPDATE_SILENT`:
    - call `mTbmFotaArbiterManager.requestInstallation()`
- `onIsmDismissPopup()`:
  - sends a broadcast to HMI with current `mTbmUpdateStatus`
- `onTbmUpdateStatusChanged()`:
  - invoked after popup actions like OK/CLOSE
  - requests installation finished through the arbiter
- `onTbmUpdateStatusStart()`:
  - if user off, hides backlight
- `onEsystemStateChanged(ESystemState eSystemState)`:
  - sends an ESystemState message to HMI if bound
  - either starts or cancels the 30-minute timer
- `onPowerButtonPressed()` and `onUserOnOffStatusChanged(boolean)`:
  - send HMI messages via `Messenger` if bound

---

## 3. Shutdown and Cleanup

### `onDestroy()`
- Unbinds the TBM FOTA HMI service if connected.
- Unbinds the ESM sync connection if connected.
- Uses `try/catch` around unbind calls to avoid crash when already disconnected.

---

## 4. Summary

### Main startup path
1. `onCreate()`
2. `onBind()`/`onStartCommand()` → `bindExtendedService()`
3. `bindExtendedService()` → `bindEsmService()` or use existing ESM binder
4. `mSyncConnection.onServiceConnected()` → `registerToExtendedServiceManager()`
5. `registerToExtendedServiceManager()` → `init()` + `bindToIsm()` + `subscribeToVps()`

### Main runtime paths
- TBM status from VCPU → `notifyTbmData()`
- status handling → arbiter or HMI broadcast
- ISM timed/state callbacks → HMI display / arbiter decisions
- app drawer status → external callback via `ITbmFotaActualUpdateStsCallback`
