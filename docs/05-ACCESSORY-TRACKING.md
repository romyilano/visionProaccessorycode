# Spatial Accessory Tracking Guide

## 🎮 What are Spatial Accessories?

Spatial accessories are physical devices tracked in 3D space by visionOS:
- **Position**: Where the device is in 3D space
- **Orientation**: Which direction it's pointing
- **Buttons**: Physical controls for interaction
- **Haptics**: Tactile feedback to the user

### Supported Devices
- Logitech MX Ink stylus
- Future spatial game controllers
- Compatible input devices

## 🏗 Architecture Overview

The app uses three frameworks together:

```
ARKit (SpatialTrackingSession)
    ↓
GameController Framework
    ↓
RealityKit (AnchorEntity)
```

## 🚀 Tracking Setup

### Step 1: Configure Tracking Session

```swift
// ContentView.swift
.task {
    let configuration = SpatialTrackingSession.Configuration(tracking: [.accessory])
    let session = SpatialTrackingSession()
    await session.run(configuration)
}
```

### Step 2: Handle Device Connections

```swift
// SculptingToolModel+GameController.swift
func handleGameControllerSetup(hapticsModel: HapticsModel) async {
    // Check already connected devices
    let controllers = GCController.controllers()
    let styluses = GCStylus.styli
    
    // Setup each spatial device
    for controller in controllers {
        if controller.productCategory == GCProductCategorySpatialController {
            try? await setupSpatialAccessory(device: controller, hapticsModel: hapticsModel)
        }
    }
    
    // Listen for new connections
    NotificationCenter.default.addObserver(
        forName: .GCControllerDidConnect,
        object: nil,
        queue: nil
    ) { notification in
        // Handle new device
    }
}
```

## 📍 Creating Tracking Anchors

### The Anchoring Process

```swift
// SculptingToolModel+Anchoring.swift
func setupSpatialAccessory<T: GCSpatialAccessory>(
    device: T, 
    hapticsModel: HapticsModel
) async throws {
    // 1. Create anchor for tracking
    let anchor = device.objectAnchor
    
    // 2. Create RealityKit entity
    sculptingEntity = AnchorEntity(anchor)
    
    // 3. Add tracking indicator
    let indicator = createTrackingIndicator()
    sculptingEntity.addChild(indicator)
    trackingStateIndicator = indicator
    
    // 4. Add to scene
    rootEntity?.addChild(sculptingEntity)
    
    // 5. Setup input handling
    if let stylus = device as? GCStylus {
        setupStylusInputs(stylus: stylus, hapticsModel: hapticsModel)
    }
}
```

## 🎯 Input Handling

### Button Mapping

For stylus devices:
```swift
// Primary button (palette)
buttonSidePrimary?.pressedInput.pressedDidChangeHandler = { _, _, pressed in
    self.handlePalettePress(pressed: pressed)
}

// Secondary button (sculpt)
buttonSideSecondary?.pressedInput.pressedDidChangeHandler = { _, _, pressed in
    self.sculptingTool.components[SculptingToolComponent.self]?.isActive = pressed
    hapticsModel.handleSculptHaptics(pressed: pressed)
}
```

For game controllers:
```swift
// Trigger button (sculpt)
input.buttons[.trigger]?.pressedInput.pressedDidChangeHandler = { _, _, pressed in
    self.sculptingTool.components[SculptingToolComponent.self]?.isActive = pressed
}

// Thumbstick button (palette)
input.buttons[.thumbstickButton]?.pressedInput.pressedDidChangeHandler = { _, _, pressed in
    self.handlePalettePress(pressed: pressed)
}
```

## 🔄 Tracking States

### Understanding Tracking Quality

```swift
enum TrackingState {
    case untracked           // No tracking data
    case positionTracked     // 3DOF - position only
    case positionOrientationTracked  // 6DOF - full tracking
}
```

### Visual Feedback

```swift
let trackingStateColor: [TrackingState: UIColor] = [
    .untracked: .red,
    .positionTracked: .orange,
    .positionOrientationTracked: .green
]
```

The app shows a colored indicator when tracking degrades.

## 📊 Transform Updates

### Every Frame Update

```swift
// ContentView.swift
_ = content.subscribe(to: SceneEvents.Update.self) { _ in
    sculpting.updateSculptingTool()
}

// SculptingToolModel.swift
func updateSculptingTool() {
    guard let sculptingEntity = sculptingEntity else { return }
    guard let rootEntity = rootEntity else { return }
    
    // Get accessory transform relative to scene
    guard let matrix = try? sculptingEntity.transform(from: rootEntity) else { return }
    
    // Update tool position
    sculptingTool.transform = Transform(matrix: simd_float4x4(matrix))
    
    // Check tracking state
    updateTrackingStateIndicatorIfDirty(sculptingEntity: sculptingEntity)
}
```

## 🎨 Toolbar Interaction

### Raycast Selection

When the palette button is pressed:

