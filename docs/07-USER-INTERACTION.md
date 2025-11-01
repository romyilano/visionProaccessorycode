# User Interaction and UI Guide

## 🎨 UI Overview

The app provides two main interaction methods:
1. **Spatial Controls** - Via the tracked accessory
2. **Visual UI** - SwiftUI elements in 3D space

## 🎯 Interaction Flow

### User Actions Hierarchy

```
Physical Input
    ↓
GameController Framework
    ↓
Button Handlers
    ↓
Component Updates
    ↓
GPU Processing
    ↓
Visual Feedback
```

## 🎮 Button Controls

### Stylus Controls

| Button | Action | Function |
|--------|--------|----------|
| Secondary | Hold | Sculpt (add/remove material) |
| Primary | Press | Show/hide toolbar |
| Primary | Release | Select toolbar item |

### Controller Controls

| Button | Action | Function |
|--------|--------|----------|
| Trigger | Hold | Sculpt (add/remove material) |
| Thumbstick | Press | Show/hide toolbar |
| Thumbstick | Release | Select toolbar item |

## 🖼 The Toolbar System

### Toolbar Elements

```swift
// ToolbarElement.swift
struct ToolbarElement: View {
    let name: String
    
    var body: some View {
        VStack {
            Image(systemName: iconForName(name))
                .font(.system(size: 40))
                .frame(width: 80, height: 80)
                .background(.regularMaterial)
                .clipShape(Circle())
            
            Text(name)
                .font(.caption)
        }
        .frame(width: 100, height: 120)
    }
}
```

### Toolbar Icons

- **Add** (green) - Switch to additive mode
- **Subtract** (red) - Switch to subtractive mode  
- **Enlarge** (+) - Increase tool radius
- **Reduce** (-) - Decrease tool radius

### Dynamic Positioning

The toolbar appears relative to hand position:

```swift
func displayToolbar(transform: Transform, accessoryAnchor: AccessoryAnchor) {
    let xTranslation: Float = {
        switch accessoryAnchor.heldChirality {
        case .left:
            return 0.05  // Right of left hand
        case .right:
            return -0.05 // Left of right hand
        default:
            return 0.0
        }
    }()
    
    // Position toolbar elements
    additiveIcon.position = transform.translation + SIMD3(xTranslation, 0.05, -0.1)
    subtractiveIcon.position = transform.translation + SIMD3(xTranslation, 0, -0.1)
    // etc...
}
```

## 🎯 Raycast Selection

### How Selection Works

1. **User points** accessory at UI element
2. **Ray is cast** from accessory position
3. **Collision detected** with UI elements
4. **Action triggered** based on element name

```swift
func selectToolbarElement(sculptingEntity: AnchorEntity) {
    // Get ray origin and direction
    let raycastOrigin = sculptingEntity.position(relativeTo: nil)
    let raycastForward = -simd_make_float3(
        sculptingEntity.transformMatrix(relativeTo: nil).columns.2
    )
    
    // Perform raycast
    if let hit = scene.raycast(
        origin: raycastOrigin,
        direction: raycastForward,
        length: 10
    ).first {
        // Handle selection
        handleToolbarSelection(hit.entity)
    }
}
```

## 📁 File Operations UI

### Save Button

```swift
Button {
    sculpting.save { document in
        Task { @MainActor in
            self.saveDocument = document
            self.isSaving = true
        }
    }
} label: {
    Text("Save")
}
.fileExporter(
    isPresented: $isSaving,
    document: saveDocument
) { result in
    // Handle save result
}
```

### Open Button

```swift
Button {
    isOpening = true
} label: {
    Text("Open")
}
.fileImporter(
    isPresented: $isOpening,
    allowedContentTypes: [VolumeDocument.utType]
) { result in
    // Handle open result
}
```

## 🎨 Visual Feedback

### Tool Visualization

The sculpting tool shows:
- **Color** - Current mode (green=add, red=remove)
- **Size** - Current radius
- **Opacity** - Semi-transparent for visibility

```swift
// Update tool appearance
sculptingToolComponent.tooltip?.components[ModelComponent.self]?.materials = [
    SimpleMaterial(color: sculptingColor[mode]!, isMetallic: false)
]
sculptingToolComponent.tooltip?.scale = SIMD3(repeating: radius)
```

### Tracking State Indicator

Shows tracking quality with colors:
- 🟢 **Green** - Full 6DOF tracking
- 🟠 **Orange** - Position only (3DOF)
- 🔴 **Red** - No tracking

## 🖱 Interaction States

### State Management

```swift
@MainActor @Observable
final class SculptingToolModel {
    // UI State
    var additiveIcon: Entity? = nil
    var subtractiveIcon: Entity? = nil
    var enlargeIcon: Entity? = nil
    var reduceIcon: Entity? = nil
    
    // Tool State
    var mode: SculptingMode = .remove
    var radius: Float = 0.035
    var isActive: Bool = false
}
```

