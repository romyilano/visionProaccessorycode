# Troubleshooting Guide

## 🚨 Common Issues and Solutions

### Build and Compilation Issues

#### "No such module 'RealityKit'"

**Problem**: Xcode can't find RealityKit framework

**Solutions**:
1. Ensure you're targeting visionOS, not iOS:
   - Select project in navigator
   - Choose target → General tab
   - Set "Supported Destinations" to include visionOS

2. Update Xcode to latest version:
   ```bash
   # Check Xcode version
   xcodebuild -version
   # Should be 15.0 or later
   ```

3. Install visionOS SDK:
   - Xcode → Settings → Platforms
   - Download visionOS SDK

#### "Cannot find type 'SpatialTrackingSession' in scope"

**Problem**: Missing ARKit types

**Solution**: 
```swift
// Add to your file
import ARKit
```

#### Metal Shader Compilation Errors

**Problem**: Shaders fail to compile

**Debug Steps**:
1. Check shader syntax in `.metal` files
2. Verify bridging header includes Metal headers:
   ```objc
   // BridgingHeader.h
   #include "MarchingCubesParams.h"
   #include "VolumeParams.h"
   #include "SculptParams.h"
   ```

3. Clean build folder:
   - Product → Clean Build Folder (Shift+Cmd+K)

### Runtime Issues

#### App Launches but No Mesh Appears

**Possible Causes**:

1. **Compute pipeline failed to initialize**
   ```swift
   // Add debug logging
   guard let pipeline = makeComputePipeline(named: "march") else {
       print("Failed to create marching cubes pipeline")
       return
   }
   ```

2. **Voxel volume creation failed**
   ```swift
   // Check initialization
   guard let voxelVolume = try? VoxelVolume(...) else {
       print("Failed to create voxel volume")
       return
   }
   ```

3. **GPU texture allocation failed**
   - Reduce volume size for testing:
   ```swift
   let dimensions = SIMD3<UInt32>(64, 64, 64) // Smaller for testing
   ```

#### Tracking Not Working

**Check List**:

1. **Privacy permissions**:
   - Info.plist must include:
   ```xml
   <key>NSHandsTrackingUsageDescription</key>
   <string>This app uses hand tracking for spatial input</string>
   ```

2. **Device connection**:
   ```swift
   // Debug accessory connection
   print("Controllers: \(GCController.controllers().count)")
   print("Styluses: \(GCStylus.styli.count)")
   ```

3. **Tracking session status**:
   ```swift
   // Monitor tracking state
   if trackingState == .untracked {
       print("WARNING: Accessory not tracked")
   }
   ```

#### Poor Performance / Low Frame Rate

**Optimization Steps**:

1. **Profile with Instruments**:
   - Product → Profile (Cmd+I)
   - Choose "Metal System Trace"
   - Look for GPU bottlenecks

2. **Reduce voxel resolution**:
   ```swift
   // Lower resolution for better performance
   let dimensions = SIMD3<UInt32>(64, 64, 64)
   ```

3. **Optimize shader code**:
   ```metal
   // Use fast math functions
   float dist = fast::length(position - center);
   ```

4. **Limit mesh updates**:
   ```swift
   // Throttle updates
   if Date().timeIntervalSince(lastUpdate) < 0.016 { return }
   ```

### UI and Interaction Issues

#### Toolbar Not Appearing

**Debug Steps**:

1. **Check button handler**:
   ```swift
   buttonPrimary?.pressedInput.pressedDidChangeHandler = { _, _, pressed in
       print("Palette button pressed: \(pressed)")
       self.handlePalettePress(pressed: pressed)
   }
   ```

2. **Verify entity setup**:
   ```swift
   // Ensure icons are created
   guard let additiveIcon = additiveIcon else {
       print("ERROR: Additive icon not created")
       return
   }
   ```

3. **Check collision components**:
   ```swift
   entity.components.set(CollisionComponent(
       shapes: [.generateBox(size: .init(repeating: 0.05))]
   ))
   ```

#### Raycast Selection Not Working

**Debug raycast**:
```swift
// Visualize ray
func debugRaycast(origin: SIMD3<Float>, direction: SIMD3<Float>) {
    let debugEntity = ModelEntity(
        mesh: .generateCylinder(height: 10, radius: 0.001),
        materials: [SimpleMaterial(color: .red)]
    )
    debugEntity.position = origin + direction * 5
    debugEntity.look(at: origin, from: debugEntity.position, relativeTo: nil)
    scene.addChild(debugEntity)
}
```

### Save/Load Issues

#### Cannot Save Sculpture

**Check**:

1. **File permissions**:
   ```swift
   // Verify save location is writable
   let documentsPath = FileManager.default.urls(
       for: .documentDirectory,
       in: .userDomainMask
   ).first!
   ```

2. **Texture data validation**:
   ```swift
   // Ensure texture has data
   let bytesPerRow = texture.width * 4
   let imageBytes = texture.height * texture.depth * bytesPerRow
   print("Texture size: \(imageBytes) bytes")
   ```

