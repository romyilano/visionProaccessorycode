# Getting Started with VisionOS Spatial Sculpting

## 📋 Prerequisites

Before diving into the code, ensure you have:

### Hardware Requirements
- **Mac** with Apple Silicon (M1 or newer)
- **Apple Vision Pro** (optional - simulator works too!)
- **Supported Accessory** (optional):
  - Logitech MX Ink stylus
  - Compatible spatial game controller

### Software Requirements
- **macOS** Sonoma 14.0 or later
- **Xcode** 15.0 or later
- **visionOS SDK** 1.0 or later
- **Swift** 5.9 or later

## 🚀 First Run

### Step 1: Open the Project

1. **Clone or download** the sample code
2. **Double-click** `SpatialSculpting.xcodeproj`
3. Xcode will open the project

### Step 2: Select Your Target

In Xcode's toolbar:
- **Device**: Choose "Apple Vision Pro" or "Vision Pro Simulator"
- **Scheme**: Ensure "SpatialSculpting" is selected

### Step 3: Build and Run

1. Press **Cmd+R** or click the **Play** button
2. Wait for the build to complete
3. The app will launch in your chosen environment

## 🎮 Using the Simulator

If you don't have a physical device:

### Navigation Controls
- **Look around**: Click and drag
- **Move forward/back**: W/S keys
- **Move left/right**: A/D keys
- **Move up/down**: Q/E keys
- **Reset view**: Shift+Cmd+R

### Simulating an Accessory
Without a physical accessory, you can:
1. Use the **virtual hands** in simulator
2. Click to simulate button presses
3. Drag to move the tool position

## 🛠 Understanding the Project Structure

```
SpatialSculpting/
├── SpatialSculptingApp.swift    # App entry point
├── ContentView.swift             # Main UI and scene setup
├── Volume/                       # Voxel data structures
│   ├── VoxelVolume.swift        # 3D texture management
│   └── VolumeParams.h           # C header for Metal
├── Mesh/                         # Mesh generation
│   ├── MarchingCubesMesh.swift  # Algorithm implementation
│   └── MarchingCubesCompute.metal # GPU shader
├── Sculpting/                    # Modification logic
│   ├── MarchingCubesMeshSculptor.swift
│   └── SculptVoxelsComputeShader.metal
├── ECS/                          # Entity Component System
│   └── SculptingToolComponent.swift
├── ViewModel/                    # Business logic
│   ├── SculptingToolModel.swift
│   └── HapticsModel.swift
├── Compute/                      # GPU pipeline setup
│   └── ComputeSystem.swift
└── UI/                          # User interface elements
    └── ToolbarElement.swift
```

## 🔧 Key Technologies Used

### RealityKit
- 3D rendering and scene management
- Entity Component System (ECS)
- Spatial tracking integration

### Metal
- GPU compute shaders
- High-performance mesh generation
- Real-time voxel manipulation

### SwiftUI
- User interface elements
- Window management
- State handling

### ARKit
- Spatial accessory tracking
- Hand tracking (fallback)
- World understanding

### GameController Framework
- Accessory button input
- Haptic feedback
- Device management

## 🏃‍♂️ Your First Modification

Try this simple change to understand the code:

### Change the Initial Shape

1. Open `SpatialSculpting/Sculpting/SculptVoxelsComputeShader.metal`
2. Find the `reset` function
3. Look for this line:
   ```metal
   float boxDistance = boxSDF(position, boxHalfExtents);
   ```
4. Try changing to a sphere:
   ```metal
   float sphereRadius = 0.3;
   float boxDistance = length(position) - sphereRadius;
   ```
5. Build and run - you'll start with a sphere instead of a box!

## 🐛 Common Issues and Solutions

### Build Errors

**"No such module 'RealityKit'"**
- Ensure you're targeting visionOS, not iOS
- Check that visionOS SDK is installed

**"Cannot find type 'SpatialTrackingSession'"**
- Update to the latest Xcode
- Verify visionOS SDK version

### Runtime Issues

**App launches but no mesh appears**
- Check console for Metal errors
- Ensure compute pipeline initialized
- Try the "Reset" button

**Accessory not tracking**
- Verify accessory is paired
- Check Privacy settings
- Ensure tracking permissions granted

## 📚 Learning Path

### For Complete Beginners
1. Start with SwiftUI basics
2. Learn about 3D coordinates
3. Understand basic RealityKit concepts
4. Explore this sample gradually

### For iOS Developers
1. Focus on visionOS-specific APIs
2. Study spatial computing concepts
3. Learn Metal compute basics
4. Experiment with 3D interactions

### For 3D Graphics Developers
1. Dive into the marching cubes implementation
2. Explore Metal shader optimization
3. Study the voxel data structures
4. Extend the sculpting algorithms

## 🎯 Next Steps

Now that you have the app running:

1. **Experiment** with the existing features
2. **Read** [Voxel System Guide](02-VOXEL-SYSTEM.md)
3. **Try** simple code modifications
4. **Explore** the Metal shaders
5. **Build** your own features!

Remember: The best way to learn is by doing. Don't be afraid to break things - you can always git reset!

## 🔗 Additional Resources

- [Apple's visionOS Documentation](https://developer.apple.com/visionos/)
- [RealityKit Framework](https://developer.apple.com/documentation/realitykit)
- [Metal Programming Guide](https://developer.apple.com/metal/)
- [Spatial Computing Design Guidelines](https://developer.apple.com/design/human-interface-guidelines/spatial-computing)
