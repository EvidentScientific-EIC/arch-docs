# Protocol Execution Engine Diagrams

## MATL (Multi-Area Time-Lapse) State Machine — MATLManagerEngine.java

```mermaid
stateDiagram-v2
    [*] --> Ready : evInitialized

    Ready --> Running : start()

    state Running {
        [*] --> ProtocolPreparing
        ProtocolPreparing --> ProtocolRunning : evPrepared / evPreparedInternal
        ProtocolRunning --> WaitingDeviceStopped : area scan complete

        state RunningMain {
            [*] --> Scanning

            state Scanning {
                [*] --> Preparing
                Preparing --> XYStageMoving : move to next area
                XYStageMoving --> Moving : evMoved
                Moving --> Preparing : next area
                Moving --> [*] : evLastScanned / evLastScannedInternal
            }
        }

        state RunningGroup {
            WaitingGroupPause --> GroupPause : evGroupPauseStarted
            GroupPause --> Running : evGroupPauseStopped

            state GroupPause {
                GroupPauseStopping
                GroupPauseStarting
            }
        }

        WaitingAreaPause --> AreaPause : evAreaPauseStarted
        AreaPause --> Running : evAreaPauseStopped

        WaitingAreaPauseCauseError --> Failing : evDeviceErrorOccurred
    }

    state CycleRest {
        [*] --> Rest
        Rest --> CountDownTimer : timer start
        CountDownTimer --> WaitInterval : evCycleRestTimerStopped
        WaitInterval --> [*] : evCycleRestEnded
    }

    Running --> CycleRest : cycle complete, rest interval configured
    CycleRest --> Running : next cycle

    Running --> Ending : evCompleted / evEnded
    Running --> Failing : evFailed / evDeviceErrorOccurred

    Failing --> Stopping : error handled
    Ending --> Completing : finalize data
    Completing --> [*] : evExternalProcessEnded

    Stopping --> [*]

    note right of AreaPause
        User-triggered pause
        between areas
    end note

    note right of CycleRest
        Configured rest between
        time-lapse cycles
        (evFrameRestStopping)
    end note

    note right of Failing
        evErrorOccurred
        evDeviceErrorOccurred
        evRecoverableErrorOccurred
        → auto-retry or abort
    end note
```

## Protocol Execution Event Flow

```mermaid
sequenceDiagram
    participant Caller as AcquisitionApplicationHandler
    participant PE as ProtocolExecutorImpl
    participant ME as MATLManagerEngine
    participant SM as StateMachine
    participant HW as Hardware/Devices
    participant DIO as Data I/O

    Caller->>PE: setProtocol(Protocol)
    PE->>ME: installProtocol()
    ME->>ME: validate structure
    ME-->>PE: evProtocolInstalled
    PE-->>Caller: notifyProtocolInstalled()

    Caller->>PE: start(ExecutionMode)
    PE->>ME: start()

    ME->>ME: evInitialized → Ready
    ME->>SM: doTransition(Running)

    loop For each Cycle
        ME->>ME: ProtocolPreparing → evPrepared

        loop For each Area in cycle
            ME->>HW: moveXYStage(areaCoords)
            HW-->>ME: evMoved
            ME->>ME: XYStageMoving → Moving

            ME->>PE: evTaskStarted(taskId, phase)
            PE-->>Caller: onTaskStarted()

            ME->>HW: executeScan()
            HW-->>DIO: frameData stream
            DIO->>DIO: writeFrame()

            ME-->>PE: evTaskCompleted(taskId)
            PE-->>Caller: onTaskCompleted()

            alt Area Pause requested
                ME->>ME: WaitingAreaPause → AreaPause
                ME-->>PE: evAreaPauseStarted
                Caller->>PE: resume()
                ME->>ME: AreaPause → Running
                ME-->>PE: evAreaPauseStopped
            end
        end

        ME-->>PE: evCycleStarted / evCycleEnded

        alt Rest interval configured
            ME->>ME: Running → CycleRest
            ME->>ME: CountDownTimer → WaitInterval
            ME->>ME: evCycleRestEnded → Running (next cycle)
        end
    end

    ME-->>PE: evCompleted
    PE-->>Caller: notifyProtocolCompleted()
    ME->>ME: Ending → Completing → [done]

    alt Error occurs
        HW-->>ME: device error
        ME->>ME: evDeviceErrorOccurred → Failing
        ME-->>PE: evErrorOccurred(errorId)
        PE-->>Caller: notifyError(HPFException)
        Caller->>Caller: ErrorDispatchService.putError()
    end
```

