# How-To: Add a Feature Gated by `FunctionEnablerService`
**Product:** FluoView / HPF / VPA
**Last Updated:** April 2026

This guide shows the **publisher → rules → subscriber** pattern end-to-end. Use it whenever you add a UI feature whose enable/disable depends on system state (acquisition state, license, device status, parameter validity, etc.).

> **Mental model.** Think of `FunctionEnablerService` as a pub/sub system whose wiring is declarative: publishers shout facts about the world (`activate(stateId)`), subscribers listen for verdicts about their feature (`functionId`), and `*.condition` XML is the contract between them.

---

## The example: a "Start Series Scan" toolbar button

The button should be:

- **Enabled** when system is idle, camera connected, license present, parameters valid
- **Disabled** during a running scan or when a device error is active
- Re-evaluated automatically — without the button knowing about cameras, licenses, or scans

Three pieces wire this up: the rule file, the publisher, the subscriber.

---

## Piece 1 — The `.condition` rule file (declarative)

Ship as a resource in your plugin (e.g. `resources/series-scan.condition`). The core service auto-discovers any file with the `.condition` extension at boot.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<conditionTable xmlns="http://www.olympus.co.jp/hpf/enabler">

    <function id="F_FV_SERIES_SCAN_START">

        <enableWhen>
            <stateActive id="S_PF_LSM_IMAGING_IDLING"/>
            <stateActive id="S_PF_CAMERA_CONNECTED"/>
            <stateActive id="S_PF_LICENSE_SERIES_SCAN"/>
            <stateActive id="S_PF_SERIES_PARAMS_VALID"/>
        </enableWhen>

        <disableWhen>
            <stateActive id="S_PF_LSM_SERIES_SCANNING"/>
            <stateActive id="S_PF_DEVICE_ERROR"/>
        </disableWhen>

    </function>
</conditionTable>
```

**What this does:** declares that `F_FV_SERIES_SCAN_START` is only clickable when all four enable states are simultaneously active, force-disabled during scan or error. No Java required.

---

## Piece 2 — The publisher (states get activated)

Lives inside your domain code (acquisition, device, license). It does **not** know about the UI.

```java
package jp.co.olympus.fluoview.acquisition.enabler;

import jp.co.olympus.hpf.core.enabler.component.FunctionEnablerDomain;
import jp.co.olympus.hpf.core.enabler.component.FunctionEnablerService;

public class SeriesScanStatePublisher {

    private static final String S_IDLING     = "S_PF_LSM_IMAGING_IDLING";
    private static final String S_SCANNING   = "S_PF_LSM_SERIES_SCANNING";
    private static final String S_CAM_OK     = "S_PF_CAMERA_CONNECTED";
    private static final String S_CAM_GONE   = "S_PF_CAMERA_DISCONNECTED";
    private static final String S_LIC_SERIES = "S_PF_LICENSE_SERIES_SCAN";
    private static final String S_PARAMS_OK  = "S_PF_SERIES_PARAMS_VALID";
    private static final String S_DEV_ERR    = "S_PF_DEVICE_ERROR";

    private final FunctionEnablerService enabler;

    public SeriesScanStatePublisher() {
        this.enabler = (FunctionEnablerService)
            FunctionEnablerDomain.getService(FunctionEnablerService.ID);
    }

    public void onScanStarted()                { enabler.activate(S_SCANNING); }
    public void onScanStopped()                { enabler.activate(S_IDLING); }
    public void onCameraConnectionChanged(boolean connected) {
        enabler.activate(connected ? S_CAM_OK : S_CAM_GONE);
    }
    public void onParamsValidated(boolean valid) {
        enabler.activate(valid ? S_PARAMS_OK : "S_PF_SERIES_PARAMS_INVALID");
    }
    public void onLicenseInitialized(boolean has) {
        if (has) enabler.activate(S_LIC_SERIES);
        // No license → never activated → function stays disabled forever.
    }
    public void onDeviceError() { enabler.activate(S_DEV_ERR); }
}
```

**Key idea:** the publisher reports facts about the world. It does not enumerate which UI features are affected.

---

## Piece 3 — The subscriber (UI button reacts)

```java
package jp.co.olympus.fluoview.acquisition.ui.toolbar;

import jp.co.olympus.hpf.core.enabler.component.FunctionEnablerDomain;
import jp.co.olympus.hpf.core.enabler.component.FunctionEnablerService;
import jp.co.olympus.hpf.core.engine.enabler.condition.Condition;
import jp.co.olympus.hpf.core.engine.enabler.condition.FunctionConditionChangedListener;
import jp.co.olympus.hpf.ui.utils.SWTHelper;
import org.eclipse.swt.SWT;
import org.eclipse.swt.events.SelectionAdapter;
import org.eclipse.swt.events.SelectionEvent;
import org.eclipse.swt.widgets.Button;
import org.eclipse.swt.widgets.Composite;

public class SeriesScanButton {

    private static final String FUNCTION_ID = "F_FV_SERIES_SCAN_START";

    private final FunctionEnablerService enabler;
    private final Button button;
    private final FunctionConditionChangedListener listener;

