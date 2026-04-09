# Protocol Execution Engine Diagrams

## MATL (Multi-Area Time-Lapse) State Machine — MATLManagerEngine.java

```mermaid
stateDiagram-v2
    [*] --> Ready : evInitialized
    Ready --> Running : start()

    state Running {
        [*] --> ProtocolPreparing
        ProtocolPreparing --> ProtocolRunning : evPrepared, evPreparedInternal

        state ProtocolRunning {
            [*] --> Preparing
            Preparing --> XYStageMoving : move to next area
            XYStageMoving --> Moving : evMoved
            Moving --> Preparing : next area
            Moving --> [*] : evLastScanned, evLastScannedInternal
        }

        ProtocolRunning --> WaitingDeviceStopped : all areas scanned
        WaitingDeviceStopped --> ProtocolPreparing : device stopped, next cycle prep

        ProtocolRunning --> WaitingAreaPause : area pause requested
        WaitingAreaPause --> AreaPause : evAreaPauseStarted
        AreaPause --> ProtocolRunning : evAreaPauseStopped

        ProtocolRunning --> WaitingGroupPause : group pause requested
        WaitingGroupPause --> GroupPausing : evGroupPauseStarted
        GroupPausing --> ProtocolRunning : evGroupPauseStopped

        ProtocolRunning --> WaitingAreaPauseCauseError : device error in area
        WaitingAreaPauseCauseError --> [*] : evDeviceErrorOccurred
    }

    state CycleRest {
        [*] --> Rest
        Rest --> CountDownTimer : timer start
        CountDownTimer --> WaitInterval : evCycleRestTimerStopped
        WaitInterval --> [*] : evCycleRestEnded
    }

    Running --> CycleRest : cycle complete, rest interval configured
    CycleRest --> Running : next cycle

    Running --> Ending : evCompleted, evEnded
    Running --> Failing : evFailed, evDeviceErrorOccurred

    Failing --> Stopping : error handled
    Ending --> Completing : finalize data
    Completing --> [*] : evExternalProcessEnded
    Stopping --> [*]

    note right of CycleRest
        Configured rest between
        time-lapse cycles
        (evFrameRestStopping)
    end note

    note right of Failing
        evErrorOccurred
        evDeviceErrorOccurred
        evRecoverableErrorOccurred
        -> auto-retry or abort
    end note
```

## Protocol Execution Event Flow

```mermaid
sequenceDiagram
    participant Caller as AcquisitionApplicationHandler
    participant PE as ProtocolExecutorImpl
    participant ME as MATLManagerEngine
    participant HW as Hardware/Devices
    participant DIO as Data I/O

    Caller->>PE: setProtocol(Protocol)
    PE->>ME: installProtocol()
    ME->>ME: validate structure
    ME-->>PE: evProtocolInstalled
    PE-->>Caller: notifyProtocolInstalled()

    Caller->>PE: start(ExecutionMode)
    PE->>ME: start()
    ME-->>PE: evInitialized, Ready state entered
    ME-->>PE: Running state entered

    loop For each Cycle
        ME-->>PE: evCycleStarted
        ME->>ME: ProtocolPreparing

        loop For each Area in cycle
            ME->>HW: moveXYStage(areaCoords)
            HW-->>ME: evMoved
            ME->>ME: XYStageMoving -> Moving

            ME-->>PE: evTaskStarted(taskId, phase)
            PE-->>Caller: onTaskStarted()

            ME->>HW: executeScan()
            HW-->>DIO: frameData stream
            DIO->>DIO: writeFrame()

            ME-->>PE: evTaskCompleted(taskId)
            PE-->>Caller: onTaskCompleted()

            alt Area Pause requested
                ME->>ME: WaitingAreaPause -> AreaPause
                ME-->>PE: evAreaPauseStarted
                Caller->>PE: resume()
                ME->>ME: AreaPause -> ProtocolRunning
                ME-->>PE: evAreaPauseStopped
            end
        end

        ME-->>PE: evCycleEnded

        alt Rest interval configured
            ME->>ME: Running -> CycleRest -> CountDownTimer -> WaitInterval
            ME->>ME: evCycleRestEnded, re-enter Running
        end
    end

    ME-->>PE: evCompleted
    PE-->>Caller: notifyProtocolCompleted()
    ME->>ME: Ending -> Completing -> done

    alt Error occurs during scan
        HW-->>ME: device error signal
        ME->>ME: evDeviceErrorOccurred -> Failing
        ME-->>PE: evErrorOccurred(errorId)
        PE-->>Caller: notifyError(HPFException)
        Caller->>Caller: ErrorDispatchService.putError()
    end
```

