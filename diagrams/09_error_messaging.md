# Error Handling & Inter-Component Messaging

## Error Dispatch Architecture

```mermaid
graph TD
    subgraph Sources["Error Sources"]
        S1["Device Communication\n(Win32 native failure)"]
        S2["Protocol Engine\n(evErrorOccurred\nevDeviceErrorOccurred\nevRecoverableErrorOccurred)"]
        S3["State Machine\n(HPFException caught\nin state transition)"]
        S4["Data I/O\n(file read/write failure)"]
        S5["License Validation\n(invalid/expired license)"]
    end

    subgraph EDS["ErrorDispatchService\n(hpf.core — MessageDomain)"]
        EDS1["putError(ErrorEvent)\n→ non-fatal, app continues"]
        EDS2["putFatal(ErrorEvent)\n→ fatal, triggers shutdown"]
        EDS3["putWarning(ErrorEvent)\n→ non-blocking warning"]
        EDS4["putInformation(InformationEvent)\n→ informational display"]
        EDS5["showError(String)\n→ immediate dialog"]
        EDS6["showQuestion(String, Style)\n→ dialog: OK/CANCEL, YES/NO"]
    end

    subgraph ErrorEvent["ErrorEvent"]
        EE1["errId: int\n(error code)"]
        EE2["message: String\n(human-readable)"]
        EE3["cause: Throwable"]
        EE4["calendar: Calendar\n(timestamp)"]
        EE5["Type enum:\nERROR | FATAL | WARNING"]
    end

    subgraph HPFEx["HPFException Hierarchy"]
        HE["HPFException extends Exception\n- errorId: int\n- HPFError enum code\n- formatted message args"]
        HE --> HE1["HPFException(HPFError)"]
        HE --> HE2["HPFException(HPFError, Object... args)"]
        HE --> HE3["HPFException(Throwable, HPFError, args)"]
    end

    subgraph Dispatch["Async Dispatch (EventProviderImpl)"]
        EV["EventProviderImpl\n(Active Object pattern)\n- Internal message queue\n- Dedicated dispatch thread\n- Sequential listener notification"]
    end

    subgraph Listeners["Error Listeners"]
        L1["Business Logic Listeners\n(state handlers, services)\nNotified FIRST"]
        L2["ErrorEventUIListener\n(replaceUiListener)\nNotified LAST\n→ shows dialog / log"]
        L3["Log Service\n(hpf.log Win32)\nAll errors timestamped"]
    end

    Sources --> EDS1 & EDS2 & EDS3
    S3 --> HPFEx --> EDS1
    EDS --> ErrorEvent
    EDS1 & EDS2 & EDS3 --> Dispatch
    Dispatch --> L1 --> L2 --> L3

    subgraph Recovery["Error Recovery Pattern"]
        R1["Catch HPFException\nin state handler"]
        R2["Transition to SAFE STATE\n(usually Idle)"]
        R3["notifier.throwException(e)\n→ dispatch to listeners"]
        R4["ErrorDispatchService.putError()\nor putFatal()"]
        R1 --> R2 --> R3 --> R4
    end

    S3 --> Recovery

    style Sources fill:#f8cecc,stroke:#b85450
    style EDS fill:#fff2cc,stroke:#d6b656
    style Dispatch fill:#d5e8d4,stroke:#82b366
    style Listeners fill:#dae8fc,stroke:#6c8ebf
    style Recovery fill:#fce4d6,stroke:#ed7d31
    style HPFEx fill:#e1d5e7,stroke:#9673a6
```

## Error Flow Sequence: Device Error During Acquisition

```mermaid
sequenceDiagram
    participant HW as Hardware (Win32)
    participant RPC as HPF COMM/RPC
    participant SM as StateMachine
    participant SEN as SeriesEventNotifier
    participant AAH as AcquisitionApplicationHandler
    participant EDS as ErrorDispatchService
    participant UI as UI Layer
    participant LOG as hpf.log (Win32)

    HW->>RPC: device error signal
    RPC->>SM: notifyDeviceError()

    SM->>SM: evDeviceErrorOccurred
    SM->>SM: synchronized(semaphore):\ndoTransition(currentState → Idle)
    Note over SM: exit(AcquisitionRunning)\nenter(Idle)

    SM->>SEN: notifyProtocolErrorStopped()
    SEN->>SEN: notifyError(HPFException)

    SEN->>AAH: onError(HPFException)

    AAH->>EDS: putError(new ErrorEvent(errId, message, cause))
    EDS->>EDS: EventProviderImpl.enqueue(event)

    par Async dispatch (dedicated thread)
        EDS->>AAH: ErrorEventListener.onError(event)
        Note over AAH: Business logic cleanup
        EDS->>UI: ErrorEventUIListener.onError(event)
        Note over UI: Show error dialog to user
        EDS->>LOG: log(timestamp, errId, message, stackTrace)
    end

    UI-->>AAH: user dismissed dialog
    AAH->>AAH: FunctionEnablerService.activate(S_PF_LSM_IMAGING_IDLING)
    AAH->>UI: update UI enabled states
```

