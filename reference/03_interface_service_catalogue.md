# Service Interface Catalogue
**Product:** HPF Framework / FluoView  
**Last Updated:** April 2026

> All services extend `FVComponentService`. Access them through their component domain — never instantiate directly.

---

## Core Services (`hpf.core`)

### `ErrorDispatchService`
**Package:** `jp.co.olympus.hpf.core.errordispatcher`  
**Component:** `MessageDomain`

Centralised error, warning, and information dispatch to all registered listeners.

| Method | Description |
|---|---|
| `putError(ErrorEvent)` | Dispatch non-fatal error; application continues |
| `putFatal(ErrorEvent)` | Dispatch fatal error; triggers application shutdown |
| `putWarning(ErrorEvent)` | Dispatch warning; non-blocking |
| `putInformation(InformationEvent)` | Send informational notification |
| `showError(String)` | Immediately show error dialog |
| `showWarning(String)` | Immediately show warning dialog |
| `showQuestion(String, Style)` | Show dialog; returns `ReturnId` (OK/CANCEL/YES/NO) |
| `addListener(ErrorEventListener)` | Subscribe to error events |
| `replaceUiListener(ErrorEventUIListener)` | Replace UI layer (used in testing) |

**Enums:** `Style { OK, OK_CANCEL, YES_NO }`, `ReturnId { OK, CANCEL, YES, NO }`  
**Dispatch:** Async via `EventProviderImpl` — business listeners notified before UI listener.

---

### `FunctionEnablerService`
**Package:** `jp.co.olympus.hpf.core.enabler.component`
**Component:** `FunctionEnablerDomain` (NOT `MessageDomain`)
**Service ID:** `jp.co.olympus.hpf.core.enabler.service`

Declarative enable/disable engine for UI features. Maps activated **stateId**s (acquisition state, license, device status, etc.) to current **Condition** for each **functionId** via `*.condition` XML rule files.

> **Distinct from `StateMachine.java`.** That governs acquisition transitions; this governs UI feature availability and is *driven by* state changes (among other inputs).

#### Lookup
```java
FunctionEnablerService svc = (FunctionEnablerService)
    FunctionEnablerDomain.getService(FunctionEnablerService.ID);
```

#### Methods (verified against the interface)

| Method | Description |
|---|---|
| `activate(String stateId)` | Activate a state. Auto-deactivates exclusive-pair counterparts declared in the rule files. |
| `activateAll(List<String> stateIdList)` | Batch activate; engine recomputes once at the end. |
| `isEnable(String functionId)` | Current enabled boolean. Listener only fires on **change** — call this for the initial state when subscribing. |
| `isValid(String functionId)` | Validity (e.g., value-in-range) verdict. |
| `isAvailableAndStable(String functionId)` | True only when enabled **and** not in a transitional state. Use before triggering long actions. |
| `hasPossibilityChange(String functionId)` | Hint that the verdict may flip soon — useful for pre-disabling to avoid races. |
| `getFunctionCondition(String functionId)` | Full `Condition` (richer than a bool). |
| `addFunctionConditionChangedListener(String, FunctionConditionChangedListener)` | Subscribe. Listener has three callbacks: `enableChanged`, `functionConditionChanged`, `validationChanged`. **Not a single-method functional interface — a lambda will not compile.** |
| `removeFunctionConditionChangedListener(...)` | Unsubscribe — wire to widget dispose listener. |
| `setDeferTargetStates(List<String>)` | Defer notifications for the listed states (and their exclusive partners) until the list is cleared; only the last-set exclusive state fires when released. Suppresses transitional flicker. |
| `addStateTransitionValidator(StateTransitionValidator)` | Reject illegal state combinations. |
| `removeStateTransitionValidator(...)` | Pair to the above. |
| `getCloneFunctionEnablerService()` | Isolated copy with same StateTable + active states — for what-if/preview computations. |
| `copyAcitavateStates(FunctionEnablerService)` *(sic — typo in interface)* | Copy active states from this instance into another. |
| `getAllState()` | Returns `List<List<DebugState>>`; for InfoView only — `@Deprecated`. |
| `addDebugStateListener` / `removeDebugStateListener` | InfoView only — `@Deprecated`. |
| `setStateTable(DocumentRoot)` | InfoView only — `@Deprecated`. |

#### Rule files
Loaded at boot from any plugin that ships `*.condition` XML (extension constant `FunctionEnablerService.STATE_TABLE_EXTENSION = ".condition"`). Each file declares which combinations of active stateIds enable / disable / validate which functionIds. Pure data — no Java required to add a new rule.

