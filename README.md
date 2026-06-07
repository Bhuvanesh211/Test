# TBMFotaService Code Explanation

## Short Summary

`TBMFotaService` is a privileged Android system service that coordinates TBM OTA/FOTA update status between the vehicle CPU side, the infotainment state manager, the OTA arbiter, the TBM FOTA HMI app, vehicle power lifecycle, and external clients such as the app drawer.

Its main job is not to download the update package. Its job is to:

1. Connect to platform/framework services through `ExtendedServiceManager`.
2. Subscribe to CPU/VCPU TBM update-status commands.
3. Ask the CPU for the current TBM update status on power/start/resume events.
4. Receive TBM update status from CPUCom/VCPU.
5. Decide whether Arbiter approval is required.
6. Notify TBM FOTA HMI so the correct popup/screen is shown.
7. Receive user actions from HMI and forward those actions back to VCPU.
8. Notify external AIDL clients when an actual TBM update is ongoing or finished.
9. Coordinate with ISM for system state, user-on/off state, popup visibility, power button events, and timers.

## Module Structure

The ZIP contains two important Android modules:

| Module | Purpose |
|---|---|
| `app` | The actual privileged Android service implementation. |
| `tbmfotaserviceinterface` | AIDL/Parcelable interface used by external clients to register callbacks for actual TBM update status. |

Important implementation classes:

| Class | Responsibility |
|---|---|
| `TBMFotaService` | Main Android service and central coordinator. |
| `TbmFotaCpuComManager` | Wrapper around `CpuComManager`; sends/receives VCPU commands. |
| `TbmFotaUpdateActionReceiver` | Receives HMI broadcasts for user actions and forwards them to service/CPUCom. |
| `TbmFotaArbiterManager` | Calls OTA Arbiter to request or finish TBM installation approval. |
| `IsmManager` | Binds to Infotainment State Manager and handles system/user state callbacks. |
| `VehiclePowerManager` | Subscribes to vehicle power lifecycle callbacks. |
| `VehicleCfgManager` | Wrapper around Vehicle Config Service reader/writer binders. |
| `TbmUpdateStatusRequest` | CPU command `0x40/0x07`, asks VCPU for current TBM update status. |
| `TbmUpdateStatusNotification` | CPU command `0x40/0x87`, subscription target for TBM update-status responses. |
| `TbmActionUpdateNotification` | CPU command `0x40/0x08`, sends HMI user action back to VCPU. |

## How The Service Is Invoked

### Android/Framework Invocation

The service is declared in `app/src/main/AndroidManifest.xml` as:

- `android:persistent="true"`: it is intended to run as a persistent system-level app.
- `android:exported="true"`: other approved components can bind/start it.
- `android:singleUser="true"`: one service instance is shared for the system user.
- `android:permission="com.mitsubishielectric.ahu.efw.common.permission.ACCESS_MELCO_APPS"`: callers need the MELCO app permission.

There is no launcher activity and no manifest receiver that starts it directly. The service is invoked by Android/framework code or another authorized app/service through:

- `startService(...)`, which calls `onStartCommand(...)`.
- `bindService(...)`, which calls `onBind(...)`.
- A framework/ESM start intent that may already contain an `ESM_SERVICE` binder.

Both `onStartCommand(...)` and `onBind(...)` call `bindExtendedService(intent)` if the service is not already bound to ESM.

### External Client Invocation Through AIDL

The service exposes `ITbmFotaActualUpdateStsManager` through `onBind(...)`. External clients can:

- `registerAsyncConnection(ITbmFotaActualUpdateStsCallback callback)`
- `unRegisterAsyncConnection(ITbmFotaActualUpdateStsCallback callback)`

After a callback is registered, TBMFotaService uses:

`callback.onTbmFotaActualUpdateStsChange(new TbmFotaActualUpdateSts(sts))`

to tell the external client whether a real TBM update is ongoing.

This is mainly used for external UI/app components such as app drawer icon-state updates.

## Complete Startup Flow

### 1. Service Created

Android calls:

`TBMFotaService.onCreate()`

The service initializes:

`mVehicleCfgManager = VehicleCfgManager.getInstance()`

At this point, the service has not yet completed framework service binding.

