# Understanding the Voxel System

## 🧊 What Are Voxels?

**Voxel** = **Vol**umetric + **Pixel**

Just as pixels are 2D picture elements, voxels are 3D volume elements. Think of them as tiny cubes in 3D space that store data.

### In This App
- Each voxel stores a **distance value** (Float)
- The 3D grid contains 128×128×128 = 2,097,152 voxels
- Total volume spans 0.8m × 0.8m × 0.8m

## 📐 The Voxel Grid

### Coordinate System

```
        Y (up)
        |
        |
        +------ X (right)
       /
      /
     Z (forward)
```

### Grid Structure
```swift
// VoxelVolume.swift
let dimensions = SIMD3<UInt32>(128, 128, 128)
let voxelSize = SIMD3<Float>(0.8, 0.8, 0.8) / SIMD3<Float>(dimensions)
let voxelStartPosition = -SIMD3<Float>(dimensions) * voxelSize / 2
```

This creates:
- **Grid dimensions**: 128×128×128 voxels
- **Each voxel size**: ~6.25mm per side
- **Grid centered at**: (0, 0, 0)

## 💾 Voxel Data Storage

### The 3D Texture

Voxels are stored in a Metal 3D texture:

```swift
// VoxelVolume.swift
let textureDescriptor = MTLTextureDescriptor()
textureDescriptor.textureType = .type3D
textureDescriptor.pixelFormat = .r32Float  // 32-bit float per voxel
textureDescriptor.width = Int(dimensions.x)
textureDescriptor.height = Int(dimensions.y)
textureDescriptor.depth = Int(dimensions.z)
```

### What Each Voxel Stores

Each voxel contains a **signed distance value**:
- **Negative** (-): Inside the surface
- **Zero** (0): On the surface (isosurface)
- **Positive** (+): Outside the surface

```
Outside (+0.5)    Surface (0)    Inside (-0.5)
     [ ]             [X]            [■]
```

## 🎨 How Sculpting Works

### Distance Fields

The app uses **Signed Distance Fields (SDF)**:

1. **Box SDF** (initial shape):
   ```metal
   float boxSDF(float3 position, float3 halfExtents) {
       float3 d = abs(position) - halfExtents;
       return length(max(d, 0.0)) + min(max(d.x, max(d.y, d.z)), 0.0);
   }
   ```

2. **Sphere SDF** (sculpting tool):
   ```metal
   float sphereSDF(float3 position, float3 center, float radius) {
       return length(position - center) - radius;
   }
   ```

### Adding Material

When you sculpt in "add" mode:
```metal
// Simplified version
float toolDistance = sphereSDF(voxelPosition, toolPosition, toolRadius);
voxelValue = min(voxelValue, -toolDistance);  // Union operation
```

### Removing Material

When you sculpt in "remove" mode:
```metal
// Simplified version
float toolDistance = sphereSDF(voxelPosition, toolPosition, toolRadius);
voxelValue = max(voxelValue, toolDistance);  // Subtraction operation
```

## 🔄 The Update Cycle

1. **User Action** → Press sculpt button
2. **Tool Position** → Get accessory location
3. **GPU Compute** → Update affected voxels
4. **Mesh Generation** → Convert to triangles
5. **Display** → Render the mesh

## 📊 Performance Considerations

### Memory Usage
- Each voxel: 4 bytes (float32)
- Total voxels: 128³ = 2,097,152
- Memory: ~8.4 MB for voxel data

### Optimization Strategies

1. **Spatial Locality**
   - Only update voxels near the tool
   - Use bounding box calculations

2. **GPU Parallelism**
   - Each voxel updates independently
   - Thousands of threads process simultaneously

3. **Smooth Blending**
   ```metal
   float blend = smoothstep(0.0, blendDistance, distance);
   ```

## 🛠 Working with Voxels

### Accessing Voxel Data

Reading a voxel value:
```metal
float voxelValue = inTexture.read(voxelCoord).r;
```

Writing a voxel value:
```metal
outTexture.write(float4(newValue), voxelCoord);
```

### Converting World to Voxel Coordinates

```metal
int3 worldToVoxel(float3 worldPos, VolumeParams params) {
    float3 localPos = worldPos - params.voxelStartPosition;
    float3 voxelPosF = localPos / params.voxelSize;
    return int3(voxelPosF);
}
```

## 🎯 Practical Examples

### Example 1: Check if Point is Inside

```swift
func isPointInside(worldPosition: SIMD3<Float>) -> Bool {
    let voxelCoord = worldToVoxel(worldPosition)
    let voxelValue = getVoxelValue(at: voxelCoord)
    return voxelValue < 0  // Negative = inside
}
```

### Example 2: Find Surface Normal

```metal
float3 calculateNormal(int3 voxelCoord, texture3d<float> volume) {
    float3 gradient;
    gradient.x = volume.read(voxelCoord + int3(1,0,0)).r - 
                 volume.read(voxelCoord - int3(1,0,0)).r;
    gradient.y = volume.read(voxelCoord + int3(0,1,0)).r - 
                 volume.read(voxelCoord - int3(0,1,0)).r;
    gradient.z = volume.read(voxelCoord + int3(0,0,1)).r - 
                 volume.read(voxelCoord - int3(0,0,1)).r;
    return normalize(gradient);
}
```

## 🔬 Advanced Concepts

### Trilinear Interpolation

For smooth values between voxels:
```metal
float trilinearSample(float3 position, texture3d<float> volume) {
    // Get the 8 surrounding voxels
    // Interpolate based on fractional position
    // Return smooth value
}
```

### Level of Detail (LOD)

Future optimization could include:
- Multiple resolution grids
- Octree structures
- Adaptive sampling

## 📚 Key Takeaways

1. **Voxels store distance values**, not just on/off states
2. **Negative = inside**, positive = outside the surface
3. **SDFs enable smooth** boolean operations
4. **GPU processes voxels** in parallel for speed
5. **Memory layout matters** for performance

## 🎮 Experiment Ideas

1. **Change Grid Resolution**
   - Try 64×64×64 for faster updates
   - Try 256×256×256 for more detail

2. **Modify Distance Functions**
   - Create cube-shaped tools
   - Add noise for organic shapes

3. **Implement New Operations**
   - Smooth/blur existing sculptures
   - Inflate/deflate surfaces

## 🔗 Related Topics

- [Marching Cubes Algorithm](03-MARCHING-CUBES.md) - How voxels become meshes
- [Metal Compute Shaders](04-METAL-COMPUTE.md) - GPU programming details
- [API Reference](API-REFERENCE.md) - VoxelVolume class details
