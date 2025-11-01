# API Reference

## 📦 Core Classes and Structures

### VoxelVolume

Manages the 3D texture representing the voxel data.

```swift
@MainActor
final class VoxelVolume {
    // Properties
    var voxelTexture: MTLTexture
    let dimensions: SIMD3<UInt32>
    let voxelSize: SIMD3<Float>
    let voxelStartPosition: SIMD3<Float>
    var volumeParams: VolumeParams
    
    // Initialization
    init(dimensions: SIMD3<UInt32>, 
         voxelSize: SIMD3<Float>, 
         voxelStartPosition: SIMD3<Float>) throws
}
```

**Key Properties:**
- `dimensions` - Number of voxels in each dimension (x, y, z)
- `voxelSize` - World space size of each voxel
- `voxelStartPosition` - World space position of first voxel
- `voxelTexture` - 3D Metal texture storing distance values

### MarchingCubesMesh

Generates triangle meshes from voxel data using marching cubes.

```swift
@MainActor
final class MarchingCubesMesh {
    // Properties
    let voxelVolume: VoxelVolume
    var meshChunks: [MarchingCubesMeshChunk] = []
    var isoValue: Float = 0
    
    // Initialization
    init(voxelVolume: VoxelVolume) throws
    
    // Methods
    func update(computeContext: inout ComputeUpdateContext)
}
```

**Key Methods:**
- `update()` - Regenerates mesh from current voxel data
- `createMeshChunk()` - Creates subdivision of mesh for performance

### MarchingCubesMeshSculptor

Handles modifications to the voxel volume.

```swift
@MainActor
struct MarchingCubesMeshSculptor {
    // Properties
    let marchingCubesMesh: MarchingCubesMesh
    
    // Methods
    func reset(computeContext: inout ComputeUpdateContext)
    func clear(computeContext: inout ComputeUpdateContext)
    func sculpt(sculptParams: SculptParams, 
                computeContext: inout ComputeUpdateContext)
    func save(destinationTexture: MTLTexture,
              computeContext: inout ComputeUpdateContext,
              onCompletion: @Sendable @escaping () throws -> Void)
    func load(sourceTexture: MTLTexture,
              computeContext: inout ComputeUpdateContext)
}
```

**Key Methods:**
- `reset()` - Resets volume to initial box shape
- `clear()` - Empties the volume completely
- `sculpt()` - Applies sculpting operation at position
- `save()` - Exports voxel data to texture
- `load()` - Imports voxel data from texture

### SculptingToolModel

Manages the state and behavior of the sculpting tool.

```swift
@MainActor @Observable
final class SculptingToolModel {
    // Constants
    let minRadius: Float = 0.01
    let maxRadius: Float = 0.5
    
    // Entities
    var rootEntity: Entity? = nil
    let sculptingTool = Entity()
    var sculptingEntity: AnchorEntity? = nil
    var trackingStateIndicator: ModelEntity? = nil
    
    // UI Elements
    var additiveIcon: Entity? = nil
    var subtractiveIcon: Entity? = nil
    var enlargeIcon: Entity? = nil
    var reduceIcon: Entity? = nil
    
    // Methods
    func updateSculptingTool()
    func selectToolbarElement(sculptingEntity: AnchorEntity)
    func displayToolbar(transform: Transform, 
                       accessoryAnchor: AccessoryAnchor)
    func handlePalettePress(pressed: Bool)
    func handleGameControllerSetup(hapticsModel: HapticsModel) async
}
```

## 🎨 Components

### SculptingToolComponent

Component attached to sculpting tool entities.

```swift
struct SculptingToolComponent: Component {
    // Properties
    let sculptor: MarchingCubesMeshSculptor
    var mode: SculptingMode = .remove
    var radius: Float = 0.035
    var isActive: Bool = false
    var reset: Bool = true
    var clear: Bool = false
    var heldChirality: Accessory.Chirality? = nil
    var trackingState: AccessoryAnchor.TrackingState = .untracked
    var tooltip: ModelEntity? = nil
    var previousPosition: SIMD3<Float>? = nil
    var saveToTexture: (MTLTexture, @Sendable () throws -> Void)? = nil
    var loadFromTexture: MTLTexture? = nil
}
```

**Key Properties:**
- `mode` - Current sculpting mode (add/remove)
- `radius` - Size of the sculpting tool
- `isActive` - Whether currently sculpting
- `trackingState` - Quality of spatial tracking

### ComputeSystemComponent

Component for entities with compute systems.

```swift
struct ComputeSystemComponent: Component {
    let computeSystem: ComputeSystem
}
```

## 🔧 Systems

### SculptingToolSystem

Processes sculpting tool entities each frame.

```swift
struct SculptingToolSystem: ComputeSystem {
    // Query
    let query = EntityQuery(where: .has(SculptingToolComponent.self))
    
    // Update method
    func update(computeContext: inout ComputeUpdateContext)
}
```

**Update Flow:**
1. Query for entities with `SculptingToolComponent`
2. Update tool visualization
3. Handle reset/clear operations
4. Process sculpting if active
5. Handle save/load operations

### ComputeDispatchSystem

Manages GPU compute operations.

```swift
class ComputeDispatchSystem: System {
    // Properties
    static let commandQueue: MTLCommandQueue?
    let query = EntityQuery(where: .has(ComputeSystemComponent.self))
    
    // Methods
    required init(scene: Scene)
    func update(context: SceneUpdateContext)
}
```

## 📊 Data Structures

### VolumeParams

Parameters for voxel volume operations.

```metal
struct VolumeParams {
    simd_uint3 dimensions;
    simd_float3 voxelSize;
    simd_float3 voxelStartPosition;
};
```

