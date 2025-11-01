# Metal Compute Shaders: GPU Programming Guide

## ⚡ What is Metal Compute?

Metal is Apple's GPU programming framework. Compute shaders are programs that run on the GPU in parallel, processing thousands of data points simultaneously.

### Why GPU for Voxels?

- **CPU**: Processes voxels one at a time
- **GPU**: Processes thousands of voxels simultaneously
- **Result**: 100-1000x speedup for voxel operations

## 🏗 The Compute Pipeline

### 1. Setup (Swift Side)

```swift
// ComputeUtilities.swift
func makeComputePipeline(named name: String) -> MTLComputePipelineState? {
    let library = metalDevice?.makeDefaultLibrary()
    let function = library?.makeFunction(name: name)
    let pipeline = try? metalDevice?.makeComputePipelineState(function: function!)
    return pipeline
}
```

### 2. Dispatch (Every Frame)

```swift
// ComputeSystem.swift
func update(context: SceneUpdateContext) {
    // Create command buffer
    let commandBuffer = commandQueue.makeCommandBuffer()
    
    // Create compute encoder
    let computeEncoder = commandBuffer.makeComputeCommandEncoder()
    
    // Set pipeline and parameters
    computeEncoder.setComputePipelineState(pipeline)
    
    // Dispatch threads
    computeEncoder.dispatchThreadgroups(threadgroups, 
                                       threadsPerThreadgroup: threadsPerGroup)
    
    // Commit to GPU
    commandBuffer.commit()
}
```

## 🎯 The Shaders

### Reset Shader - Creating the Initial Box

```metal
kernel void reset(
    texture3d<float, access::write> outTexture [[texture(0)]],
    constant VolumeParams& volumeParams [[buffer(1)]],
    uint3 id [[thread_position_in_grid]])
{
    // Check bounds
    if (any(id >= uint3(volumeParams.dimensions))) {
        return;
    }
    
    // Convert voxel coordinate to world position
    float3 voxelPosF = float3(id);
    float3 position = volumeToWorld(voxelPosF, volumeParams);
    
    // Create a box shape
    float3 boxHalfExtents = float3(0.3, 0.3, 0.3);
    float distance = boxSDF(position, boxHalfExtents);
    
    // Write to 3D texture
    outTexture.write(float4(distance), id);
}
```

### Sculpt Shader - Modifying the Volume

```metal
kernel void sculpt(
    texture3d<float, access::read> inTexture [[texture(0)]],
    texture3d<float, access::write> outTexture [[texture(1)]],
    constant VolumeParams& volumeParams [[buffer(2)]],
    constant SculptParams& sculptParams [[buffer(3)]],
    uint3 id [[thread_position_in_grid]])
{
    // Read current value
    float currentValue = inTexture.read(id).r;
    
    // Calculate tool influence
    float3 position = volumeToWorld(float3(id), volumeParams);
    float toolInfluence = calculateToolInfluence(position, sculptParams);
    
    // Apply sculpting operation
    float newValue;
    if (sculptParams.mode == add) {
        newValue = smoothMin(currentValue, toolInfluence);
    } else {
        newValue = smoothMax(currentValue, -toolInfluence);
    }
    
    // Write new value
    outTexture.write(float4(newValue), id);
}
```

## 🔧 Key Concepts

### Thread Organization

```
Grid (entire volume)
 └── Threadgroups (8×8×8 blocks)
      └── Threads (individual voxels)
```

```swift
// Ideal thread configuration
let threadsPerThreadgroup = MTLSize(width: 8, height: 8, depth: 8)
let threadgroupCount = MTLSize(
    width: (dimensions.x + 7) / 8,
    height: (dimensions.y + 7) / 8,
    depth: (dimensions.z + 7) / 8
)
```

### Memory Types

1. **Texture Memory** - 3D voxel data
   ```metal
   texture3d<float, access::read_write> voxels
   ```

2. **Buffer Memory** - Parameters
   ```metal
   constant VolumeParams& params [[buffer(0)]]
   ```

3. **Shared Memory** - Within threadgroup
   ```metal
   threadgroup float sharedData[512];
   ```

## 📊 Performance Optimization

### 1. Coalesced Memory Access

Good pattern:
```metal
// Adjacent threads access adjacent memory
float value = texture.read(id).r;
```

Bad pattern:
```metal
// Random access pattern
float value = texture.read(randomCoord()).r;
```

### 2. Minimize Texture Reads