#### Listener interface (`FunctionConditionChangedListener`)
```java
void enableChanged(String functionId, boolean isEnable);
void functionConditionChanged(String functionId, Condition condition);
void validationChanged(String functionId, boolean isValid);
```

See `how-to/05_function_enabler_feature.md` for end-to-end sample.

---

### `MessageDeliveryService` ⚠️ DEPRECATED
**Package:** `jp.co.olympus.hpf.core.message.delivery`

Legacy pub/sub channel. Still in use — do not remove, but do not add new usage.

| Method | Description |
|---|---|
| `crateDelivery(String deliveryId)` | Create a named delivery channel |
| `deleteDelivery(String deliveryId)` | Remove a delivery channel |
| `addListener(String, MessageEventListener)` | Subscribe to a channel |
| `putMessageEvent(String, MessageEvent)` | Publish an event to a channel |

---

## Acquisition Services (`hpf.acquisition`)

### `AcquisitionService`
**Package:** `jp.co.olympus.hpf.acquisition.component`

Manages image capture from connected imaging devices (LSM and camera).

| Method | Description |
|---|---|
| `getLaserImageSource()` | Get LSM image acquisition source |
| `getCameraAdapter()` | Get camera imaging adapter |
| `getLsmLiveHPFImage()` | Get live LSM image model |
| `getCameraLiveHPFImage()` | Get live camera image model |
| `getPreferenceSettings()` | Get acquisition preferences |
| `getSynchronizationSource()` | Get synchronisation control |
| `getEventRecorder()` | Get event recorder |
| `getLSMSeriesManager()` | Get LSM series manager |
| `getCameraSeriesManager()` | Get camera series manager |
| `getLsmRestManager()` | Get LSM rest manager |
| `getCameraRestManager()` | Get camera rest manager |

---

### `SequenceExecutorService`
**Package:** `jp.co.olympus.hpf.acquisition.component`

| Method | Description |
|---|---|
| `getSequenceExecutor(ImagingDeviceType, String protocolManagerId)` | Get executor for device + protocol |

---

### `MATLExecutionService`
**Package:** `jp.co.olympus.hpf.protocol.matl.acquisition.component`

| Method | Description |
|---|---|
| `getMATLManager()` | Get MATL execution controller |
| `getAutoStitchManager()` | Get image stitching controller |

---

## Device Services (`hpf.device`)

### `DeviceControlService`
**Package:** `jp.co.olympus.hpf.device.component`

All hardware device communication.

| Method | Description |
|---|---|
| `connect()` / `disconnect()` | Manage hardware connection |
| `sendMessage(int msgId, MessageObject)` | Send device control message (async) |
| `transmitMessage(...)` | Send message and wait for response (timeout: 50,000ms) |
| `sendByteData(String portId, ByteData, ...)` | Send raw frame data |
| `sendGetDeviceSettingMessage(MessageObject)` | Query device settings |
| `sendChangeDeviceSettingsMessage(...)` | Update device configuration |
| `addMessageReceivedListener(MessageReceivedListener)` | Subscribe to incoming messages |
| `addFrameReceivedListener(String portId, FrameReceivedListener)` | Subscribe to frame data |
| `getDeviceConfiguration()` | Get current hardware configuration |
| `getLaserIndex(String laserId)` | Get laser index by ID |
| `startConcatenation(StartConcatenationMsg)` | Begin batch command block |
| `endConcatenation(EndConcatenationMsg)` | End batch command block |
| `getCameraDeviceSettings()` | Get camera-specific settings |
| `getDeviceErrorSettings()` | Get PMT overrange thresholds |

**Pre-condition:** Must call `connect()` before sending messages.  
**Timeout:** 50,000 ms for synchronous `transmitMessage`.

---

### `StageControlService`
**Package:** `jp.co.olympus.hpf.device.stage.component`

| Method | Description |
|---|---|
| `isXYStageConnected()` | Check if motorised stage is connected |

Extends `SettingsManageable<Settings>`.

---

## Data Services (`hpf.data`)

### `DataManagementService`
**Package:** `jp.co.olympus.hpf.data.management`

Central data lifecycle management.

| Method | Description |
|---|---|
| `createHPFImageModel(ImagingDeviceType, FrameCacher, ROINameSetterManager)` | Create live image model |
| Save/load methods with `SaveAsCallBack` | File persistence with progress callbacks |

---

### `DataFinderService`
**Package:** `jp.co.olympus.hpf.data.management`

Locate and manage frame players for image data access.

