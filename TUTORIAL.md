# VisionOS Spatial Sculpting Tutorial: A Complete Beginner's Guide

## 🎯 What You'll Learn

This tutorial will walk you through Apple's Spatial Sculpting sample code, which demonstrates how to create a 3D sculpting app for visionOS that works with handheld accessories like the Logitech MX Ink stylus. By the end, you'll understand:

- How visionOS apps start and initialize
- 3D voxel representations and mesh generation
- Metal compute shaders for GPU acceleration
- Spatial accessory tracking
- RealityKit's Entity Component System (ECS)
- User interaction in 3D space

## 📱 App Overview

This app lets you:
- Sculpt 3D objects in virtual space using a tracked accessory
- Add or remove material like digital clay
- Adjust the sculpting tool size
- Save and load your creations
- Get haptic feedback while sculpting

## 🚀 The App Launch Journey

Let's follow the app from launch to your first sculpt!

### Step 1: App Entry Point (`SpatialSculptingApp.swift`)

```swift
@main
struct SpatialSculptingApp: App {
    init() {
        ComputeDispatchSystem.registerSystem()
    }
    
    var body: some Scene {
        WindowGroup {
            ContentView().frame(width: 1500, height: 1500).frame(depth: 1500)
        }
        .windowStyle(.volumetric)
    }
}
```

**What happens here:**

1. **`@main`** - Marks this as the app's starting point
2. **`init()`** - Registers the compute system (for GPU operations)
3. **`WindowGroup`** - Creates a volumetric window (3D space)
4. **`.frame()`** - Sets the 3D volume to 1.5m × 1.5m × 1.5m

### Step 2: Creating the 3D World (`ContentView.swift`)

The ContentView is where the magic begins. Let's break it down:

#### 2.1 Initialization

```swift
init() {
    let dimensions = SIMD3<UInt32>(128, 128, 128)
    let voxelSize = SIMD3<Float>(0.8, 0.8, 0.8) / SIMD3<Float>(dimensions)
    let voxelStartPosition = -SIMD3<Float>(dimensions) * voxelSize / 2
    
    // Create voxel volume (3D grid of points)
    guard let voxelVolume = try? VoxelVolume(...) else { return }
    
    // Create mesh generator
    self.marchingCubesMesh = try? MarchingCubesMesh(voxelVolume: voxelVolume)
    
    // Create sculptor (handles modifications)
    self.sculptor = MarchingCubesMeshSculptor(marchingCubesMesh: marchingCubesMesh)
}
```

**What's happening:**
- Creates a 128×128×128 grid of voxels (volumetric pixels)
- Each voxel represents a small cube in 3D space
- The grid is centered at the origin (0,0,0)
- Total size is 0.8m × 0.8m × 0.8m

#### 2.2 Building the Scene

```swift
RealityView { content, attachments in
    // Create mesh entities
    for meshChunk in meshChunks {
        let meshEntity = createMeshChunkEntity(meshChunk: meshChunk)
        root.addChild(meshEntity)
    }
    
    // Add sculpting tool
    root.addChild(sculpting.sculptingTool)
    
    // Add to scene
    content.add(root)
    
    // Update every frame
    _ = content.subscribe(to: SceneEvents.Update.self) { _ in
        sculpting.updateSculptingTool()
    }
}
```

### Step 3: Understanding the Core Systems

#### 🧊 **Voxel System**
Think of voxels as 3D pixels. Each voxel stores a value representing:
- Positive = Outside the surface
- Negative = Inside the surface
- Zero = On the surface

#### 🔺 **Marching Cubes Algorithm**
Converts voxel data into a 3D mesh:
1. Examines each cube of 8 voxels
2. Determines which edges the surface crosses
3. Creates triangles to represent the surface

#### ⚡ **Metal Compute**
Uses the GPU for fast calculations:
- `reset` shader - Creates initial box shape
- `clear` shader - Empties the volume
- `sculpt` shader - Adds/removes material
- `march` shader - Generates mesh triangles

### Step 4: Accessory Tracking

The app tracks spatial accessories (stylus/controller) using:

```swift
// In ContentView
.task {
    let configuration = SpatialTrackingSession.Configuration(tracking: [.accessory])
    let session = SpatialTrackingSession()
    await session.run(configuration)
}
```

Game Controller Framework handles:
- Button presses (primary/secondary)
- Position tracking
- Haptic feedback

### Step 5: User Interaction Flow

1. **Accessory Movement** → Updates tool position
2. **Button Press** → Activates sculpting
3. **Sculpting** → Modifies voxel values
4. **Mesh Update** → GPU regenerates triangles
5. **Display** → RealityKit renders the mesh

### Step 6: The Sculpting Process

When you press the trigger:

```swift
// In SculptingToolSystem
if sculptingToolComponent.isActive {
    let sculptParams = SculptParams(
        mode: mode,  // add or remove
        toolPositionAndRadius: SIMD4(position, radius),
        previousPositionAndHasPosition: previousPosition
    )
    sculptor.sculpt(sculptParams, computeContext)
}
```

The GPU shader:
1. Calculates distance from tool to each voxel
2. Modifies voxel values within radius
3. Smoothly blends changes
4. Creates capsule shape between positions

### Step 7: Toolbar and UI

The toolbar appears when you press the palette button:

- **Green sphere** - Add mode
- **Red sphere** - Subtract mode
- **"+"** - Increase tool size
- **"-"** - Decrease tool size

Bottom toolbar:
- **Save** - Export sculpture file
- **Open** - Load sculpture file
- **Clear** - Empty the volume
- **Reset** - Return to box shape

## 🎮 Try It Yourself!

1. **Run the app** on visionOS simulator or device
2. **Connect** a supported accessory
3. **Press and hold** the secondary button to sculpt
4. **Press** the primary button to show the toolbar
5. **Point at** toolbar items to select them
6. **Save** your creation when done!

## 🔧 Key Concepts for Beginners

### Entity Component System (ECS)
- **Entity**: A container (like the sculpting tool)
- **Component**: Data attached to entities (like position, appearance)
- **System**: Logic that processes components each frame

### Coordinate System
- X = Left/Right
- Y = Up/Down  
- Z = Forward/Backward
- Origin (0,0,0) = Center of the volume

### Performance Tips
- Uses GPU for heavy calculations
- Splits mesh into chunks for efficiency
- Only updates changed areas
- Runs at 90Hz for smooth interaction

## 📚 Next Steps

Explore the detailed guides:
1. [Getting Started](docs/01-GETTING-STARTED.md) - Setup and requirements
2. [Voxel System](docs/02-VOXEL-SYSTEM.md) - Deep dive into voxels
3. [Marching Cubes](docs/03-MARCHING-CUBES.md) - Mesh generation algorithm
4. [Metal Compute](docs/04-METAL-COMPUTE.md) - GPU programming
5. [Accessory Tracking](docs/05-ACCESSORY-TRACKING.md) - Spatial input
6. [ECS Architecture](docs/06-ECS-ARCHITECTURE.md) - RealityKit patterns
7. [User Interaction](docs/07-USER-INTERACTION.md) - UI and controls

Happy Sculpting! 🎨
