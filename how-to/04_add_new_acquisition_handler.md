# How-To: Add a New Acquisition Handler / Startup Component
**Product:** FluoView / HPF / VPA  
**Last Updated:** April 2026

---

## Overview

`ApplicationRootHandler` is the entry point for any code that must run at application startup. Use it to initialize services, register listeners, load saved settings, or activate feature states. This is also the pattern for adding cross-cutting acquisition behaviour.

---

## Step 1: Implement ApplicationRootHandler

```java
public class MyFeatureApplicationHandler implements ApplicationRootHandler {

    private DeviceControlService deviceControlService;
    private FunctionEnablerService functionEnablerService;

    @Override
    public void onPostInitialization() {
        // Get services from their component domains.
        // NB: each service has its OWN domain — pick the right one.
        deviceControlService = (DeviceControlService)
            DeviceDomain.getService("jp.co.olympus.hpf.device.component.deviceservice");
        functionEnablerService = (FunctionEnablerService)
            FunctionEnablerDomain.getService(FunctionEnablerService.ID);

        // Load saved settings
        loadSavedSettings();

        // Register listeners for events you care about
        deviceControlService.addMessageReceivedListener(this::onDeviceMessage);

        // Activate initial states
        functionEnablerService.activate("MY_FEATURE_IDLE_STATE");
    }

    @Override
    public int getPriority() {
        // Choose priority carefully:
        // AcquisitionApplicationHandler = 100 (runs first)
        // DeviceApplicationHandler      = 50
        // Your handler                  = ??? (must run AFTER its dependencies)
        return 30;
    }

    private void loadSavedSettings() {
        // Load from Eclipse preferences or settings file
    }

    private void onDeviceMessage(MessageReceivedEvent event) {
        // React to device messages
    }
}
```

---

## Step 2: Register in plugin.xml

```xml
<extension point="jp.co.olympus.hpf.core.handler">
  <handler class="com.example.MyFeatureApplicationHandler" priority="30"/>
</extension>
```

---

## Step 3: Choose the Right Priority

Priority is **DESCENDING** — higher integer runs **first**.

| Handler | Priority | Runs |
|---|---|---|
| `AcquisitionApplicationHandler` | 100 | First |
| `DeviceApplicationHandler` | 50 | Second |
| Your handler | 30 | After device is ready |
| Low-priority handlers | <30 | Last |

> **Rule:** If your handler depends on another service being initialized, your priority must be **lower** than that service's handler priority.

---

## Step 4: Implement a Component and Service (for larger features)

If your feature is substantial enough to own services, register a full component:

```java
// The component domain
public class MyFeatureDomain extends FVComponentDomain {
    public static final String ID = "my.feature.component";

    @Override
    public void initialize() {
        // Initialize your component's internal state
    }

    public static FVComponentService getService(String serviceId) {
        return FVComponentService.PROVIDER.getService(ID, serviceId);
    }
}

// A service within the component
public class MyFeatureService extends FVComponentService {
    public static final String ID = "my.feature.service";

    public void doSomething() { ... }
}
```

```xml
<!-- Register component -->
<extension point="jp.co.olympus.hpf.core.components">
  <component id="my.feature.component"
             name="My Feature Component"
             domain="com.example.MyFeatureDomain"
             prereqs="jp.co.olympus.hpf.device.component.device,
                      jp.co.olympus.hpf.core.component.message"/>
</extension>

<!-- Register service within component -->
<extension point="jp.co.olympus.hpf.core.services">
  <service id="my.feature.service"
           class="com.example.MyFeatureService"
           componentId="my.feature.component"
           name="My Feature Service"/>
</extension>
```

---

## Step 5: Observe Service State Changes (Optional)

If you need to react when another service changes state:

```java
public class MyServiceObserver extends ServiceObserverAdapter {
    @Override
    public void serviceAvailable(String serviceId) {
        if ("jp.co.olympus.hpf.device.component.deviceservice".equals(serviceId)) {
            // Device service is now available — safe to use it
        }
    }

    @Override
    public void serviceUnavailable(String serviceId) {
        // Handle service going away
    }
}
```

```xml
<extension point="jp.co.olympus.hpf.core.serviceObserver">
  <serviceObserver id="my.observer"
                   class="com.example.MyServiceObserver"/>
</extension>
```

---

## Step 6: Managing Feature States

If your handler needs to gate UI elements by state:

```java
// Activate state (enables linked UI)
functionEnablerService.activate("MY_FEATURE_READY");

// Listen for state changes from other parts of the system.
// NOTE: FunctionConditionChangedListener has THREE methods — a lambda will not compile.
functionEnablerService.addFunctionConditionChangedListener(
    "MY_FUNCTION_ID",
    new FunctionConditionChangedListener() {
        @Override
        public void enableChanged(String functionId, boolean isEnable) {
            updateUI(isEnable);
        }
        @Override
        public void functionConditionChanged(String functionId, Condition condition) { }
        @Override
        public void validationChanged(String functionId, boolean isValid) { }
    }
);

// Defer conflicting states during complex transitions
functionEnablerService.setDeferTargetStates(
    List.of("STATE_A", "STATE_B")
);
```

---

## Common Patterns

### Pattern: Load settings, apply to device, activate state
```java
@Override
public void onPostInitialization() {
    MySettings settings = loadFromPreferences();
    applyToDeviceModel(settings, deviceControlService.getDeviceConfiguration());
    functionEnablerService.activate("MY_FEATURE_IDLING");
    registerMessageListeners();
}
```

### Pattern: React to acquisition state changes
```java
// Listen on a functionId, not a stateId — the engine maps states to function condition.
functionEnablerService.addFunctionConditionChangedListener(
    "F_FV_ACQUISITION_RUNNING",
    new FunctionConditionChangedListener() {
        @Override
        public void enableChanged(String functionId, boolean isEnable) {
            if (isEnable) onAcquisitionStarted();
            else          onAcquisitionStopped();
        }
        @Override
        public void functionConditionChanged(String fId, Condition c) { }
        @Override
        public void validationChanged(String fId, boolean ok) { }
    }
);
```

> **See also:** `how-to/05_function_enabler_feature.md` for end-to-end publisher/subscriber/`*.condition` example.

### Pattern: Dispatch errors correctly
```java
try {
    riskyOperation();
} catch (HPFException e) {
    ErrorDispatchService eds = (ErrorDispatchService)
        MessageDomain.getService(ErrorDispatchService.ID);
    eds.putError(new ErrorEvent(e.getErrorId(), e.getMessage(), e));
}
```