### State Flow

```
Button Press → Update Component → System Processes → Visual Update
```

## 🎯 Gesture Recognition

### Palette Press Gesture

```swift
func handlePalettePress(pressed: Bool) {
    if pressed {
        // Show toolbar
        displayToolbar(transform: sculptingTool.transform, 
                      accessoryAnchor: accessoryAnchor)
    } else {
        // Hide toolbar and process selection
        selectToolbarElement(sculptingEntity: sculptingEntity)
        hideToolbar()
    }
}
```

## 🛠 UI Customization

### Custom Toolbar Items

Add new toolbar elements:

```swift
// Add new mode
enum SculptingMode {
    case add
    case remove
    case smooth  // New mode
}

// Create icon
let smoothIcon = ToolbarElement(name: "Smooth")

// Handle selection
case "SmoothIcon":
    sculptingTool.components[SculptingToolComponent.self]?.mode = .smooth
```

### Custom Controls

Implement gesture controls:

```swift
// Double-tap to toggle mode
var lastTapTime: TimeInterval = 0

func handleTap() {
    let currentTime = Date().timeIntervalSince1970
    if currentTime - lastTapTime < 0.3 {
        // Double tap detected
        toggleMode()
    }
    lastTapTime = currentTime
}
```

## 📱 SwiftUI Integration

### RealityView with Attachments

```swift
RealityView { content, attachments in
    // Set up 3D content
    content.add(root)
    
    // Add UI attachments
    if let attachment = attachments.entity(for: "Additive") {
        root.addChild(attachment)
    }
} attachments: {
    // Define SwiftUI views as attachments
    Attachment(id: "Additive") {
        ToolbarElement(name: "Add")
    }
}
```

### Ornaments

```swift
.ornament(attachmentAnchor: .scene(.bottomFront)) {
    HStack {
        saveButton()
        openButton()
        clearButton()
        resetButton()
    }
    .padding()
    .glassBackgroundEffect()
}
```

## 🎮 Best Practices

### 1. Responsive Feedback

Always provide immediate feedback:
```swift
// Visual feedback
button.scaleEffect(pressed ? 0.9 : 1.0)

// Haptic feedback
haptics.playPattern(.click)

// Audio feedback
audioPlayer.play("buttonPress.m4a")
```

### 2. Clear Visual Hierarchy

- Primary actions: Larger, centered
- Secondary actions: Smaller, to the side
- Dangerous actions: Require confirmation

### 3. Accessibility

```swift
Button("Save") {
    // Action
}
.accessibilityLabel("Save sculpture")
.accessibilityHint("Exports the current sculpture to a file")
```

## 🐛 Common UI Issues

### Toolbar Not Appearing

**Check:**
- Button handlers connected
- Entities enabled
- Collision components set

### Selection Not Working

**Debug:**
```swift
// Visualize raycast
let debugLine = createDebugLine(from: origin, to: origin + direction * 10)
scene.addChild(debugLine)
```

### UI Elements Overlapping

**Solution:**
```swift
// Add spacing
let spacing: Float = 0.15
icon1.position.y = baseY + spacing * 0
icon2.position.y = baseY + spacing * 1
```

## 💡 UI Enhancement Ideas

### 1. Radial Menu

```swift
func createRadialMenu(around center: SIMD3<Float>) {
    let radius: Float = 0.2
    let angleStep = 2 * .pi / Float(items.count)
    
    for (index, item) in items.enumerated() {
        let angle = Float(index) * angleStep
        item.position = center + SIMD3(
            cos(angle) * radius,
            0,
            sin(angle) * radius
        )
    }
}
```

### 2. Tool Preview

Show preview of operation:
```swift
// Ghost object showing result
let preview = sculptingTool.clone()
preview.opacity = 0.5
preview.scale *= 1.2
```

### 3. Undo/Redo System

```swift
struct Action {
    let texture: MTLTexture
    let timestamp: Date
}

var undoStack: [Action] = []
var redoStack: [Action] = []
```

## 📚 Resources

- [SwiftUI in visionOS](https://developer.apple.com/documentation/visionos/bringing-your-swiftui-app-to-visionos)
- [Spatial UI Guidelines](https://developer.apple.com/design/human-interface-guidelines/spatial-computing)
- [RealityKit Gestures](https://developer.apple.com/documentation/realitykit/adding-gestures-to-realitykit-entities)

## 🎯 Key Takeaways

1. **Keep interactions simple** - Don't overwhelm users
2. **Provide clear feedback** - Visual, haptic, and audio
3. **Design for 3D space** - Think spatially
4. **Test with real devices** - Simulator isn't enough
5. **Respect user comfort** - Avoid arm fatigue

Great UI in spatial computing feels magical - the interface should enhance, not distract from, the creative experience!
