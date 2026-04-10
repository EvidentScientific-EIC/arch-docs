# How-To: Add a New Analysis Module
**Product:** FluoView / HPF / VPA  
**Last Updated:** April 2026

---

## Overview

Analysis modules are commands in an analysis protocol. Each module type has a `Module` configuration object and an `AnalysisCommandHandler` that executes it. Modules are chained in a sequence via `AnalysisSequenceManagerService`.

---

## Step 1: Create the Plugin

```
jp.co.olympus.fluoview.analysis.{modulename}
jp.co.olympus.fluoview.analysis.{modulename}.ui    (if UI needed)
```

**`META-INF/MANIFEST.MF` dependencies:**
```
Require-Bundle: jp.co.olympus.fluoview.analysis,
 jp.co.olympus.fluoview.analysis.model,
 jp.co.olympus.hpf.data,
 jp.co.olympus.hpf.core
```

---

## Step 2: Define the Module Configuration Class

```java
public class MyAnalysisModule extends Module {

    private double threshold;
    private int channelIndex;

    public MyAnalysisModule() {
        super("MY_ANALYSIS_TYPE");
    }

    @Override
    public void setupModule(FrameContext ctx) {
        this.channelIndex = ctx.getChannelIndex();
        // Initialize from frame context
    }

    @Override
    public ValidationResult validate() {
        ValidationResult result = new ValidationResult();
        if (threshold < 0) {
            result.addError("Threshold must be non-negative");
        }
        return result;
    }

    public double getThreshold() { return threshold; }
    public void setThreshold(double t) { this.threshold = t; }
}
```

---

## Step 3: Implement the Command Handler

```java
public class MyAnalysisCommandHandler implements AnalysisCommandHandler {

    @Override
    public boolean canHandle(String moduleType) {
        return "MY_ANALYSIS_TYPE".equals(moduleType);
    }

    @Override
    public ModuleResult execute(Module module) {
        MyAnalysisModule myModule = (MyAnalysisModule) module;

        try {
            // Perform the analysis
            Object resultData = performAnalysis(myModule);

            // Return result — include any updated ROIs
            return new ModuleResult(module, resultData, Collections.emptyList(), true, null);

        } catch (Exception e) {
            return new ModuleResult(module, null, Collections.emptyList(),
                false, "Analysis failed: " + e.getMessage());
        }
    }

    @Override
    public int getPriority() {
        return 0;
    }

    private Object performAnalysis(MyAnalysisModule module) {
        // Your analysis logic here
        // Access image data via HPFImageModel / FramePlayer
        return new MyAnalysisResult();
    }
}
```

---

## Step 4: Implement the Command Factory Provider

```java
public class MyAnalysisCommandFactoryProvider extends AbstractCommandFactoryProvider {

    @Override
    public List<String> getSupportedTypes() {
        return List.of("MY_ANALYSIS_TYPE");
    }

    @Override
    public CommandFactory createCommandFactory(String type) {
        if ("MY_ANALYSIS_TYPE".equals(type)) {
            return new MyAnalysisCommandFactory();
        }
        return null;
    }
}
```

---

## Step 5: Register via Extension Points

```xml
<!-- Register command factory provider -->
<extension point="jp.co.olympus.fluoview.analysis.CommandFactoryProvider">
  <CommandFactoryProvider class="com.example.MyAnalysisCommandFactoryProvider"/>
</extension>

<!-- Register command handler -->
<extension point="jp.co.olympus.fluoview.analysis.AnalysisCommandHandlerProvider">
  <AnalysisCommandHandlerProvider class="com.example.MyAnalysisCommandHandlerProvider"/>
</extension>
```

---

## Step 6: Add Protocol File Support (Optional)

If your module needs to be configured via an analysis protocol file, add a parser for your module type in the protocol reader. Analysis protocol files define sequences of module types with parameters.

---

## Step 7: Add Results Display (Optional)

Register a graph/display provider for your results:

```java
public class MyAnalysisResultRenderer {
    public void render(MyAnalysisResult result, AnalysisGraphDisplayManager manager) {
        // Draw charts, overlays, tables using the display manager
    }
}
```

---

## Step 8: Write Tests

```java
public class MyAnalysisCommandHandlerTest {

    @Test
    public void testCanHandle() {
        MyAnalysisCommandHandler handler = new MyAnalysisCommandHandler();
        assertTrue(handler.canHandle("MY_ANALYSIS_TYPE"));
        assertFalse(handler.canHandle("OTHER_TYPE"));
    }

    @Test
    public void testExecute() {
        MyAnalysisModule module = new MyAnalysisModule();
        module.setThreshold(0.5);
        
        MyAnalysisCommandHandler handler = new MyAnalysisCommandHandler();
        ModuleResult result = handler.execute(module);
        
        assertTrue(result.isSuccess());
        assertNotNull(result.getData());
    }

    @Test
    public void testValidation() {
        MyAnalysisModule module = new MyAnalysisModule();
        module.setThreshold(-1.0);  // Invalid
        
        ValidationResult validation = module.validate();
        assertFalse(validation.isValid());
        assertFalse(validation.getErrors().isEmpty());
    }
}
```

---

## Reference: Existing Analysis Modules

| Module Type | Handler Class | Plugin |
|---|---|---|
| FRAP | `FRAPCommandHandler` | `hpf.image.frap` |
| FRET | `FRETCommandHandler` | `hpf.image.fret` |
| RRCM Background | `AnalysisBackgroundCommandHandler` | `fluoview.rrcm.analysis` |
| RRCM Calculation | `AnalysisCalculationCommandHandler` | `fluoview.rrcm.analysis` |

Use `hpf.image.frap` as the reference implementation — it is the most complete and tested.