    public SeriesScanButton(Composite parent, Runnable startScanAction) {
        this.enabler = (FunctionEnablerService)
            FunctionEnablerDomain.getService(FunctionEnablerService.ID);

        this.button = new Button(parent, SWT.PUSH);
        this.button.setText("Start Series Scan");
        this.button.addSelectionListener(new SelectionAdapter() {
            @Override
            public void widgetSelected(SelectionEvent e) {
                // Belt-and-braces: only act when stable, not mid-transition.
                if (enabler.isAvailableAndStable(FUNCTION_ID)) {
                    startScanAction.run();
                }
            }
        });

        // The listener has THREE callbacks — a lambda will not compile.
        this.listener = new FunctionConditionChangedListener() {
            @Override
            public void enableChanged(final String functionId, final boolean isEnable) {
                // Notifications come from an async dispatcher → marshal to SWT.
                SWTHelper.execSyncInDisplayThread(button, () -> {
                    if (!button.isDisposed()) button.setEnabled(isEnable);
                });
            }
            @Override
            public void functionConditionChanged(String fId, Condition c) { /* not needed for a plain button */ }
            @Override
            public void validationChanged(String fId, boolean ok)        { /* not needed for a plain button */ }
        };

        enabler.addFunctionConditionChangedListener(FUNCTION_ID, listener);

        // Listener fires only on CHANGE — set initial state explicitly.
        button.setEnabled(enabler.isEnable(FUNCTION_ID));

        // Always unsubscribe on dispose.
        button.addDisposeListener(e ->
            enabler.removeFunctionConditionChangedListener(FUNCTION_ID, listener));
    }
}
```

---

## Wiring sequence (boot → runtime)

```
Boot (HPFRoot):
  └─ FunctionEnablerServiceImpl loads every *.condition file
  └─ Acquisition domain creates SeriesScanStatePublisher
  └─ License/device handlers fire onLicenseInitialized / onCameraConnectionChanged
  └─ Acquisition fires onScanStopped → activate(S_PF_LSM_IMAGING_IDLING)

Perspective opens:
  └─ SeriesScanButton constructed
       ├─ addFunctionConditionChangedListener(...)
       └─ button.setEnabled(enabler.isEnable(FUNCTION_ID))   ← initial state

Runtime:
  T=0  user validates params → onParamsValidated(true)
       └─ activate(S_PF_SERIES_PARAMS_VALID)
       └─ engine recomputes → ENABLED
       └─ enableChanged(..., true) → button on

  T=1  user clicks → startScanAction.run() → onScanStarted()
       └─ activate(S_PF_LSM_SERIES_SCANNING)
       └─ disableWhen hit → DISABLED → button off

  T=2  scan ends → onScanStopped()
       └─ activate(S_PF_LSM_IMAGING_IDLING) (S_PF_LSM_SERIES_SCANNING auto-cleared)
       └─ ENABLED → button on
```

---

## Suppressing flicker through transitional states

When you pass through a brief intermediate state, listeners would otherwise see `enabled → disabled → enabled → disabled`. Wrap with `setDeferTargetStates`:

```java
public void beginScan() {
    enabler.setDeferTargetStates(List.of("S_PF_LSM_TRANSITIONING"));
    try {
        enabler.activate("S_PF_LSM_TRANSITIONING");
        doAllTheHardwarePrep();
        enabler.activate("S_PF_LSM_SERIES_SCANNING");
    } finally {
        enabler.setDeferTargetStates(List.of());   // release
    }
}
```

Result: listeners see exactly one transition.

---

## Testing

The framework ships a stub at `jp.co.olympus.hpf.core.test.util/FunctionEnablerServiceStub`.
Real-world example: `MATLShiftZSettingsDialogTest` (line 54) drives `FunctionID.F_MATL_SHIFT_Z` directly:

```java
@Test
public void buttonEnabledWhenFunctionEnabled() {
    SeriesScanButton subject = new SeriesScanButton(shell, () -> {});
    stub.fireEnableChanged("F_FV_SERIES_SCAN_START", true);
    assertTrue(subject.getButton().isEnabled());
}
```

---

## Quick map: what you change for which task

| Task | Files you touch |
|---|---|
| Add a button to an existing function | Subscriber only (Piece 3); pick an existing `functionId` from a `.condition` file |
| Add a new feature with new enable rules | New `.condition` file (Piece 1) + subscriber (Piece 3) |
| Add a new state input that should affect existing features | New publisher call (Piece 2) + edit the relevant `.condition` files to reference the new `stateId` |
| Suppress flicker during a multi-step transition | Wrap with `setDeferTargetStates` |
| Forbid an illegal state combination | `addStateTransitionValidator(StateTransitionValidator)` |

---

## Common mistakes

1. **Using `MessageDomain.getService(...)` to look up `FunctionEnablerService`.** The correct domain is `FunctionEnablerDomain`. Only `ErrorDispatchService` and the legacy `MessageDeliveryService` belong to `MessageDomain`.
2. **Lambda for `FunctionConditionChangedListener`.** Three methods → not a functional interface. Use an anonymous class.
3. **Forgetting the initial state.** The listener fires only on **change**. After subscribing, set the initial enabled state by calling `enabler.isEnable(functionId)` once.
4. **Forgetting to unsubscribe.** Wire `removeFunctionConditionChangedListener` to the widget's dispose listener.
5. **Calling `setEnabled` from a non-UI thread.** Always marshal via `SWTHelper.execSyncInDisplayThread`.
6. **Direct `button.setEnabled(...)` based on subsystem state.** That's exactly what the enabler exists to centralize. Add a state and a rule instead.

---

## Cross-references

- Service catalogue: `reference/03_interface_service_catalogue.md` ▸ `FunctionEnablerService`
- Glossary: `reference/01_domain_glossary.md` ▸ FunctionEnablerService, StateID, functionId, Condition, `*.condition` file, FunctionConditionChangedListener, DeferTargetStates
- Tech debt: `architecture/02_technical_debt_register.md` (UI framework category)
- UI framework patterns: `how-to/06_ui_framework_patterns.md`