### 2. Service Started Or Bound

Android then calls either:

- `onStartCommand(Intent intent, int flags, int startId)`
- `onBind(Intent intent)`

Both methods check `mBindStatus`. If not already bound, they call:

`bindExtendedService(intent)`

`onStartCommand(...)` returns `Service.START_STICKY`, so Android should try to recreate/restart the service if the process is killed.

### 3. ExtendedServiceManager Binding

`bindExtendedService(intent)` has two paths:

| Case | Behavior |
|---|---|
| Intent contains `Const.ESM_SERVICE` binder | The service was started by ESM. It calls `initiateExtSrv(...)`. |
| Intent does not contain binder | The service explicitly binds to ESM using `bindEsmService()`. |

`bindEsmService()` builds an intent to:

`Const.ESM_PACKAGE / Const.ESM_SERVICE`

and binds with `Context.BIND_AUTO_CREATE`.

When the ESM connection completes, `mSyncConnection.onServiceConnected(...)` calls:

`registerToExtendedServiceManager(service)`

### 4. Register With ESM And Load Framework Binders

`registerToExtendedServiceManager(IBinder esmBinder)` is the key initialization gate:

1. Sets `mBindStatus = true`.
2. Calls `ExtSrvManager.setBinder(esmBinder)`.
3. Gets `ExtSrvManager.getInstance()`.
4. Reads the Vehicle Config Reader binder:
   `mExtSrvManager.getService(Const.VEHICLE_CONFIG_READER_SERVICE)`
5. Passes it to:
   `initiateVehicleConfigReaderService(vehicleConfigReaderBinder)`
6. Checks whether TBM FOTA is applicable for this vehicle:
   `checkVehicleLineConfigIsApplicableForTbm()`

If the vehicle is applicable, it calls:

1. `init()`
2. `bindToIsm()`
3. `subscribeToVps()`

If the vehicle is not applicable, it unbinds from ESM when appropriate and stops further initialization.

### 5. Vehicle Configuration Gate

`checkVehicleLineConfigIsApplicableForTbm()` reads these config values:

| Config Key | Meaning |
|---|---|
| `PROXI_VEHICLE_LINE_CONFIGURATION` | Vehicle line. |
| `VEHICLE_MODEL` | Vehicle model string. |
| `PROXI_MODEL_YEAR` | Model year. |

The service proceeds only when:

- vehicle line is not `WL`, and
- vehicle model/year is not `DT` with model year `>= 25`.

This means TBM FOTA support is enabled only for applicable vehicle configurations.

### 6. Internal Manager Initialization

`init()` creates and binds the service's main collaborators:

```text
VehiclePowerManager
TbmFotaCpuComManager
TbmFotaArbiterManager
IsmManager
```

It also binds to TBM FOTA HMI:

```text
Package: com.mitsubishielectric.ahu.app.tbmfotahmi
Service: com.mitsubishielectric.ahu.app.tbmfotahmi.TbmFotaHmiUpdateService
```

When HMI binding succeeds, the service stores a `Messenger` so it can send direct messages to HMI for:

- system state changes
- power button pressed
- user-on/off changes

`init()` also registers three receiver groups:

| Receiver | Why |
|---|---|
| ESM reboot receiver | Refreshes ESM binder when ESM restarts. |
| Framework service reboot receiver | Refreshes Vehicle Config Reader binder. |
| HMI action receiver | Receives user actions from TBM FOTA HMI popups. |

### 7. Bind To ISM

`bindToIsm()` calls:

`mIsmManager.bindService(getApplicationContext())`

`IsmManager` binds to:

```text
Package: com.mitsubishielectric.ahu.appservice.infotainmentstatemanager
Service: com.mitsubishielectric.ahu.appservice.infotainmentstatemanager.InfotainmentStateService
```

After connection, `IsmManager` subscribes to:

- availability/user-on-off changes
- system state transitions
- TBM FOTA update status changes
- system state callbacks

Then it calls back into:

`TBMFotaService.onIsmServiceConnected()`

If the current TBM status is available, forced, or silent, the service requests Arbiter installation approval.

### 8. Subscribe To Vehicle Power And CPUCom

`subscribeToVps()` does two major things:

1. Gets the CPUCom binder from ESM:
   `mExtSrvManager.getService(Const.CPU_COM_SERVICE)`
2. Passes it to:
   `mFotaCpuComManager.setCpuComBinder(cpuComServiceBinder)`

Inside `TbmFotaCpuComManager.setCpuComBinder(...)`:

1. Calls `CpuComManager.setBinder(service)`.
2. Subscribes to `TbmUpdateStatusNotification`.
3. Subscribes to CPUCom error callbacks.

`subscribeToVps()` also initializes `VehiclePowerManager` and subscribes its `IVehiclePowerServiceListener`.

## Vehicle Power Lifecycle Flow

Vehicle Power Service can call these callbacks:

| Callback | What TBMFotaService does |
|---|---|
| `onAppStart()` | Sends initial TBM update status request, notifies start complete, starts ISM system-state subscription. |
| `onAppRestart()` | Sends initial TBM update status request, notifies restart complete, starts ISM subscription. |
| `onAppResume()` | Sends initial TBM update status request, notifies resume complete, starts ISM subscription. |
| `onAppStop()` | Notifies stop processing complete. |

For start/restart/resume, the important CPU call is:

`TbmFotaCpuComManager.initialRequest()`

That sends:

`new TbmUpdateStatusRequest()`

to VCPU.

## CPU/VCPU Communication Flow

### Commands

| Purpose | Command | Sub-command | Class |
|---|---:|---:|---|
| Request current TBM update status | `0x40` | `0x07` | `TbmUpdateStatusRequest` |
| Receive TBM update status result | `0x40` | `0x87` | `TbmUpdateStatusNotification` |
| Send HMI user action to VCPU | `0x40` | `0x08` | `TbmActionUpdateNotification` |

### Status Codes

| Code | Constant | Meaning |
|---:|---|---|
| `0` | `TBM_UPDATE_NOT_AVAIL` | No TBM update available. |
| `1` | `TBM_UPDATE_AVAIL` | Update available. |
| `2` | `TBM_UPDATE_START` | Update started/ongoing. |
| `3` | `TBM_UPDATE_END` | Update completed/end. |
| `4` | `TBM_UPDATE_FORCED` | Forced update available. |
| `5` | `TBM_UPDATE_FAIL` | Update failed. |
| `6` | `TBM_UPDATE_SILENT` | Silent update. |

### CPU Receive Path

When VCPU/CPUCom sends a command to the service:

`TbmFotaCpuComManager.mCpuComServiceListener.onReceiveCmd(CpuCommand cpuCommand)`

The manager checks:

```text
command == 0x40
subCommand == 0x87
```

Then it reads:

`data[0]`

and calls:

`mTbmFotaCpuComServiceListener.notifyTbmData(data[0])`

Because `TBMFotaService` implements `ITbmFotaCpuComServiceListener`, this lands in:

`TBMFotaService.notifyTbmData(int tbmUpdateStausCode)`

## Main Update Status Handling Flow

When `notifyTbmData(status)` is called:

1. If `mIsmManager` exists, update ISM manager's stored TBM status:
   `mIsmManager.setTbmStatusCode(status)`
2. Store it in:
   `mTbmUpdateStatus`
3. Notify App Drawer/external AIDL clients if needed:
   `checkTbmStatusForChangingAppDrawerIcons(status)`
4. Route update status through Arbiter/HMI:
   `sendUpdateStatusToArbiter(status)`

### App Drawer / External Client Update

`checkTbmStatusForChangingAppDrawerIcons(status)` maps status to a boolean:

| Status | Callback Value |
|---|---|
| `TBM_UPDATE_START` | `true` |
| `TBM_UPDATE_END`, `TBM_UPDATE_FAIL`, `TBM_UPDATE_NOT_AVAIL` | `false` |
| Other statuses | No app-drawer callback. |

If a client registered through AIDL, the service calls:

`onTbmFotaActualUpdateStsChange(new TbmFotaActualUpdateSts(sts))`

### Arbiter/HMI Routing

`sendUpdateStatusToArbiter(status)` decides the next step:

| TBM Status | Next Step |
|---|---|
| `TBM_UPDATE_AVAIL` | Ask Arbiter: `requestInstallation()` |
| `TBM_UPDATE_FORCED` | Ask Arbiter: `requestInstallation()` |
| `TBM_UPDATE_SILENT` | Report finish to Arbiter: `requestInstallationFinished()` |
| Any other status | Broadcast status to TBM FOTA HMI. |

Important point for explanation:

Available/forced statuses are not immediately broadcast as normal status. They first go through OTA Arbiter because another OTA update may already be in progress or higher priority. Arbiter decides whether TBM OTA can proceed.

## Arbiter Flow

`TbmFotaArbiterManager` connects to:

`ArbiterServiceManager`

When TBMFotaService calls:

`requestInstallation()`

the manager calls:

`installationRequest(UPDATE_TYPE.OTA_TBM, mIInstallationRequestApprovalListener)`

Arbiter replies asynchronously through:

- `onRequestGranted()`
- `onRequestDenied()`

The manager stores the state as:

- `ON_REQUEST_GRANTED`
- `ON_REQUEST_DENIED`
- `ON_REQUEST_NONE`

When the user finishes, timeout occurs, or the service otherwise concludes a TBM popup/update flow, the service calls:

`requestInstallationFinished()`

which calls:

`installationFinished(UPDATE_TYPE.OTA_TBM)`

This releases Arbiter coordination for TBM OTA.

## HMI Notification Flow

When TBMFotaService decides HMI must be notified, it calls:

`sendBroadcastToTbmFotaHmi(status)`

It builds a broadcast intent with:

```text
Package: com.mitsubishielectric.ahu.app.tbmfotahmi
Category: com.mitsubishielectric.ahu.appservice.tbmfotaservice.category.TBM_UPDATE
Action: com.mitsubishielectric.ahu.appservice.tbmfotaservice.action.TBM_UPDATE
```

Extras:

| Extra | Meaning |
|---|---|
| `TBM_UPDATE_STATUS_CODE` | Current TBM status code. |
| `TBM_ECALL_BUTTON_VALUE` | ECall button variant, read from Vehicle Config for selected statuses. |
| `TBM_UPDATE_USER_ON_OFF_STATE` | Whether user-on/off state is available. |

Special handling:

- If status is `TBM_UPDATE_NOT_AVAIL`, it calls `onTbmUpdateStatusChanged()` to finish Arbiter state.
- If user is off and status is `TBM_UPDATE_END`, it calls `onTbmUpdateStatusChanged()` and hides popup through ISM.

## HMI Action Flow Back To VCPU

HMI sends user actions as broadcasts. `TbmFotaUpdateActionReceiver` listens for:

| Action | Behavior |
|---|---|
| `ACTION_TBM_UPDATE` | Reads `EXTRA_TBM_UPDATE_ACTION`, calls `onTbmUpdateStatusChanged()`, sends `TbmActionUpdateNotification(action)` to VCPU. |
| `ACTION_TBM_UPDATE_NOT_AVAILABLE` | Calls `onTbmUpdateStatusChanged()`; no CPU action command. |
| `ACTION_TBM_UPDATE_ONGOING` | Calls `onTbmUpdateStatusStart()`. |
| `ACTION_TBM_UPDATE_TIMEOUT` | Calls `onIsmOneMinuteExpiredInTbmUpdate(AVAILABLE_USER_OFF)`. |

The main case is `ACTION_TBM_UPDATE`:

1. HMI user presses an action such as OK/update/later/close.
2. Receiver extracts the action value.
3. Service reports Arbiter finished through `onTbmUpdateStatusChanged()`.
4. Receiver sends `TbmActionUpdateNotification(updateAction)` to VCPU.
5. VCPU receives command `0x40/0x08` with one byte of action data.

## ISM Flow

`IsmManager` is responsible for infotainment state, user-on/off status, and popup display status.

### ISM Connection

On successful ISM bind:

1. Store `IInfotainmentStateManagerService`.
2. Subscribe to availability.
3. Subscribe to system-state transition for TBM FOTA update.
4. Subscribe to TBM FOTA update status.
5. Read current `UserOnOff` availability.
6. Notify service through `onIsmServiceConnected()`.

### System State Changes