### SculptParams

Parameters for sculpting operations.

```metal
typedef enum {
    add = 0,
    remove = 1,
} sculpt_mode;

struct SculptParams {
    sculpt_mode mode;
    simd_float4 toolPositionAndRadius;
    simd_float4 previousPositionAndHasPosition;
};
```

### MarchingCubesParams

Parameters for marching cubes mesh generation.

```metal  
struct MarchingCubesParams {
    simd_uint3 dimensions;
    simd_float3 voxelSize;
    simd_float3 voxelStartPosition;
    simd_uint3 chunkDimensions;
    uint32_t chunkStartZ;
    uint32_t maxVertexCount;
};
```

### MeshVertex

Vertex data structure for generated meshes.

```metal
struct MeshVertex {
    simd_float3 position;
    simd_float3 normal;
};
```

## 🎮 Enumerations

### SculptingMode

```swift
enum SculptingMode {
    case add      // Add material
    case remove   // Remove material
}
```

### TrackingState

```swift
enum TrackingState {
    case untracked                    // No tracking
    case positionTracked              // 3DOF tracking
    case positionOrientationTracked   // 6DOF tracking
}
```

## 🛠 Utility Functions

### Metal Pipeline Creation

```swift
func makeComputePipeline(named name: String) -> MTLComputePipelineState? {
    let library = metalDevice?.makeDefaultLibrary()
    let function = library?.makeFunction(name: name)
    return try? metalDevice?.makeComputePipelineState(function: function!)
}
```

### Command Queue Creation

```swift
func makeCommandQueue(labeled label: String) -> MTLCommandQueue? {
    return metalDevice?.makeCommandQueue()
}
```

### Coordinate Conversion

```metal
// Convert voxel coordinates to world position
float3 volumeToWorld(float3 voxelPos, VolumeParams params) {
    return params.voxelStartPosition + voxelPos * params.voxelSize;
}

// Convert world position to voxel coordinates  
int3 worldToVolume(float3 worldPos, VolumeParams params) {
    float3 localPos = worldPos - params.voxelStartPosition;
    return int3(localPos / params.voxelSize);
}
```

## 📱 SwiftUI Views

### ContentView

Main view containing the 3D scene.

```swift
struct ContentView: View {
    // State
    @State var sculpting: SculptingToolModel = SculptingToolModel()
    @State var haptics: HapticsModel = HapticsModel()
    
    // Mesh management
    let marchingCubesMesh: MarchingCubesMesh!
    let sculptor: MarchingCubesMeshSculptor!
    
    // UI state
    @State var saveDocument: VolumeDocument? = nil
    @State var isOpening = false
    @State var isSaving = false
    
    // View body
    var body: some View {
        ZStack {
            sculptingVolume()
                .ornament(attachmentAnchor: .scene(.bottomFront)) {
                    // Bottom toolbar
                }
        }
    }
}
```

### ToolbarElement

UI element for toolbar items.

```swift
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

## 🔌 Extensions

### SculptingToolModel Extensions

- `+GameController.swift` - Game controller input handling
- `+Anchoring.swift` - Spatial accessory anchoring
- `+VolumeUtilities.swift` - Volume save/load operations

### HapticsModel Extensions

- `+Haptics.swift` - Haptic feedback implementation

## 📝 Document Support

### VolumeDocument

Document type for saving/loading sculptures.

```swift
struct VolumeDocument: FileDocument {
    static var readableContentTypes: [UTType] = [.data]
    
    var data: Data
    
    init(data: Data) {
        self.data = data
    }
    
    init(configuration: ReadConfiguration) throws {
        self.data = configuration.file.regularFileContents ?? Data()
    }
    
    func fileWrapper(configuration: WriteConfiguration) throws -> FileWrapper {
        return FileWrapper(regularFileWithContents: data)
    }
}
```

## 🎯 Key Constants

### Sizes and Limits

```swift
// Volume dimensions
let dimensions = SIMD3<UInt32>(128, 128, 128)

// Tool radius limits
let minRadius: Float = 0.01
let maxRadius: Float = 0.5

// Default tool radius
let defaultRadius: Float = 0.035

// Max vertices per mesh chunk
let maxVertexCapacityPerMeshChunk = 8_500_000
```

### Colors

```swift
let sculptingColor: [SculptingMode: UIColor] = [
    .add: .green,
    .remove: .red
]

let trackingStateColor: [TrackingState: UIColor] = [
    .untracked: .red,
    .positionTracked: .orange,
    .positionOrientationTracked: .green
]
```

## 💡 Usage Examples

### Creating a Voxel Volume

```swift
let dimensions = SIMD3<UInt32>(128, 128, 128)
let voxelSize = SIMD3<Float>(0.8, 0.8, 0.8) / SIMD3<Float>(dimensions)
let startPosition = -SIMD3<Float>(dimensions) * voxelSize / 2

let volume = try VoxelVolume(
    dimensions: dimensions,
    voxelSize: voxelSize,
    voxelStartPosition: startPosition
)
```

### Performing a Sculpt Operation

```swift
let sculptParams = SculptParams(
    mode: .add,
    toolPositionAndRadius: SIMD4(x, y, z, radius),
    previousPositionAndHasPosition: SIMD4(px, py, pz, 1.0)
)

sculptor.sculpt(sculptParams: sculptParams, 
               computeContext: &computeContext)
```

### Querying Entities

```swift
let query = EntityQuery(where: .has(SculptingToolComponent.self))
for entity in scene.performQuery(query) {
    // Process sculpting tools
}
```

This API reference covers the main classes, structures, and functions used in the Spatial Sculpting app. For more detailed implementation examples, refer to the individual topic guides.
