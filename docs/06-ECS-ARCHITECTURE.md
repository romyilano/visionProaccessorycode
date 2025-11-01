# Entity Component System (ECS) in RealityKit

## 🎯 What is ECS?

Entity Component System is a design pattern that favors **composition over inheritance**:

- **Entity**: A unique identifier (like a container)
- **Component**: Data attached to entities (position, mesh, etc.)
- **System**: Logic that processes components

Think of it like LEGO:
- Entity = The base plate
- Components = Different LEGO pieces
- Systems = Instructions for what to do with the pieces

## 🏗 ECS in This App

### The Entities

```swift
// Main entities in the app
let root = Entity()                    // Scene root
let sculptingTool = Entity()           // The tool entity
let meshChunkEntity = Entity()         // Mesh display entities
let sculptingEntity = AnchorEntity()   // Accessory anchor
```

### The Components

#### Built-in RealityKit Components

```swift
// Visual representation
ModelComponent(mesh: mesh, materials: materials)

// Transparency
OpacityComponent(opacity: 0.01)

// Physics/collision
CollisionComponent(shapes: [.generateBox(size: size)])

// Transform in 3D space
Transform(translation: position, rotation: rotation, scale: scale)
```

#### Custom Component

```swift
// SculptingToolComponent.swift
struct SculptingToolComponent: Component {
    let sculptor: MarchingCubesMeshSculptor
    var mode: SculptingMode = .remove
    var radius: Float = 0.035
    var isActive: Bool = false
    // ... more properties
}
```

### The Systems

#### ComputeSystem Protocol

```swift
protocol ComputeSystem {
    @MainActor
    func update(computeContext: inout ComputeUpdateContext)
}
```

#### SculptingToolSystem

```swift
struct SculptingToolSystem: ComputeSystem {
    // Query for entities with SculptingToolComponent
    let query = EntityQuery(where: .has(SculptingToolComponent.self))
    
    func update(computeContext: inout ComputeUpdateContext) {
        // Process all sculpting tools
        for sculptingTool in computeContext.sceneUpdateContext.scene.performQuery(query) {
            // Update logic here
        }
    }
}
```

## 📦 Component Management

### Adding Components

```swift
// Add a component
entity.components.set(SculptingToolComponent(sculptor: sculptor))

// Or with the subscript syntax
entity.components[ModelComponent.self] = ModelComponent(mesh: mesh)
```

### Accessing Components

```swift
// Get a component (returns optional)
if let component = entity.components[SculptingToolComponent.self] {
    // Use component
}

// Check if entity has component
if entity.components.has(ModelComponent.self) {
    // Component exists
}
```

### Modifying Components

```swift
// Get mutable reference
guard var component = entity.components[SculptingToolComponent.self] else { return }

// Modify
component.radius = 0.05
component.mode = .add

// Set back (important!)
entity.components.set(component)
```

## 🔄 The Update Loop

### System Registration

```swift
// SpatialSculptingApp.swift
init() {
    ComputeDispatchSystem.registerSystem()
}
```

### Frame Updates

```
1. RealityKit calls ComputeDispatchSystem.update()
2. ComputeDispatchSystem queries for ComputeSystemComponents
3. Each ComputeSystem.update() is called
4. Systems process their entities
5. GPU commands are dispatched
```

### Update Flow in Detail

```swift
// ComputeSystem.swift
class ComputeDispatchSystem: System {
    func update(context: SceneUpdateContext) {
        // Get entities with compute systems
        let entities = context.entities(matching: query)
        
        // Create compute context
        var computeContext = ComputeUpdateContext(...)
        
        // Update each system
        for entity in entities {
            let component = entity.components[ComputeSystemComponent.self]
            component?.computeSystem.update(computeContext: &computeContext)
        }
        
        // Commit GPU work
        commandBuffer.commit()
    }
}
```

## 🎨 Entity Hierarchy

### Scene Structure

```
root (Entity)
├── meshChunkEntity (Entity)
│   └── ModelComponent
├── sculptingTool (Entity)
│   ├── ModelComponent
│   ├── OpacityComponent
│   └── SculptingToolComponent
└── sculptingEntity (AnchorEntity)
    └── trackingIndicator (ModelEntity)
```

### Parent-Child Relationships

```swift
// Add child
parent.addChild(child)

// Remove from parent
child.removeFromParent()

// Access parent
let parent = child.parent

// Access children
for child in parent.children {
    // Process child
}
```

## 🔍 Entity Queries

### Basic Query