## Inter-Component Messaging Architecture

```mermaid
graph TD
    subgraph MessageDomain["MessageDomain (hpf.core.component.message)"]
        MD["MessageDomain\nID: jp.co.olympus.hpf.core.component.message\nextends FVComponentDomain\nStatic service registry"]
        MD --> EDS2["ErrorDispatchService\n(primary service)"]
        MD --> MDS["MessageDeliveryService\n(@Deprecated legacy pub/sub)"]
    end

    subgraph MessageDelivery["MessageDeliveryService (Legacy Pub/Sub)"]
        MDI["MessageDeliveryServiceImpl"]
        MDI --> DEL["Per-deliveryId\nEventProviderImpl queues"]
        MDI --> ML["MessageEventListener\nsubscribers"]
        API1["crateDelivery(deliveryId)\naddListener(deliveryId, listener)\nputMessageEvent(deliveryId, event)"]
        MDI --> API1
    end

    subgraph EventDispatcher["EventDispatcher (hpf.core.engine)"]
        ED["EventDispatcher interface\n+putEvent(Event)\n+dispose()"]
        EDP["EventProviderImpl\n(Active Object)\n- BlockingQueue of Events\n- Worker thread\n- Sequential dispatch"]
        ED --> EDP
    end

    subgraph AcqNotifiers["Acquisition Event Notifiers"]
        SEN2["SeriesEventNotifier\nListenerManagerBase<SeriesExecutionListener>"]
        REN["RepeatEventNotifier\nListenerManagerBase<RepeatExecutionListener>"]
        ASF["AcquisitionStateForwarder\nimplements RepeatExecutionListener\nimplements SeriesExecutionListener\nMultiplexes to multiple subscribers"]

        SEN2Events["notifyProtocolInstalled()\nnotifyProtocolStarted()\nnotifyProtocolStopped()\nnotifyProtocolErrorStopped()\nnotifyProtocolCompleted()\nnotifyError(HPFException)"]

        SEN2 --> SEN2Events
        ASF --> SEN2 & REN
    end

    subgraph FES2["FunctionEnablerService (hpf.core)"]
        FES["FunctionEnablerServiceImpl\n+activate(stateId)\n+deactivate(stateId)\n+isEnable(functionId)\n+addFunctionConditionChangedListener()\n+hasPossibilityChange(functionId)"]
        COND["FunctionEnablerImpl\nCondition tables\nTransition validators"]
        FES --> COND
    end

    subgraph Consumers["Event Consumers"]
        UI2["UI Components\n(buttons, menus)\nenabled/disabled by FES"]
        ACQHandler["AcquisitionApplicationHandler\n(primary coordinator)"]
        AnaHandler["Analysis Handlers"]
    end

    MessageDomain --> EventDispatcher
    MessageDelivery --> EventDispatcher
    AcqNotifiers --> EventDispatcher
    FES2 --> Consumers
    EDS2 --> Consumers
    ASF --> ACQHandler
    ACQHandler --> FES2

    style MessageDomain fill:#fff2cc,stroke:#d6b656
    style EventDispatcher fill:#d5e8d4,stroke:#82b366
    style AcqNotifiers fill:#dae8fc,stroke:#6c8ebf
    style FES2 fill:#e1d5e7,stroke:#9673a6
    style Consumers fill:#fce4d6,stroke:#ed7d31
```

## Active Object Pattern (EventProviderImpl) — Thread Safety Detail

```mermaid
sequenceDiagram
    participant Caller as Any Component
    participant EPI as EventProviderImpl
    participant Q as BlockingQueue[Event]
    participant WT as Worker Thread
    participant L as Listeners[]

    Note over EPI: Constructed at component init<br/>Worker thread started immediately

    Caller->>EPI: putEvent(event)
    EPI->>Q: offer(event)
    Note over Caller: Returns immediately<br/>(non-blocking)

    loop Worker thread running
        WT->>Q: take() [blocks until event]
        Q-->>WT: event

        loop For each listener (sequential)
            WT->>L: listener.onEvent(event)
            L-->>WT: (return)
        end
    end

    Caller->>EPI: dispose()
    EPI->>WT: interrupt / poison pill
    WT-->>EPI: thread exits cleanly
```
