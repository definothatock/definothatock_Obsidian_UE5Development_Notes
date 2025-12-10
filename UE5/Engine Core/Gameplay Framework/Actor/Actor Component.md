In Unreal, “components” are concrete engine types that attach to an Actor at runtime to add data/behavior. They’re not a packaging unit; they’re part of the Actor model.

- Core idea: Actor = container. Components = pluggable pieces of functionality you attach to that Actor.
    
- Base types:
    
    - UActorComponent: non-visual behavior/state (tick, replication, events).
        
    - USceneComponent: adds a transform (location/rotation/scale) and can be parented in a scene hierarchy.
        
    - UPrimitiveComponent: renderable/collidable scene component (meshes, collision shapes).
        

Examples:

- MovementComponent (movement logic)
    
- HealthComponent (custom, user-defined)
    
- AudioComponent (sound playback)
    
- StaticMeshComponent / SkeletalMeshComponent (visuals)
    
- CapsuleComponent / BoxComponent (collision)
    
- WidgetComponent (3D UI)
    

Architecturally:

- Composition over inheritance: most features are built by composing components onto a minimal Actor class.
    
- Ownership/lifecycle: Actor owns components; engine ticks and replicates them with the Actor.
    
- Messaging: Components receive actor/component events (BeginPlay, Tick, OnDestroyed) and can broadcast delegates.
    
- Networking: Components can replicate state via their owning Actor.
    
- Reuse: The same component class can be reused across many different Actors.
    

So, when choosing your HP system:

- Prefer a HealthComponent (UActorComponent) attached to the Actor that “has health.”
    
- Optionally expose an interface (IDamageable) for generic interaction.
    
- Keep Controllers for input/AI; health lives with the Actor via its component.
