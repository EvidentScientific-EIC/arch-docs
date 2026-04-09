# Expanded Class Diagram — Full Domain Model

## Bootstrap & Component Framework

```mermaid
classDiagram
    class HPFRoot {
        -static HPFRoot instance$
        -ParallelComponentManager componentManager
        -List~ApplicationRootHandler~ handlers
        +static getInstance() HPFRoot
        -initializeComponents() void
        -loadHandlers() void
        +shutdown() void
    }

    class FVComponentDomain {
        <<abstract>>
        +initialize() void
        +dispose() void
        +getService(String serviceId) FVComponentService
    }

    class FVComponentService {
        <<abstract>>
        +static PROVIDER ServiceProvider
        +initialize() void
        +dispose() void
    }

    class FVComponentWithPrereqs {
        -FVComponentDomain object
        -String[] prereqs
        +getObject() FVComponentDomain
        +getPrereqs() String[]
    }

    class ApplicationRootHandler {
        <<interface>>
        +onPostInitialization() void
        +getPriority() int
    }

    class ParallelComponentManager {
        -List~FVComponent~ components
        +initializeComponents() void
        +shutdownComponents() void
    }

    class HPFExtensionRegistry {
        <<singleton>>
        +static INSTANCE HPFExtensionRegistry$
        -IExtensionRegistry registry
        +getConfigurationElementsFor(String id) IConfigurationElement[]
        +getExtensions(String id) IExtension[]
    }

    class ExtensionLoader {
        +static loadExtension(String pointId, ExtensionLoaderCallback cb) void
    }

    HPFRoot *-- ParallelComponentManager
    HPFRoot o-- ApplicationRootHandler
    ParallelComponentManager o-- FVComponentWithPrereqs
    FVComponentWithPrereqs --> FVComponentDomain
    FVComponentDomain o-- FVComponentService
    HPFExtensionRegistry --> ExtensionLoader
```

## Service Layer

```mermaid
classDiagram
    class ErrorDispatchService {
        <<interface>>
        +putError(ErrorEvent) void
        +putFatal(ErrorEvent) void
        +putWarning(ErrorEvent) void
        +putInformation(InformationEvent) void
        +showError(String) void
        +showQuestion(String, Style) DialogResult
        +addListener(ErrorEventListener) void
        +replaceUiListener(ErrorEventUIListener) void
    }

    class ErrorEvent {
        +int errId
        +String message
        +Throwable cause
        +Calendar calendar
        +Type type
    }

    class ErrorEventType {
        <<enumeration>>
        ERROR
        FATAL
        WARNING
    }

    class ErrorEventListener {
        <<interface>>
        +onError(ErrorEvent) void
        +onWarning(ErrorEvent) void
        +onFatal(ErrorEvent) void
    }

    class HPFException {
        -int errorId
        +HPFException(HPFError err)
        +HPFException(HPFError err, Object args)
        +HPFException(Throwable cause, HPFError err, Object args)
        +getErrorId() int
        +getMessage() String
    }

    class FunctionEnablerService {
        <<interface>>
        +activate(String stateId) void
        +activateAll(List stateIds) void
        +deactivate(String stateId) void
        +isEnable(String functionId) boolean
        +getFunctionCondition(String functionId) Condition
        +hasPossibilityChange(String functionId) boolean
        +addFunctionConditionChangedListener(String, Listener) void
    }

    class DeviceControlService {
        <<interface>>
        +sendCommand(DeviceCommand) CommandResult
        +getDeviceStatus(String deviceId) DeviceStatus
        +resetDevice(String deviceId) void
        +addListener(DeviceEventListener) void
    }

    class AcquisitionService {
        <<interface>>
        +startRepeat(RepeatSettings) void
        +stopRepeat(Runnable callback) void
        +startAcquisition(ExecutionMode, Protocol) void
        +stopAcquisition() void
        +getLivePackagedModel() LiveModel
        +addListener(AcquisitionListener) void
    }

    class DataFinderService {
        <<interface>>
        +findFiles(String path) List~HPFImageModel~
        +openFile(String path) HPFImageModel
    }

    class DataProviderService {
        <<interface>>
        +getAvailableFormats() List~FormatInfo~
        +createModel(FormatType) HPFImageModel
    }

    ErrorDispatchService --> ErrorEvent
    ErrorEvent --> ErrorEventType
    ErrorDispatchService --> ErrorEventListener
    HPFException --> ErrorDispatchService : dispatched via
```

## Device & Acquisition Models