```swift
// Find all entities with a specific component
let query = EntityQuery(where: .has(SculptingToolComponent.self))
let entities = scene.performQuery(query)
```

### Complex Queries

```swift
// Multiple conditions
let query = EntityQuery(where: 
    .has(ModelComponent.self) && 
    .has(SculptingToolComponent.self)
)

// With relationships
let query = EntityQuery(where: 
    .has(ModelComponent.self) && 
    .has(Parent.self)
)
```

## 💡 ECS Best Practices

### 1. Keep Components Simple

```swift
// Good: Single responsibility
struct HealthComponent: Component {
    var health: Int
    var maxHealth: Int
}

// Bad: Too many responsibilities
struct CharacterComponent: Component {
    var health: Int
    var position: SIMD3<Float>
    var inventory: [Item]
    var quests: [Quest]
    // Too much!
}
```

### 2. Prefer Composition

Instead of:
```swift
class SculptingToolEntity: Entity {
    // Inheritance-based
}
```

Use:
```swift
let entity = Entity()
entity.components.set(SculptingToolComponent(...))
entity.components.set(ModelComponent(...))
// Composition-based
```

### 3. Systems Process Components

```swift
// System only cares about components, not entity types
struct DamageSystem: System {
    let query = EntityQuery(where: .has(HealthComponent.self))
    
    func update(context: SceneUpdateContext) {
        for entity in context.entities(matching: query) {
            // Process any entity with HealthComponent
        }
    }
}
```

## 🛠 Custom Components Tips

### Component Definition

```swift
struct MyComponent: Component {
    // Use value types when possible
    var position: SIMD3<Float>
    
    // Reference types need care
    weak var target: Entity?
    
    // Computed properties are useful
    var isValid: Bool {
        return target != nil
    }
}
```

### Component Lifecycle

```swift
// Component added
entity.components.set(MyComponent())

// Component accessed (creates copy)
var component = entity.components[MyComponent.self]

// Component modified (must set back)
component?.position = newPosition
entity.components.set(component!)

// Component removed
entity.components.remove(MyComponent.self)
```

## 🎮 Practical Examples

### Example 1: Tool State Management

```swift
// Toggle sculpting mode
if var toolComponent = entity.components[SculptingToolComponent.self] {
    toolComponent.mode = toolComponent.mode == .add ? .remove : .add
    entity.components.set(toolComponent)
}
```

### Example 2: Finding Entities

```swift
// Find all mesh chunks
let meshQuery = EntityQuery(where: .has(ModelComponent.self))
for meshEntity in scene.performQuery(meshQuery) {
    // Update mesh appearance
}
```

### Example 3: System Communication

```swift
// One system sets data
sculptingTool.components[SculptingToolComponent.self]?.saveToTexture = texture

// Another system reads it
if let (texture, completion) = component.saveToTexture {
    // Perform save operation
}
```

## 🚀 Performance Considerations

### Query Caching

```swift
struct MySystem: System {
    // Store query as property
    static let query = EntityQuery(where: .has(MyComponent.self))
    
    func update(context: SceneUpdateContext) {
        // Reuse cached query
        let entities = context.entities(matching: Self.query)
    }
}
```

### Component Access

```swift
// Inefficient: Multiple lookups
if entity.components.has(MyComponent.self) {
    let component = entity.components[MyComponent.self]!
    // Use component
}

// Efficient: Single lookup
if let component = entity.components[MyComponent.self] {
    // Use component
}
```

## 📚 Advanced Topics

### Custom Entity Types

```swift
class SculptingToolEntity: Entity {
    required init() {
        super.init()
        self.components.set(SculptingToolComponent(...))
    }
}
```

### Component Events

```swift
// Listen for component changes
entity.components.set(MyComponent())
entity.subscribe(to: ComponentEvents.DidSet.self) { event in
    print("Component was set: \(event.componentType)")
}
```

## 🔗 Related Topics

- [User Interaction](07-USER-INTERACTION.md) - How UI interacts with ECS
- [API Reference](API-REFERENCE.md) - Component details
- [Metal Compute](04-METAL-COMPUTE.md) - GPU system integration

## 💡 Key Takeaways

1. **Entities are just IDs** - Components hold the data
2. **Systems contain logic** - Keep components data-only
3. **Composition is flexible** - Mix and match components
4. **Queries find entities** - Systems process matching entities
5. **RealityKit handles rendering** - Focus on your game logic

The ECS pattern makes the code more modular, reusable, and easier to extend. Each piece has a single responsibility, making the system easier to understand and maintain!
