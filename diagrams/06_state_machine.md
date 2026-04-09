# State Machine Diagrams

## Acquisition State Machine (StateMachine.java — hpf.acquisition)

```mermaid
stateDiagram-v2
    [*] --> Idle : HPFRoot init complete

    Idle --> RepeatStarting : startRepeat()
    Idle --> AcquisitionPreparing : startAcquisition(mode, protocol)

    state "Repeat Scanning" as RepeatGroup {
        RepeatStarting --> RepeatRunning : evRepeatStarted
        RepeatRunning --> RepeatStopping : stopRepeat(callback)
        RepeatStopping --> [*]
    }
    RepeatGroup --> Idle : evRepeatStopped / evError→recovery

    state "Series Acquisition" as SeriesGroup {
        AcquisitionPreparing --> AcquisitionStarting : notifyProtocolInstalled()
        AcquisitionStarting --> AcquisitionRunning : evProtocolStarted
        AcquisitionRunning --> AcquisitionStopping : stopAcquisition() / evCompleted
        AcquisitionStopping --> [*]

        AcquisitionPreparing --> Idle : HPFException → doTransition(Idle)\n+ notifier.throwException(e)
        AcquisitionStarting --> Idle : HPFException
        AcquisitionRunning --> Idle : evProtocolAborted / evError
    }
    SeriesGroup --> Idle : evProtocolCompleted / evProtocolAborted

    note right of Idle
        StateID active:
        S_PF_LSM_IMAGING_IDLING
    end note

    note right of RepeatRunning
        StateID active:
        S_PF_LSM_REPEAT_SCANNING
        FunctionEnablerService updated
    end note

    note right of AcquisitionRunning
        StateID active:
        S_PF_LSM_SERIES_SCANNING
        Data I/O active
        Live analysis optional
    end note
```

## Macro State Machine (hpf.macro)

```mermaid
stateDiagram-v2
    [*] --> MacroIdle

    MacroIdle --> MacroExecuting : execute()
    MacroIdle --> MacroRecording : startRecord()
    MacroIdle --> MacroLocal : setLocalMode()
    MacroIdle --> MacroRemote : setRemoteMode()  

    MacroExecuting --> MacroPausing : pause()
    MacroPausing --> MacroPause : evPaused
    MacroPause --> MacroExecuting : resume()
    MacroExecuting --> MacroStopping : stop()
    MacroStopping --> MacroIdle : evStopped

    MacroRecording --> MacroIdle : stopRecord()
    MacroRecording --> MacroAdd : addStep()
    MacroAdd --> MacroRecording : stepAdded

    MacroLocal --> MacroIdle : clearMode()
    MacroRemote --> MacroIdle : clearMode()

    note right of MacroRemote
        S_MACRO_REMOTE:
        RDK remote control active
    end note
```

## Application-Level StateID Reference

```mermaid
graph LR
    subgraph LSMStates["LSM Imaging States (VPA StateID)"]
        direction TB
        S1["S_PF_LSM_IMAGING_IDLING\nIdle — ready for commands"]
        S2["S_PF_LSM_REPEAT_STARTING\nInitiating live view"]
        S3["S_PF_LSM_REPEAT_SCANNING\nLive/repeat scan running"]
        S4["S_PF_LSM_REPEAT_STOPPING\nStopping live view"]
        S5["S_PF_LSM_SERIES_STARTING\nSeries acquisition starting"]
        S6["S_PF_LSM_SERIES_SCANNING\nSeries acquisition running"]
        S7["S_PF_LSM_SERIES_STOPPING\nSeries acquisition stopping"]
        S8["S_PF_LSM_RESTING\nDevice cool-down / rest period"]
        S9["S_PF_LSM_REST_ENDING\nExiting rest period"]
        S10["S_PF_LSM_SERIES_WAITING_APPEND\nWaiting for user append command"]
        S11["S_PF_LSM_SERIES_WAITING_TRIGGER_START\nWaiting for external trigger signal"]
    end

    subgraph AppStates["Application Lifecycle States"]
        A1["S_VPA_APPLICATION_STARTING\nApp bootstrap in progress"]
        A2["S_VPA_APPLICATION_STARTING_END\nBootstrap complete, UI ready"]
    end

    subgraph MacroStates["Macro States (HPF Macro)"]
        M1["S_MACRO_IDLING"]
        M2["S_MACRO_EXECUTING"]
        M3["S_MACRO_PAUSE / S_MACRO_PAUSING"]
        M4["S_MACRO_RECORDING"]
        M5["S_MACRO_STOPPING"]
        M6["S_MACRO_ADD"]
        M7["S_MACRO_LOCAL / S_MACRO_REMOTE"]
    end

    subgraph FES["FunctionEnablerService\n(hpf.core)"]
        FES1["activate(stateId)\nactivateAll(List)\ndeactivate(stateId)\nisEnable(functionId)\nhasPossibilityChange(functionId)\naddFunctionConditionChangedListener()"]
    end

    LSMStates --> FES
    AppStates --> FES
    MacroStates --> FES

    FES --> UI["UI Components\n(buttons, menus, panels)\nenabled/disabled based\non active state"]
```

## State Transition Guard & Thread Safety

```mermaid
flowchart TD
    subgraph StateMachine["StateMachine.java\nhpf.acquisition"]
        SM1["doTransition(fromState, toState)"]
        SM2{"synchronized\nsemaphore"}
        SM3{"fromState ==\ncurrentState?"}
        SM4["exit(currentState)"]
        SM5["currentState = toState"]
        SM6["toState.addListener(listener)"]
        SM7["enter(toState)"]
        SM8["ABORT: stale transition\n(concurrent call won the lock)"]
    end

    SM1 --> SM2
    SM2 --> SM3
    SM3 -- Yes --> SM4 --> SM5 --> SM6 --> SM7
    SM3 -- No --> SM8

    subgraph StateLifecycle["State Lifecycle Methods"]
        SL1["enter(State)\n- start devices\n- register listeners\n- activate StateIDs"]
        SL2["exit(State)\n- stop devices\n- deregister listeners\n- deactivate StateIDs"]
    end

    SM4 --> SL2
    SM7 --> SL1

    style StateMachine fill:#fff2cc,stroke:#d6b656
    style StateLifecycle fill:#d5e8d4,stroke:#82b366
```