#### Corrupted Save Files

**Validation**:
```swift
// Add file validation
func validateSaveFile(data: Data) -> Bool {
    // Check minimum size
    let expectedSize = dimensions.x * dimensions.y * dimensions.z * 4
    return data.count == expectedSize
}
```

### Metal and GPU Issues

#### GPU Command Buffer Errors

**Common Errors**:

1. **"Execution of command buffer was aborted"**
   - Usually means shader timeout
   - Reduce workload or optimize shaders

2. **"Invalid texture access"**
   - Check texture bounds in shaders:
   ```metal
   if (any(id >= uint3(params.dimensions))) {
       return;
   }
   ```

3. **"Resource shortage"**
   - Too many resources allocated
   - Release unused textures/buffers

#### Debugging Metal Shaders

**Enable Metal Validation**:
1. Edit Scheme → Run → Diagnostics
2. Enable "Metal API Validation"
3. Enable "Metal Shader Validation"

**Add debug output**:
```metal
// Debug values to texture
kernel void debugKernel(
    texture3d<float, access::write> debugTexture [[texture(0)]],
    uint3 id [[thread_position_in_grid]])
{
    // Write debug color based on thread ID
    float3 color = float3(id) / float3(128);
    debugTexture.write(float4(color, 1.0), id);
}
```

### Performance Profiling

#### Using GPU Frame Capture

1. Run app on device
2. In Xcode, click "M" button in debug bar
3. Click "Capture GPU Frame"
4. Analyze:
   - Texture memory usage
   - Shader execution time
   - Draw call count

#### Key Metrics to Monitor

```swift
// Add performance monitoring
class PerformanceMonitor {
    var frameCount = 0
    var lastTime = CACurrentMediaTime()
    
    func update() {
        frameCount += 1
        let currentTime = CACurrentMediaTime()
        if currentTime - lastTime >= 1.0 {
            print("FPS: \(frameCount)")
            frameCount = 0
            lastTime = currentTime
        }
    }
}
```

### Haptic Feedback Issues

#### No Haptic Feedback

**Debug**:
```swift
// Check haptic support
if !CHHapticEngine.capabilitiesForHardware().supportsHaptics {
    print("Device doesn't support haptics")
}

// Verify engine started
do {
    try hapticEngine?.start()
} catch {
    print("Haptic engine failed to start: \(error)")
}
```

## 🛠 Debug Tools and Techniques

### Console Logging

Add strategic logging:
```swift
// Component updates
print("📍 Tool position: \(position)")
print("🎨 Mode: \(mode), Radius: \(radius)")
print("📊 Tracking: \(trackingState)")
```

### Visual Debugging

```swift
// Show coordinate axes
func createDebugAxes() -> Entity {
    let axes = Entity()
    
    // X axis (red)
    let xAxis = ModelEntity(
        mesh: .generateCylinder(height: 1, radius: 0.01),
        materials: [SimpleMaterial(color: .red)]
    )
    xAxis.orientation = simd_quatf(angle: .pi/2, axis: [0,0,1])
    
    // Y axis (green)
    let yAxis = ModelEntity(
        mesh: .generateCylinder(height: 1, radius: 0.01),
        materials: [SimpleMaterial(color: .green)]
    )
    
    // Z axis (blue)
    let zAxis = ModelEntity(
        mesh: .generateCylinder(height: 1, radius: 0.01),
        materials: [SimpleMaterial(color: .blue)]
    )
    zAxis.orientation = simd_quatf(angle: .pi/2, axis: [1,0,0])
    
    axes.children.append(contentsOf: [xAxis, yAxis, zAxis])
    return axes
}
```

### Memory Debugging

Enable memory debugging:
1. Edit Scheme → Run → Diagnostics
2. Enable "Malloc Guard Edges"
3. Enable "Address Sanitizer"

## 📋 Checklist for Bug Reports

When reporting issues, include:

1. **System Information**
   - macOS version
   - Xcode version
   - Device model
   - visionOS version

2. **Steps to Reproduce**
   - Exact sequence of actions
   - Input device used
   - Expected vs actual behavior

3. **Error Messages**
   - Console output
   - Crash logs
   - Metal validation errors

4. **Code Changes**
   - Modified files
   - Custom shaders
   - Configuration changes

## 🔗 Additional Resources

- [Apple Developer Forums](https://developer.apple.com/forums/tags/visionos)
- [Metal Debugging Guide](https://developer.apple.com/documentation/metal/debugging_tools)
- [RealityKit Troubleshooting](https://developer.apple.com/documentation/realitykit/troubleshooting)
- [Instruments User Guide](https://help.apple.com/instruments/mac/current/)

## 💡 Prevention Tips

1. **Always check return values** from initialization
2. **Use guard statements** for optional unwrapping
3. **Add error handling** to async operations
4. **Profile regularly** during development
5. **Test on real hardware** early and often

Remember: Most issues stem from resource constraints, initialization failures, or coordinate system mismatches. Start with the basics and work your way up!