## General Protocol Event Taxonomy

```mermaid
graph TD
    subgraph ProtocolEvents["Protocol Events (hpf.protocol.model)"]
        PE["ProtocolEvent (base)"]

        PE --> TE["TaskEvent\n- taskId\n- phase: TaskPhaseType\n- condition"]
        PE --> TME["TimedEvent"]
        PE --> TRE["TimerEvent"]
        PE --> UE["UserEvent"]
        PE --> ETE["ExternalTriggerEvent"]
        PE --> CE["ConditionEvent"]
    end

    subgraph MATLEvents["MATL Engine Events (70+ total)"]
        ME1["evInitialized — SM ready"]
        ME2["evPrepared / evPreparedInternal"]
        ME3["evMoved — stage movement done"]
        ME4["evCompleted — scan done"]
        ME5["evFailed — execution failure"]
        ME6["evEnded — sequence ended"]
        ME7["evLastScanned / evLastScannedInternal"]
        ME8["evAreaPauseStarted / Stopped\nevAreaPauseStartedInternal / StoppedInternal"]
        ME9["evGroupPauseStarted / Stopped"]
        ME10["evCycleRestEnded / evCycleRestTimerStopped"]
        ME11["evDeviceErrorOccurred — device-level error"]
        ME12["evDeviceRestStarted / Stopped"]
        ME13["evProtocolCompleted / Failed / Installed"]
        ME14["evFrameRestStopping"]
        ME15["evExternalProcessEnded"]
        ME16["evRecoverableErrorOccurred — auto-retry"]
    end

    subgraph ProtocolLifecycle["Protocol Executor Events (hpf.protocol.core)"]
        PL1["evProtocolStarted"]
        PL2["evProtocolCompleted"]
        PL3["evProtocolAborted"]
        PL4["evTaskStarted / Completed / Failed / Skipped"]
        PL5["evTaskWaitingTrigger"]
        PL6["evCycleStarted / evCycleEnded"]
        PL7["evProtocolPaused / Resumed / Stopped"]
    end

    subgraph EventPattern["Event Class Pattern (RiJEvent-based)"]
        EP["class evInitialized extends RiJEvent\n  static final int ID = 25416\n  lId = ID (set in constructor)\n  isTypeOf(id): returns id == ID"]
    end

    MATLEvents --> EventPattern
    ProtocolLifecycle --> EventPattern
```

## FluoView Sequence Protocol Structure

```mermaid
graph TD
    subgraph SeqProto["jp.co.olympus.fluoview.protocol.sequence"]
        SM["SequenceManagerImpl\nOrchestrates acquisition sequences"]

        subgraph Subtasks["Sub-Task Factories"]
            ST1["LoadParametersSubTaskFactoryForSequence\nLoads observation params per sequence step"]
            ST2["PrepareParametersSubTaskForSequence\nPrepares device parameters"]
        end

        subgraph Extenders["Sequence Manager Extenders"]
            EX1["ChannelSequenceManagerExtender\nHandles channel-specific sequencing"]
            EX2["ExperimentInfoSequenceManagerExtender\nAttaches experiment metadata"]
            EX3["Other extenders (optical elements,\ndisk space check, etc.)"]
        end

        subgraph Commands["Protocol Commands"]
            CMD1["OpticalElementChangeCommand\nSwitches objectives/filters between steps"]
            CMD2["DiskSpaceCheckCommand\nValidates available storage before start"]
            CMD3["Acquisition step commands"]
        end
    end

    SM --> ST1 & ST2
    SM --> EX1 & EX2 & EX3
    SM --> CMD1 & CMD2 & CMD3
    SM --> HPFProto["hpf.protocol.core\nProtocolExecutorImpl"]
```
