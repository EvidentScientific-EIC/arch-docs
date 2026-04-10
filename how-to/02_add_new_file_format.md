# How-To: Add a New Image File Format
**Product:** FluoView / HPF / VPA  
**Last Updated:** April 2026

---

## Overview

File formats are registered via the `jp.co.olympus.hpf.data.FileFormat` extension point. You provide three factory classes and a type identifier. The framework discovers your format at startup and uses it automatically for matching file extensions.

---

## Step 1: Create the Plugin

```
jp.co.olympus.{project}.data.format.{formatname}
```

**`META-INF/MANIFEST.MF` dependencies:**
```
Require-Bundle: jp.co.olympus.hpf.data,
 jp.co.olympus.hpf.data.io,
 jp.co.olympus.hpf.core
```

Add to parent `pom.xml` modules list.

---

## Step 2: Implement the Persistent Adapter

This is the main read/write class for your format.

```java
public class MyFormatPersistentAdapter extends HPFObjectAdapterImpl
        implements MultipleImagePersistentAdapter {

    private final HPFImageModel model;

    public MyFormatPersistentAdapter(HPFImageModel model) {
        this.model = model;
    }

    @Override
    public void read() throws HPFException {
        // Deserialize file → populate model
        // Use model.getMetainfo(), model.addFrame(), etc.
        try (InputStream in = new FileInputStream(getFilePath())) {
            // parse your format
        } catch (IOException e) {
            throw new HPFException(HPFError.DATA_READ_FAILED, e.getMessage());
        }
    }

    @Override
    public void write() throws HPFException {
        // Serialize model → write file
        try (OutputStream out = new FileOutputStream(getFilePath())) {
            // write your format
        } catch (IOException e) {
            throw new HPFException(HPFError.DATA_WRITE_FAILED, e.getMessage());
        }
    }

    @Override
    public ImageBlockData readFrame(int frameIndex) {
        // Read and return a single frame's pixel data
        return new ImageBlockData(/* ByteBuffer */);
    }

    @Override
    public void writeFrame(int frameIndex, ImageBlockData data) {
        // Write a single frame's pixel data
    }

    @Override
    public int getFrameCount() {
        return model.getFrameCount();
    }
}
```

---

## Step 3: Implement the Factory

```java
public class MyFormatPersistentAdapterFactory implements AbstractPersistentAdapterFactory {

    @Override
    public String getExtension() {
        return ".myfmt";  // The file extension your format uses
    }

    @Override
    public PersistentAdapter createAdapter(HPFImageModel model) {
        return new MyFormatPersistentAdapter(model);
    }
}
```

---

## Step 4: Implement the Frame Index Factory

Controls how frames are ordered and indexed during save:

```java
public class MyFormatSaveFrameIndexListFactory implements AbstractSaveFrameIndexListFactory {

    @Override
    public SaveFrameIndexList createFactory() {
        return new DefaultSaveFrameIndexList(); // or custom ordering
    }
}
```

---

## Step 5: Implement the Information Utility

Provides format metadata for the UI:

```java
public class MyFormatInfomationUtil implements InfomationUtil {

    @Override
    public FormatInfo getFormatInfo() {
        return new FormatInfo("My Format", ".myfmt", "Description of format");
    }
}
```

---

## Step 6: Register via Extension Point

```xml
<!-- In plugin.xml -->
<extension point="jp.co.olympus.hpf.data.FileFormat">
  <FileFormat type="MYFORMAT"
              PersistentAdapterFactory="com.example.MyFormatPersistentAdapterFactory"
              AbstractSaveFrameIndexListFactory="com.example.MyFormatSaveFrameIndexListFactory"
              InfomationUtil="com.example.MyFormatInfomationUtil"/>
</extension>
```

The `type` string becomes the `FormatType` identifier used throughout the system.

---

## Step 7: Register a File Reader Selector (Optional)

If your format needs custom file detection logic beyond extension matching:

```java
public class MyFormatFileReaderSelector implements FileReaderSelector {
    @Override
    public boolean canRead(IPath filePath) {
        // Return true if this format can read the file
        return filePath.getFileExtension().equals("myfmt") ||
               isMyFormatMagicBytes(filePath);
    }
}
```

```xml
<extension point="jp.co.olympus.hpf.data.fileReaderSelector">
  <fileReaderSelector class="com.example.MyFormatFileReaderSelector"/>
</extension>
```

---

## Step 8: Add Property Extraction (Optional)

To extract metadata properties from files of your format:

```java
public class MyFormatPropertyExtractor implements PropertyExtractor {
    @Override
    public Map<String, Object> extract(IPath filePath) {
        // Parse and return metadata (dimensions, channels, timestamps, etc.)
        return Map.of("width", 1024, "height", 1024, "channels", 3);
    }
}
```

```xml
<extension point="jp.co.olympus.hpf.data.propertyExtractor">
  <propertyExtractor class="com.example.MyFormatPropertyExtractor"/>
</extension>
```

---

## Step 9: Write Tests

```java
public class MyFormatPersistentAdapterTest {
    @Test
    public void testRoundTrip() throws Exception {
        HPFImageModel model = createTestModel();
        MyFormatPersistentAdapter adapter = new MyFormatPersistentAdapter(model);
        
        // Write
        adapter.write();
        
        // Read back
        HPFImageModel readModel = createEmptyModel();
        MyFormatPersistentAdapter readAdapter = new MyFormatPersistentAdapter(readModel);
        readAdapter.read();
        
        assertEquals(model.getFrameCount(), readModel.getFrameCount());
        // assert metadata equality
    }
}
```

---

## Existing Format Reference

| Format | Factory Class | Extension |
|---|---|---|
| OIF | `OIFPersistentAdapterFactory` | `.oif` |
| OIB | `OIBPersistentAdapterFactory` | `.oib` |
| IDA (VSI) | `IdaPersistentAdapterFactory` | `.vsi` |
| IDA2 | `Ida2PersistentAdapterFactory` | `.ida` |
| OIR (Raw) | `RawPersistentAdapterFactory` | `.oir` |
| HDF5 | `MultiResolutionPersistentAdapterFactory` | `.hdf5` |

Use the IDA or OIF adapters as reference implementations — they are the most complete.