```mermaid
classDiagram
    class DeviceConfiguration {
        +List~CameraDeviceParam~ cameras
        +List~LaserParam~ lasers
        +StageParam stage
        +PiezoParam piezo
        +ScannerSettings scanner
        +ObjectiveLensParam objectiveLens
    }

    class CameraDeviceParam {
        +String cameraId
        +CameraImageSizeFeatureParam imageSize
        +CameraClipSizeFeatureParam clipSize
        +List~CameraFeature~ features
        +CameraFeature getFeature(CameraFeatureType) CameraFeature
    }

    class CameraFeature {
        +CameraFeatureType type
        +Object value
        +boolean enabled
    }

    class CameraFeatureType {
        <<enumeration>>
        IMAGE_SIZE
        COLOR_DEPTH
        IMAGING_MODE
        EXPOSURE_TIME
        GAIN
        EM_GAIN
        COOLING
        BINNING
        TRIGGER_MODE
    }

    class LaserParam {
        +String laserId
        +double wavelength
        +double power
        +boolean enabled
        +LaserType type
    }

    class PMTParam {
        +int pmtIndex
        +double voltage
        +double gain
        +double offset
        +boolean enabled
    }

    class ScannerSettings {
        +double pixelDwellTime
        +int scanSpeed
        +ScanMode scanMode
        +double zoomFactor
        +double rotation
    }

    class ObservationSettings {
        +String methodName
        +List~ISMChannelSettings~ channels
        +ScannerSettings scanner
        +ZStackSettings zStack
        +TimeLapseSettings timeLapse
        +String objectiveLensId
    }

    class ISMChannelSettings {
        +int channelIndex
        +double laserPower
        +double pmtVoltage
        +double gain
        +double offset
        +String fluorochrome
        +int detectorIndex
    }

    class ZStackSettings {
        +double topZ
        +double bottomZ
        +double stepSize
        +int sliceCount
    }

    class TimeLapseSettings {
        +int cycleCount
        +double intervalSeconds
        +boolean hasRestInterval
    }

    class DeviceCommand {
        +String targetDevice
        +String commandType
        +Map~String_Object~ parameters
        +int priority
    }

    class CommandResult {
        +boolean success
        +String errorCode
        +Object payload
        +long timestamp
    }

    DeviceConfiguration "1" *-- "many" CameraDeviceParam
    DeviceConfiguration "1" *-- "many" LaserParam
    DeviceConfiguration "1" *-- "1" ScannerSettings
    CameraDeviceParam "1" *-- "many" CameraFeature
    CameraFeature --> CameraFeatureType
    ObservationSettings "1" *-- "many" ISMChannelSettings
    ObservationSettings "1" *-- "1" ScannerSettings
    ObservationSettings "1" *-- "0..1" ZStackSettings
    ObservationSettings "1" *-- "0..1" TimeLapseSettings
    DeviceControlService ..> DeviceCommand
    DeviceControlService ..> CommandResult
```

## Image Model (EMF/ECORE)

```mermaid
classDiagram
    class HPFImageModel {
        +String modelId
        +ImageSetMetainfo metainfo
        +List~ImageFrame~ frames
        +FormatType format
        +getFrame(int index) ImageFrame
        +getChannelCount() int
        +getZSliceCount() int
        +getTimePointCount() int
    }

    class ImageSetMetainfo {
        <<abstract>>
        +encode() byte[]
        +decode(byte[]) void
        +getMajorVersion() int
        +getMinorVersion() int
    }

    class OIFImageSetMetainfo {
        +IFileInformationHolder
        +ILutSet
        +IImagePropertyHolder
        +IAnnotatable
        +IOverlayItemListHolder
        +IReferenceImageInfoHolder
        +IEventListHolder
    }

    class FvCommonAcquisition {
        +String acquisitionId
        +Date timestamp
        +ImageProperties properties
    }

    class LSMAcquisition {
        +List~LSMPhase~ phases
        +LSMImagingParam imagingParam
        +LSMImageProperties imageProperties
    }

    class LSMPhase {
        +int phaseIndex
        +String phaseName
        +List~LambdaChannel~ lambdaChannels
        +ScanArea scanArea
        +double zPosition
    }

    class LambdaChannel {
        +double excitationWavelength
        +double emissionWavelength
        +int detectorIndex
        +String fluorochrome
        +PMTParam pmtSettings
    }

    class LSMImagingParam {
        +List~PMTParam~ pmts
        +ScannerSettings scanner
        +BrightnessAdjustmentParam brightness
        +List~LaserParam~ activeLasers
    }

    class LSMImageProperties {
        +String objectiveLens
        +double magnification
        +double numericalAperture
        +String immersionType
        +double pinholeDiameter
    }

    class ImageProperties {
        +int width
        +int height
        +int bitDepth
        +double pixelSizeX
        +double pixelSizeY
        +double pixelSizeZ
        +String unit
        +int channels
        +int zSlices
        +int timePoints
    }

    class ImageBlockData {
        -ByteBuffer buffer
        +getData() ByteBuffer
        +getSize() int
        +getWidth() int
        +getHeight() int
        +getBitDepth() int
    }

    HPFImageModel *-- ImageSetMetainfo
    ImageSetMetainfo <|-- OIFImageSetMetainfo
    LSMAcquisition ..|> FvCommonAcquisition
    FvCommonAcquisition *-- ImageProperties
    LSMAcquisition "1" *-- "many" LSMPhase
    LSMAcquisition *-- LSMImagingParam
    LSMAcquisition *-- LSMImageProperties
    LSMPhase "1" *-- "many" LambdaChannel
    LSMImagingParam "1" *-- "many" PMTParam
    HPFImageModel --> ImageBlockData : frames stored as
```