```swift
func selectToolbarElement(sculptingEntity: AnchorEntity) {
    guard let scene = sculptingEntity.scene else { return }
    
    // Cast ray from accessory
    let raycastOrigin = sculptingEntity.position(relativeTo: nil)
    let raycastForward = -simd_make_float3(
        sculptingEntity.transformMatrix(relativeTo: nil).columns.2
    )
    
    // Check what we hit
    if let hit = scene.raycast(
        origin: raycastOrigin,
        direction: raycastForward,
        length: 10
    ).first {
        switch hit.entity.name {
        case "AdditiveIcon":
            sculptingTool.components[SculptingToolComponent.self]?.mode = .add
        case "SubtractiveIcon":
            sculptingTool.components[SculptingToolComponent.self]?.mode = .remove
        // ... handle other icons
        }
    }
}
```

## 💫 Haptic Feedback

### Setup Haptics Engine

```swift
// HapticsModel+Haptics.swift
func setupHaptics(haptics: GCDeviceHaptics) {
    self.haptics = haptics
    
    // Create haptic engine
    hapticEngine = haptics.createEngine(withLocality: .rightTrigger)
    
    // Load haptic patterns
    let pattern = loadHapticPattern(named: "Sculpt")
    hapticPlayer = try? hapticEngine?.makePlayer(with: pattern)
}
```

### Trigger Haptics

```swift
func handleSculptHaptics(pressed: Bool) {
    if pressed {
        // Start continuous haptic
        hapticPlayer?.start(atTime: 0)
    } else {
        // Stop haptic
        hapticPlayer?.stop(atTime: 0)
    }
}
```

## 🐛 Common Issues

### Device Not Tracking

**Check:**
1. Device is powered on
2. Bluetooth is enabled
3. Privacy permissions granted
4. Device firmware updated

### Tracking Loss

**Causes:**
- Occlusion (device hidden)
- Out of tracking range
- Poor lighting conditions

**Solutions:**
- Keep device visible
- Stay within tracking volume
- Ensure adequate lighting

### Input Lag

**Optimize:**
- Reduce scene complexity
- Profile GPU usage
- Check frame rate

## 🎮 Experiment Ideas

### 1. Custom Gestures

```swift
// Detect drawing patterns
func detectGesture(positions: [SIMD3<Float>]) -> GestureType? {
    // Analyze position history
    // Return detected gesture
}
```

### 2. Pressure Sensitivity

Some accessories support pressure:
```swift
if let pressure = stylus.input?.axes[.pressure]?.value {
    sculptingTool.components[SculptingToolComponent.self]?.radius *= pressure
}
```

### 3. Multi-Device Support

Track multiple accessories:
```swift
var accessories: [ObjectIdentifier: AnchorEntity] = [:]

func addAccessory(_ device: GCSpatialAccessory) {
    let id = ObjectIdentifier(device)
    accessories[id] = AnchorEntity(device.objectAnchor)
}
```

## 🔧 Advanced Techniques

### Prediction and Smoothing

```swift
// Predict future position for lower latency
func predictPosition(
    current: SIMD3<Float>,
    velocity: SIMD3<Float>,
    deltaTime: Float
) -> SIMD3<Float> {
    return current + velocity * deltaTime
}

// Smooth jittery input
func smoothPosition(
    new: SIMD3<Float>,
    previous: SIMD3<Float>,
    smoothing: Float = 0.1
) -> SIMD3<Float> {
    return mix(previous, new, smoothing)
}
```

### Custom Tracking Indicators

```swift
func createCustomIndicator() -> ModelEntity {
    var material = SimpleMaterial()
    material.color.tint = .systemBlue
    material.metallic = 0.8
    material.roughness = 0.2
    
    let mesh = MeshResource.generateSphere(radius: 0.02)
    return ModelEntity(mesh: mesh, materials: [material])
}
```

## 📚 Key Concepts

### Coordinate Systems

1. **Device Space**: Relative to the accessory
2. **World Space**: Global scene coordinates
3. **View Space**: Relative to the user's head

### Transform Hierarchy

```
Scene Root
 └── Accessory Anchor
      ├── Tracking Indicator
      └── Tool Visualization
```

## 🔗 Resources

- [ARKit Documentation](https://developer.apple.com/documentation/arkit)
- [GameController Framework](https://developer.apple.com/documentation/gamecontroller)
- [Spatial Tracking Session Guide](https://developer.apple.com/documentation/arkit/spatialtrackingsession)
- [Core Haptics](https://developer.apple.com/documentation/corehaptics)

## 💡 Best Practices

1. **Always check tracking state** before using position data
2. **Provide visual feedback** for tracking quality
3. **Handle disconnections gracefully**
4. **Test with multiple devices** if supporting various accessories
5. **Optimize for battery life** when using haptics

## 🎯 Next Steps

- Learn about [ECS Architecture](06-ECS-ARCHITECTURE.md)
- Explore [User Interaction](07-USER-INTERACTION.md)
- Read Apple's Spatial Input Guidelines

Remember: Good spatial input feels natural and responsive. The user should focus on their creation, not the tool!
