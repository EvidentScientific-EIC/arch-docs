# How-To: Add a New Hardware Device
**Product:** FluoView / HPF / VPA  
**Last Updated:** April 2026

---

## Overview

Adding a new hardware device involves:
1. Defining a device configuration model (optional — if the device has configurable parameters)
2. Creating device communication handlers
3. Registering with the device control extension points
4. Wiring into the acquisition pipeline

---

## Step 1: Create the Plugin

Create a new OSGi plugin following existing naming:
```
jp.co.olympus.fluoview.device.{devicename}
jp.co.olympus.fluoview.device.{devicename}.ui        (if UI needed)
jp.co.olympus.fluoview.acquisition.{devicename}      (acquisition integration)
```

**`META-INF/MANIFEST.MF` minimum dependencies:**
```
Require-Bundle: jp.co.olympus.hpf.core,
 jp.co.olympus.hpf.device,
 jp.co.olympus.hpf.device.model,
 jp.co.olympus.fluoview.device
```

**Add to parent `pom.xml`:**
```xml
<module>plugins/jp.co.olympus.fluoview.device.{devicename}</module>
```

---

## Step 2: Define Device Parameters (if needed)

If the device has configurable parameters, add them to the EMF model in `hpf.device.model`:

1. Open `genmodel/configuration.ecore` in Eclipse EMF editor
2. Add new EClass for your device parameters (e.g., `MyDeviceParam`)
3. Add fields as EAttributes
4. Regenerate model code: right-click `.genmodel` → Generate Model Code

Alternatively, create a standalone Java class for simpler parameter sets.

---

## Step 3: Implement Device Communication

Create a class that sends commands via `DeviceControlService`:

```java
public class MyDeviceController {
    private final DeviceControlService deviceControlService;

    public MyDeviceController(DeviceControlService dcs) {
        this.deviceControlService = dcs;
    }

    public void setParameter(int value) {
        DeviceCommand cmd = new DeviceCommand("MY_DEVICE", "SET_PARAM", 
            Map.of("value", value));
        CommandResult result = deviceControlService.sendCommand(cmd);
        if (!result.isSuccess()) {
            throw new HPFException(HPFError.DEVICE_COMMAND_FAILED, result.getErrorCode());
        }
    }
}
```

---

## Step 4: Register as a Device Controller (External Process)

If your device is controlled via an external executable:

```xml
<!-- In plugin.xml -->
<extension point="jp.co.olympus.hpf.device.deviceController">
  <deviceController name="MyDevice"
                    path="devices/mydevice"
                    process="mydevice_controller.exe"
                    plugin="jp.co.olympus.fluoview.device.mydevice"
                    paramsGenerator="com.example.MyDeviceParamsGenerator"/>
</extension>
```

---

## Step 5: Register a Command Target Filter

To ensure device commands are routed only to your device:

```java
public class MyDeviceCommandTargetFilter implements CommandTargetFilter {
    @Override
    public boolean accept(DeviceCommand command) {
        return "MY_DEVICE".equals(command.getTargetDevice());
    }
}
```

```xml
<extension point="jp.co.olympus.hpf.device.defaultCommandTargetFilter">
  <defaultCommandTargetFilter class="com.example.MyDeviceCommandTargetFilter"/>
</extension>
```

---

## Step 6: Integrate with Acquisition Pipeline

To apply device parameters during acquisition setup:

### 6a. Implement `LoadParametersExtender`
Called to enrich `ObservationSettings` before applying to hardware:

```java
public class MyDeviceLoadParametersExtender implements LoadParametersExtender {
    @Override
    public void extend(ObservationSettings settings, DeviceConfiguration config) {
        // Read settings, populate device-specific parameters into config
        MyDeviceParam param = config.getMyDeviceParam();
        param.setPower(settings.getMyDevicePower());
    }
}
```

```xml
<extension point="jp.co.olympus.fluoview.acquisition.loadParametersExtender">
  <loadParametersExtender class="com.example.MyDeviceLoadParametersExtender"/>
</extension>
```

### 6b. Implement an `ApplyParametersCommandGenerator`
Called to generate hardware commands from parameters:

```java
public class MyDeviceCommandGenerator implements ApplyParametersCommandGenerator {
    @Override
    public List<DeviceCommand> generate(DeviceConfiguration config) {
        MyDeviceParam param = config.getMyDeviceParam();
        return List.of(new DeviceCommand("MY_DEVICE", "SET_POWER",
            Map.of("power", param.getPower())));
    }
}
```

```xml
<extension point="jp.co.olympus.fluoview.acquisition.applyParametersCommandGenerator">
  <applyParametersCommandGenerator class="com.example.MyDeviceCommandGenerator"/>
</extension>
```

---

## Step 7: Add an ApplicationRootHandler for Initialization

```java
public class MyDeviceApplicationHandler implements ApplicationRootHandler {
    @Override
    public void onPostInitialization() {
        DeviceControlService dcs = DeviceDomain.getService("...");
        // Initialize device, load saved parameters, register listeners
    }

    @Override
    public int getPriority() {
        return 40; // Lower than AcquisitionApplicationHandler (100)
    }
}
```

```xml
<extension point="jp.co.olympus.hpf.core.handler">
  <handler class="com.example.MyDeviceApplicationHandler" priority="40"/>
</extension>
```

---

## Step 8: Write Tests

Create `jp.co.olympus.fluoview.device.{devicename}.test` in `test/plugins/`:

```java
public class MyDeviceControllerTest {
    @Test
    public void testSetParameter() {
        DeviceControlService mockDcs = mock(DeviceControlService.class);
        when(mockDcs.sendCommand(any())).thenReturn(new CommandResult(true, null, null));
        
        MyDeviceController controller = new MyDeviceController(mockDcs);
        controller.setParameter(42);
        
        verify(mockDcs).sendCommand(argThat(cmd ->
            "MY_DEVICE".equals(cmd.getTargetDevice()) &&
            "SET_PARAM".equals(cmd.getCommandType())));
    }
}
```

---

## Checklist

- [ ] Plugin created with correct naming
- [ ] Added to `pom.xml` modules
- [ ] `MANIFEST.MF` dependencies correct
- [ ] Device parameters modelled (EMF or POJO)
- [ ] Command target filter registered
- [ ] `LoadParametersExtender` registered
- [ ] `ApplyParametersCommandGenerator` registered
- [ ] `ApplicationRootHandler` registered with appropriate priority
- [ ] Error handling uses `HPFException` + `ErrorDispatchService`
- [ ] Test plugin created and passing
- [ ] `.license` stub plugin created if feature is license-gated
