### Key Features of an Interface in Unreal Engine:

1. **No Implementation**:
    
    - Interfaces only specify the function signatures (name, parameters, and return type) but provide no implementation. The actual behavior of the functions must be implemented in the classes that use the interface.
        
2. **Multi-Class Support**:
    
    - Unlike inheritance, where a class can only inherit from one base class, a class in Unreal can implement multiple interfaces. This allows for a form of "multiple inheritance."
        
3. **Blueprint Support**:
    
    - Interfaces can be created and used in both **C++** and **Blueprints**, making them versatile for developers and designers.
        
4. **Reusable Contracts**:
    
    - Interfaces are great for defining a common set of functionality that unrelated classes can implement. For instance, you might have an interface for "Damageable" entities, which both a player character and an enemy NPC could implement, even if they are entirely different classes.

### Advantages of Using Interfaces

1. **Decoupling**:
    
    - Interfaces help decouple systems in your game, making the code more modular and reusable.
        
2. **Flexibility**:
    
    - Classes don't need to share a common parent class to implement the same interface. This is particularly useful when working with Unreal’s actor and component hierarchy.
        
3. **Blueprint-Friendly**:
    
    - Interfaces allow designers to easily create systems without needing to touch code.
      
### Think in layers.

- Pure C++ interface
    
    - Lives only in C++ (no engine systems involved).
        
    - No Unreal reflection. No UObject. No Blueprint.
        
    - Best when you stay 100% in native C++ and don’t need editor/Blueprint visibility.
        
- UE C++ interface (UInterface + IInterface)
    
    - Built on top of UObject and the Unreal Reflection system (UHT/UClass/UFunction).
        
    - Uses engine casting (Cast, ImplementsInterface) and reflection metadata.
        
    - This is the “engine-level” interface for gameplay that must cross C++ and Blueprint.
        
    - Best default choice for gameplay contracts in UE projects.
        
- Blueprint Interface (BPI)
    
    - Editor asset that compiles to reflection data and Blueprint VM thunks.
        
    - Runs through the Blueprint VM when implemented in BP.
        
    - Best for quick, BP-first decoupling between Blueprints.
        

### How to choose

- Need Blueprint access or editor discoverability? UE C++ interface.
    
- Pure performance/engine-agnostic module? Pure C++ interface.
    
- Rapid BP-only communication, minimal C++? Blueprint Interface.
    
- Mixed teams (C++ + BP) or long-term maintainability? Start with UE C++ interface; let BPs implement it as needed.

[[Engine Core]]