When ISM reports `onSystemStateChange(ESystemState state)`:

| State | Behavior |
|---|---|
| `FULL` | Notify HMI with system state and send a new initial CPU request. Also cancel timeout timer in service callback. |
| `TIMED` + status available/forced/silent | Notify TBMFotaService via `onIsmSystemStateTimed(status)`. |
| Other states | Log as not timed. |

`TBMFotaService.onEsystemStateChanged(...)` also forwards system state to HMI through `Messenger` if HMI service is bound.

### Timed State Handling

When the system enters TIMED state:

- For silent status:
  - If user is off, service calls `requestInstallationFinished()`.
  - If user is on, service broadcasts silent status to HMI.
- For other available/forced states:
  - Service checks Arbiter request result and approver state.
  - If Arbiter allows or is not denied, service broadcasts to HMI.

### Thirty-Minute Timer

In non-FULL states, TBMFotaService asks ISM manager to start a 30-minute timer.

If the timer expires and the system is still not FULL and TBM FOTA status is not `NOT_AVAILABLE`, `IsmManager` calls:

`TBMFotaService.onTimeOutTimerExpired()`

The service then reports:

`mTbmFotaArbiterManager.requestInstallationFinished()`

### User On/Off Changes

When ISM availability changes for `UserOnOff`:

- Service sends a `USER_ON_OFF_CHANGED` Messenger message to HMI.
- If user turns on after start/fail while previously unavailable, service dismisses/rebroadcasts popup.
- If user turns off, service updates HMI and stores unavailable state.

## External Services Used And Why

| External Service/API | Used By | Why It Is Needed |
|---|---|---|
| `ExtSrvManager` / Extended Service Manager | `TBMFotaService`, `VehiclePowerManager` | Gateway to framework binders such as CPUCom, Vehicle Config Reader, Vehicle Power. |
| `CpuComManager` / CPUCom Service | `TbmFotaCpuComManager` | Sends TBM commands to VCPU and receives TBM update-status responses. |
| VCPU / CPU side | Through CPUCom commands | Source of actual TBM update status and receiver of HMI user actions. |
| `VehicleConfigManager` | `VehicleCfgManager`, `TBMFotaService` | Reads vehicle line/model/year and ECall button variant. Controls applicability and HMI extras. |
| `VehiclePowerServiceManager` | `VehiclePowerManager` | Notifies service of start/restart/resume/stop lifecycle and requires completion callbacks. |
| `IInfotainmentStateManagerService` / ISM | `IsmManager` | Provides system state, user-on/off availability, TBM popup display APIs, power button callback, and update-status callback. |
| `ArbiterServiceManager` | `TbmFotaArbiterManager` | Coordinates whether TBM OTA can proceed with other OTA/update flows. |
| TBM FOTA HMI service | `TBMFotaService` | Bound with `Messenger` for system/user/power events. |
| TBM FOTA HMI app broadcast receiver | `TBMFotaService` | Receives update status broadcasts and shows TBM update UI. |
| App Drawer or external AIDL clients | AIDL interface module | Gets boolean actual update status to change UI/icon state. |
| `VsmServiceInterfaceConstants` | `TBMFotaService` | Vehicle-line/model/year constants used for applicability rules. |
| `LogUtility` / `TraceLog` | All modules | Logging and trace diagnostics. |

## End-To-End Example: Update Available

1. Android starts or binds `TBMFotaService`.
2. Service gets ESM binder.
3. Service gets Vehicle Config Reader binder.
4. Service checks vehicle line/model/year.
5. If applicable, service initializes managers.
6. Service binds TBM HMI, ISM, CPUCom, Vehicle Power, and Arbiter.
7. Vehicle Power calls `onAppStart()` or `onAppResume()`.
8. Service sends CPU command `0x40/0x07` to ask for TBM update status.
9. VCPU replies through CPUCom using command `0x40/0x87`, data byte `1`.
10. CPUCom manager calls `TBMFotaService.notifyTbmData(1)`.
11. Service stores `TBM_UPDATE_AVAIL`.
12. Service does not mark app drawer ongoing yet because ongoing starts only at status `2`.
13. Service calls Arbiter `installationRequest(OTA_TBM, listener)`.
14. Arbiter grants the request.
15. When system state and user-on/off conditions allow, service broadcasts update-available status to TBM HMI.
16. HMI shows popup.
17. User chooses an action.
18. HMI broadcasts `ACTION_TBM_UPDATE` with action value.
19. `TbmFotaUpdateActionReceiver` receives it.
20. Service calls `requestInstallationFinished()` to release Arbiter.
21. Receiver sends CPU command `0x40/0x08` with action byte to VCPU.
22. VCPU proceeds according to action and later sends updated status such as start/end/fail.