```metal
// Bad: Multiple reads
float v1 = texture.read(coord).r;
float v2 = texture.read(coord).r;

// Good: Read once, reuse
float value = texture.read(coord).r;
// Use 'value' multiple times
```

### 3. Use Fast Math

```metal
// Enable fast math functions
float distance = fast::length(position - center);
float result = fast::normalize(gradient);
```

## 🛠 Shader Utilities

### Signed Distance Functions (SDFs)

```metal
// Sphere SDF
float sphereSDF(float3 p, float3 center, float radius) {
    return length(p - center) - radius;
}

// Box SDF
float boxSDF(float3 p, float3 halfExtents) {
    float3 d = abs(p) - halfExtents;
    return length(max(d, 0.0)) + min(max(d.x, max(d.y, d.z)), 0.0);
}

// Capsule SDF (for smooth tool strokes)
float capsuleSDF(float3 p, float3 a, float3 b, float radius) {
    float3 pa = p - a;
    float3 ba = b - a;
    float h = saturate(dot(pa, ba) / dot(ba, ba));
    return length(pa - ba * h) - radius;
}
```

### Smooth Operations

```metal
// Smooth minimum (for unions)
float smoothMin(float a, float b, float k = 0.1) {
    float h = saturate(0.5 + 0.5 * (b - a) / k);
    return mix(b, a, h) - k * h * (1.0 - h);
}

// Smooth maximum (for subtractions)
float smoothMax(float a, float b, float k = 0.1) {
    return -smoothMin(-a, -b, k);
}
```

## 🐛 Debugging Metal Shaders

### 1. Use Color for Debug Output

```metal
// Visualize thread IDs
float3 debugColor = float3(id) / float3(volumeParams.dimensions);
outTexture.write(float4(debugColor, 1.0), id);
```

### 2. GPU Frame Capture

In Xcode:
1. Run the app
2. Click "M" button in debug bar
3. Capture GPU frame
4. Inspect textures and buffers

### 3. Common Issues

**Black Screen**
- Check texture write permissions
- Verify thread bounds checking
- Ensure pipeline state is set

**Incorrect Results**
- Print intermediate values to texture
- Check coordinate transformations
- Verify parameter passing

## 🎮 Experiments

### 1. Custom Shapes

Add a torus tool:
```metal
float torusSDF(float3 p, float2 radii) {
    float2 q = float2(length(p.xz) - radii.x, p.y);
    return length(q) - radii.y;
}
```

### 2. Texture Effects

Add noise to sculptures:
```metal
float noise = simplexNoise(position * 10.0) * 0.01;
distance += noise;
```

### 3. Performance Testing

```swift
// Measure GPU time
commandBuffer.addCompletedHandler { buffer in
    let gpuTime = buffer.gpuEndTime - buffer.gpuStartTime
    print("GPU time: \(gpuTime * 1000)ms")
}
```

## 📈 Advanced Topics

### Compute Shader Variants

```metal
// Function constants for compile-time optimization
constant bool useSmoothing [[function_constant(0)]];

kernel void sculptOptimized(...) {
    if (useSmoothing) {
        // Smooth blending path
    } else {
        // Hard edge path
    }
}
```

### Indirect Dispatch

For dynamic workloads:
```swift
// Dispatch based on runtime data
encoder.dispatchThreadgroupsWithIndirectBuffer(
    indirectBuffer,
    indirectBufferOffset: 0,
    threadsPerThreadgroup: threadsPerGroup
)
```

## 💡 Best Practices

1. **Always check bounds** in kernels
2. **Minimize memory bandwidth** usage
3. **Use appropriate data types** (half vs float)
4. **Profile before optimizing**
5. **Test on actual hardware** (not just simulator)

## 🔗 Resources

- [Metal Shading Language Spec](https://developer.apple.com/metal/Metal-Shading-Language-Specification.pdf)
- [Metal Performance Shaders](https://developer.apple.com/documentation/metalperformanceshaders)
- [GPU Frame Debugging](https://developer.apple.com/documentation/xcode/debugging-gpu-side-errors)
- [Metal Sample Code](https://developer.apple.com/metal/sample-code/)

## 📚 Next Steps

- Explore [Accessory Tracking](05-ACCESSORY-TRACKING.md)
- Study [ECS Architecture](06-ECS-ARCHITECTURE.md)
- Read Apple's Metal Programming Guide

Remember: GPU programming is about thinking in parallel. Each thread should do simple work, but thousands run simultaneously!