## General Protocol Event Taxonomy

```mermaid
graph TD
    subgraph EventPattern["Event Class Pattern (RiJEvent-based)"]
        EP["RiJEvent base class\n  lId: long (unique event ID)\n  isTypeOf(id): boolean\n  e.g. evInitialized ID = 25416"]
    end

    subgraph MATLEvents["MATL Engine Events (70+ total)"]
        ME1["evInitialized — SM ready"]
        ME2["evPrepared, evPreparedInternal"]
        ME3["evMoved — stage movement done"]
        ME4["evCompleted — scan done"]
        ME5["evFailed — execution failure"]
        ME6["evEnded — sequence ended"]
        ME7["evLastScanned, evLastScannedInternal"]
        ME8["evAreaPauseStarted, Stopped\nevAreaPauseStartedInternal, StoppedInternal"]
        ME9["evGroupPauseStarted, Stopped"]
        ME10["evCycleRestEnded, evCycleRestTimerStopped"]
        ME11["evDeviceErrorOccurred — device-level error"]
        ME12["evDeviceRestStarted, Stopped"]
        ME13["evProtocolCompleted, Failed, Installed"]
        ME14["evFrameRestStopping"]
        ME15["evExternalProcessEnded"]
        ME16["evRecoverableErrorOccurred — auto-retry"]
    end

    subgraph ProtocolEvents["Protocol Events (hpf.protocol.model)"]
        PE2["ProtocolEvent (base)"]
        PE2 --> TE["TaskEvent\n- taskId\n- phase: TaskPhaseType\n- condition"]
        PE2 --> TME["TimedEvent"]
        PE2 --> TRE["TimerEvent"]
        PE2 --> UE["UserEvent"]
        PE2 --> ETE["ExternalTriggerEvent"]
        PE2 --> CE["ConditionEvent"]
    end

    subgraph ProtocolLifecycle["Protocol Executor Events (hpf.protocol.core)"]
        PL1["evProtocolStarted"]
        PL2["evProtocolCompleted"]
        PL3["evProtocolAborted"]
        PL4["evTaskStarted, Completed, Failed, Skipped"]
        PL5["evTaskWaitingTrigger"]
        PL6["evCycleStarted, evCycleEnded"]
        PL7["evProtocolPaused, Resumed, Stopped"]
    end

    EventPattern --> MATLEvents
    EventPattern --> ProtocolLifecycle
    EventPattern --> ProtocolEvents
```

## FluoView Sequence Protocol Structure

```mermaid
graph TD
    subgraph SeqProto["Sequence Protocol\njp.co.olympus.fluoview.protocol.sequence"]
        SM["SequenceManagerImpl\nOrchestrates acquisition sequences"]

        subgraph Subtasks["Sub-Task Factories"]
            ST1["LoadParametersSubTaskFactory\nLoads observation params per step"]
            ST2["PrepareParametersSubTask\nPrepares device parameters"]
        end

        subgraph Extenders["Sequence Manager Extenders"]
            EX1["ChannelSequenceManagerExtender\nHandles channel-specific sequencing"]
            EX2["ExperimentInfoSequenceManagerExtender\nAttaches experiment metadata"]
            EX3["OpticalElement, DiskSpaceCheck\nand other extenders"]
        end

        subgraph Commands["Protocol Commands"]
            CMD1["OpticalElementChangeCommand\nSwitches objectives/filters between steps"]
            CMD2["DiskSpaceCheckCommand\nValidates storage before start"]
            CMD3["Acquisition step commands"]
        end
    end

    HPFProto["hpf.protocol.core\nProtocolExecutorImpl"]

    SM -->|"creates"| ST1
    SM -->|"creates"| ST2
    SM -->|"uses"| EX1
    SM -->|"uses"| EX2
    SM -->|"uses"| EX3
    SM -->|"generates"| CMD1
    SM -->|"generates"| CMD2
    SM -->|"generates"| CMD3
    SM -->|"delegates execution to"| HPFProto
```