## End-To-End Example: Update Started

1. VCPU sends status `2` through CPUCom.
2. Service receives `notifyTbmData(2)`.
3. Service calls `checkTbmStatusForChangingAppDrawerIcons(2)`.
4. External callback receives `TbmFotaActualUpdateSts(true)`.
5. Because status is not available/forced/silent, service broadcasts status directly to TBM FOTA HMI.
6. HMI shows ongoing update state.

## End-To-End Example: Update End Or Fail

1. VCPU sends status `3` or `5`.
2. Service receives `notifyTbmData(status)`.
3. App drawer callback receives `TbmFotaActualUpdateSts(false)`.
4. Service broadcasts status to HMI.
5. If user is off and status is end, service hides the popup through ISM.
6. If user action or timeout flow requires it, service reports Arbiter installation finished.

## End / Teardown Flow

When Android destroys the service:

`TBMFotaService.onDestroy()`

The service attempts to unbind:

- `mTbmFotaHmiServiceConnection`
- `mSyncConnection`

The code catches `IllegalArgumentException` to avoid Android 14 crashes if a connection was already unbound or never registered.

One thing to note: `onDestroy()` does not explicitly unregister all dynamically registered broadcast receivers in this code. The service is persistent, so teardown is probably uncommon, but from a code-review standpoint that is something worth being aware of.

## How To Explain This In A Meeting

Use this simple talk-track:

> TBMFotaService is a coordinator service for TBM FOTA update state. It does not own the update package itself. It starts as a privileged persistent Android service, obtains framework binders through ExtendedServiceManager, verifies that the current vehicle configuration supports TBM FOTA, then connects to CPUCom, ISM, Vehicle Power, Arbiter, and TBM FOTA HMI. Vehicle Power and ISM lifecycle events trigger initial status requests to VCPU. VCPU returns a TBM update status through CPUCom. The service stores that status, notifies app-drawer clients when an actual update starts or ends, asks OTA Arbiter before showing available/forced update UI, broadcasts allowed status to TBM FOTA HMI, and forwards the user's HMI action back to VCPU using CPU command `0x40/0x08`. ISM is used to respect system state, user-on/off state, popup visibility, power-button behavior, and timeout handling.

## Key Code References

| Area | Source |
|---|---|
| Service declaration, permissions, libraries | `app/src/main/AndroidManifest.xml` |
| Main lifecycle/startup | `app/src/main/java/com/mitsubishielectric/ahu/appservice/tbmfotaservice/TBMFotaService.java` |
| CPU commands and callbacks | `app/src/main/java/com/mitsubishielectric/ahu/appservice/tbmfotaservice/cpucom/TbmFotaCpuComManager.java` |
| HMI user-action receiver | `app/src/main/java/com/mitsubishielectric/ahu/appservice/tbmfotaservice/TbmFotaUpdateActionReceiver.java` |
| ISM integration | `app/src/main/java/com/mitsubishielectric/ahu/appservice/tbmfotaservice/ism/IsmManager.java` |
| Arbiter integration | `app/src/main/java/com/mitsubishielectric/ahu/appservice/tbmfotaservice/arbiter/TbmFotaArbiterManager.java` |
| Vehicle power integration | `app/src/main/java/com/mitsubishielectric/ahu/appservice/tbmfotaservice/VehiclePowerManager.java` |
| Vehicle config integration | `app/src/main/java/com/mitsubishielectric/ahu/appservice/tbmfotaservice/data/config/VehicleCfgManager.java` |
| AIDL manager/callback | `tbmfotaserviceinterface/src/main/aidl/com/mitsubishielectric/ahu/lib/tbmfotaserviceinterface` |