| Method | Description |
|---|---|
| `createFramePlayer(HPFImage)` | Create frame player; returns existing if available |
| `getFramePlayer(HPFImage)` | Get existing player or null |
| `deleteFramePlayer(HPFImage)` | Clean up player resources |
| `exists(HPFImage)` | Check if player exists |

---

### `DataProviderService` ⚠️ DEPRECATED
**Package:** `jp.co.olympus.hpf.data.provider`

| Method | Description |
|---|---|
| `createDataSource(String id)` | Create/get data source |
| `getDataSource(String id)` | Retrieve data source |
| `removeDataSource(DataSource)` | Clean up data source |

---

### `ExportService`
**Package:** `jp.co.olympus.hpf.data.export.component`

| Method | Description |
|---|---|
| `getExportManager()` | Get export coordinator |
| `createMovieFileExporter()` | Create movie file exporter |
| `createImageFileExporter()` | Create image file exporter |

---

## Protocol Services (`hpf.protocol`)

### `ProtocolService`
**Package:** `jp.co.olympus.hpf.protocol.core.component`

| Method | Description |
|---|---|
| `getProtocolManager(String id)` | Get executor for protocol ID |
| `getProtocolManagers()` | Get all active protocol managers |
| `addListener(ProtocolServiceListener)` | Subscribe to service events |

---

### `ProtocolManager` (extends `ProtocolExecutor`)
**Package:** `jp.co.olympus.hpf.protocol.core.component`

| Method | Description |
|---|---|
| `getId()` | Get manager ID |
| `getRunningTaskIds()` | Get currently executing task IDs |
| `addExecutionModeChangedListener(Listener)` | Subscribe to execution mode changes |

**Via `ProtocolExecutor`:** `setProtocol()`, `start()`, `stop()`, `pause()`, `resume()`, `releaseProtocol()`

---

### `ExternalTaskService`
**Package:** `jp.co.olympus.hpf.protocol.core.component`

| Method | Description |
|---|---|
| `getExternalTaskCommand(String contributorId, String commandId)` | Get command from external provider |
| `getExternalTaskCommandInfos(ExternalTaskCommandProviderType)` | List available commands |

---

### `MATLDataService`
**Package:** `jp.co.olympus.hpf.protocol.matl.component`

| Method | Description |
|---|---|
| `getImage(IPath datapath)` | Load existing MATL output |
| `getImage(IPath, String baseName, settings, boolean isCreateMapImage)` | Create new MATL output |

---

## Image Processing Services (`hpf.image`)

### `ImageProcessingService`
**Package:** `jp.co.olympus.hpf.image.component`

| Method | Description |
|---|---|
| `getImageCommandFactory()` | Get image processing command builder |
| `isCapability(String apiId, int capability)` | Check native processor capability |

---

### `FRAPProcessingService`
**Package:** `jp.co.olympus.hpf.image.frap.component`

| Method | Description |
|---|---|
| `getFRAPImageCommandFactory()` | Get FRAP analysis command factory |

---

## Communication Services (`hpf.comm`)

### `CommunicationService`
**Package:** `jp.co.olympus.hpf.comm.component`

Low-level connection management.

| Method | Description |
|---|---|
| `createConnection(String id)` | Create new connection |
| `deleteConnection(String id)` | Close and clean up connection |
| `getConnection(String id)` | Retrieve existing connection |
| `isConnectionAvailable(String id)` | Check connection status |

---

### `RpcService`
**Package:** `jp.co.olympus.hpf.rpc.component`

RPC engine management for external scripting/API access.

| Method | Description |
|---|---|
| `getRpcEngineManager()` | Get RPC coordinator |
| `startRpcEngine(RpcEngineType)` | Start RPC service |
| `stopRpcEngine()` | Shutdown RPC service |
| `setServiceURI(RpcEngineType, String uri)` | Configure endpoint |
| `setServicePort(RpcEngineType, int port)` | Set service port |
| `registerFunctionId(String methodId, String functionId)` | Map RPC call to internal function |

---

## CIF Service (`hpf.cif`)

### `CIFService`
**Package:** `jp.co.olympus.hpf.cif.component`

| Method | Description |
|---|---|
| `getManager()` | Get CIF API manager; `null` if native library failed to load |

**Pre-condition:** Requires CIF native Win32 DLL to be loaded.

---

## Base Interface

### `FVComponentService`
**Package:** `jp.co.olympus.hpf.core.component`

Base for all services.

| Method | Description |
|---|---|
| `getId()` | Return service identifier |
| `isAvailable()` | Check if service is operational |

Access pattern:
```java
MyService service = (MyService) MyDomain.getService("jp.co.olympus.hpf.my.service.id");
```