## Analysis Framework

```mermaid
classDiagram
    class Module {
        <<abstract>>
        +String moduleId
        +String moduleType
        +int frameIndex
        +int channelIndex
        +ROI roi
        +setupModule(FrameContext ctx) void
        +validate() ValidationResult
    }

    class AnalysisCommandHandler {
        <<interface>>
        +execute(Module) ModuleResult
        +canHandle(String moduleType) boolean
        +getPriority() int
    }

    class AbstractCommandFactoryProvider {
        <<abstract>>
        +createCommandFactory(String type) CommandFactory
        +getSupportedTypes() List~String~
    }

    class AnalysisSequenceManagerService {
        -List~Module~ moduleSequence
        -List~ROI~ roiList
        -AnalysisCommandHandlerProvider handlerProvider
        +loadProtocol(File protocolFile) void
        +executeSequence() void
        +pauseSequence() void
        +stopSequence() void
        -updateROIList(ModuleResult result) void
    }

    class AnalysisCommandHandlerProvider {
        -List~AnalysisCommandHandler~ handlers
        +getHandler(String moduleType) AnalysisCommandHandler
        +register(AnalysisCommandHandler handler) void
    }

    class ModuleResult {
        +Module source
        +Object data
        +List~ROI~ updatedROIs
        +boolean success
        +String errorMessage
    }

    class ValidationResult {
        +boolean valid
        +List~String~ errors
        +List~String~ warnings
    }

    class ROI {
        +String roiId
        +ROIType type
        +List~Point2D~ points
        +int channelIndex
        +int frameIndex
    }

    class AnalysisGraphDisplayManager {
        +renderResult(ModuleResult result) void
        +clearResults() void
        +exportGraph(File output) void
    }

    AnalysisSequenceManagerService "1" o-- "many" Module
    AnalysisSequenceManagerService --> AnalysisCommandHandlerProvider
    AnalysisCommandHandlerProvider "1" o-- "many" AnalysisCommandHandler
    AnalysisCommandHandler ..> Module
    AnalysisCommandHandler ..> ModuleResult
    Module --> ValidationResult
    Module --> ROI
    ModuleResult --> ROI
    AnalysisSequenceManagerService --> AnalysisGraphDisplayManager
```

## Protocol Engine

```mermaid
classDiagram
    class ProtocolExecutor {
        <<interface>>
        +setProtocol(Protocol) void
        +releaseProtocol() void
        +start(ExecutionMode) void
        +stop() void
        +pause() void
        +resume() void
        +addListener(ProtocolExecutionListener) void
    }

    class ProtocolExecutorImpl {
        -ProtocolExecutorDelegate delegate
        -ProtocolManagerEngine monitor
        -ExecutionMode mode
        -ListenerManagerBase listeners
    }

    class Protocol {
        +String protocolId
        +List~Task~ tasks
        +List~Cycle~ cycles
        +ProtocolSettings settings
    }

    class Task {
        +String taskId
        +TaskType type
        +TaskPhaseType phase
        +List~ProtocolEvent~ triggerEvents
        +List~ProtocolEvent~ completionEvents
    }

    class SeriesEventNotifier {
        -ListenerManagerBase~SeriesExecutionListener~ listeners
        -SeriesImagingStateSynchronizer stateSynchronizer
        +notifyProtocolInstalled() void
        +notifyProtocolStarted() void
        +notifyProtocolStopped() void
        +notifyProtocolErrorStopped() void
        +notifyProtocolCompleted() void
        +notifyError(HPFException e) void
    }

    class AcquisitionStateForwarder {
        -ListenerManagerBase~RepeatExecutionListener~ repeatListeners
        -ListenerManagerBase~SeriesExecutionListener~ seriesListeners
        +addRepeatListener(RepeatExecutionListener) void
        +addSeriesListener(SeriesExecutionListener) void
    }

    class StateMachine {
        -State currentState
        -Object semaphore
        +doTransition(State from, State to) void
        -enter(State s) void
        -exit(State s) void
    }

    class State {
        <<abstract>>
        +startRepeat() void
        +stopRepeat(Runnable) void
        +startAcquisition(ExecutionMode, Protocol) void
        +stopAcquisition() void
        +addListener(StateListener) void
        +removeListener(StateListener) void
    }

    class StateIdle { }
    class StateRepeatRunning { }
    class StateAcquisitionPreparing {
        +notifyProtocolInstalled() void
    }
    class StateAcquisitionRunning { }

    ProtocolExecutorImpl ..|> ProtocolExecutor
    ProtocolExecutorImpl --> SeriesEventNotifier
    Protocol "1" *-- "many" Task
    StateMachine *-- State
    State <|-- StateIdle
    State <|-- StateRepeatRunning
    State <|-- StateAcquisitionPreparing
    State <|-- StateAcquisitionRunning
    AcquisitionStateForwarder --> SeriesEventNotifier
    StateAcquisitionPreparing ..> ProtocolExecutor
```
