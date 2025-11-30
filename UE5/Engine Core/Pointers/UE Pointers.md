# Why Unreal has so many pointer types

In a typical university C++ project:

- You have `MyClass* Ptr = new MyClass();`
- You worry about:
    - When to `delete` it
    - Who owns it
    - Avoiding dangling pointers and leaks

In Unreal:

- There is a **reflection system** (UCLASS, USTRUCT, UPROPERTY).
- There is **garbage collection (GC)** for `UObject`s, not manual `delete`.
- The engine needs to:
    - Know which objects reference which other objects (for GC, saving, blueprints).
    - Handle objects that might be loaded/unloaded from disk/levels asynchronously.
    - Provide thread-safe, editor-safe handling of resources.

So Unreal adds pointer types that encode:

- **Ownership** (who is responsible for the lifetime?)
- **Validity** (can it become null when object gets destroyed/unloaded?)
- **Reflectability** (does GC know this reference exists?)
- **Load behavior** (is the object guaranteed to be loaded now, or just referenced by path?)


# Pointer Families

There are 2 main pointer categories in Unreal Engine:
- [[GC Aware]]
- [[Non GC Aware]]
An one special category:
 - [[Delegates]]

As Discussed above, Non-GC-Aware **do not** talk to Unreal GC. They’re for plain C++ memory management.

For [[Delegates]], they are not pointing to any object, but to functions instead. Thus, They have completely different behaviour and worth discussing on its own.

---

## When to use what (cheat sheet)

### 1.1 Inside a `UObject` / `AActor` member

You want the engine to:

- Keep the object alive
- Reflect it in editor/blueprints
- Serialize it in saves/levels

Use:

```cpp
UPROPERTY()
TObjectPtr<USomeUObject> StrongRef;
```

Common categories:

- Components, owned sub-objects
- Owner -> child references (like a weapon’s mesh, character’s inventory objects)

### 1.2 Cached reference but not owning

You want:

- To remember “last target”, “previous instigator”, “nearby enemies”, etc.
- Not to stop them from being destroyed.
- To detect if they were destroyed.

Use:

```cpp
UPROPERTY()
TWeakObjectPtr<AActor> WeakActorRef;
```

### 1.3 Asset references that may not be loaded

You want:

- To reference an asset but not load it immediately.
- To load on demand (for memory and startup performance).

Use:

```cpp
UPROPERTY(EditAnywhere)
TSoftObjectPtr<UTexture2D> Icon;
```

### 1.4 Referencing a class type (like for spawning)

You want:

- Editor-exposed class that must derive from a base.
- To spawn instances at runtime.

Use:

```cpp
UPROPERTY(EditAnywhere)
TSubclassOf<APawn> PawnClass;
```

### 1.5 Pure C++ objects (non-UObject)

You want:

- RAII-style memory management
- Destruction controlled by your C++ scopes

Use:

- `TUniquePtr<T>` for single-owner
- `TSharedPtr<T>` / `TWeakPtr<T>` for shared-owner graphs
- Raw `T*` for non-owning, or short-lived references

---

## 2. Concrete examples with analogies

### Example 1: Player with a current weapon

Ownership & GC:

```cpp
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // Strong: as long as the character exists, weapon stays alive.
    UPROPERTY()
    TObjectPtr<AWeapon> CurrentWeapon;

    // Weak: remembers last interacted object, but doesn't keep it alive.
    UPROPERTY()
    TWeakObjectPtr<AActor> LastInteractedActor;
};
```

Analogy:

- **CurrentWeapon** = you’re renting an apartment; as long as you pay, the apartment exists (you own it).
- **LastInteractedActor** = you know someone’s Instagram handle; even if they delete their account, you still have the string, but it may no longer correspond to a real person.

### Example 2: UI icon system

Soft reference to icon assets:

```cpp
UCLASS()
class UItemData : public UDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TSoftObjectPtr<UTexture2D> Icon;
};

void UMyInventoryWidget::SetItem(UItemData* ItemData)
{
    if (!ItemData) return;

    UTexture2D* IconTexture = ItemData->Icon.Get();
    if (!IconTexture)
    {
        IconTexture = ItemData->Icon.LoadSynchronous();
    }

    IconImage->SetBrushFromTexture(IconTexture);
}
```

Analogy:

- **TSoftObjectPtr** = storing a URL/bookmark; webpage may not be open; you can open it when needed.

### Example 3: Mixed Non-UObject and UObject

```cpp
// Plain C++ object, no reflection, no GC.
struct FJob
{
    FString Name;
    int32 Priority;
};

// Manager that lives as a subsystem (UObject).
UCLASS()
class UJobManager : public UObject
{
    GENERATED_BODY()

    // Shared ownership among subsystems or other managers.
    TArray<TSharedPtr<FJob>> ActiveJobs;

public:
    void AddJob(const FString& Name, int32 Priority)
    {
        ActiveJobs.Add(MakeShared<FJob>(FJob{Name, Priority}));
    }
};
```

Here, `FJob` is managed by `TSharedPtr`, no GC involved.

---

## 3. Common pitfalls for someone used to raw C++ pointers

1. **Assuming `UObject*` is always safe if not null.**
    
    - If it’s not referenced by a `UPROPERTY` anywhere, it may be GC’d and your pointer becomes dangling.
    - Always ensure long-lived references are via GC-visible fields: `UPROPERTY(TObjectPtr<T>)`.
2. **Using raw pointers where you need soft references.**
    
    - If a resource can be unloaded and reloaded (like assets in large projects), prefer `TSoftObjectPtr`.
3. **Using `TSharedPtr` / `TUniquePtr` for UObjects.**
    
    - Never do this. `UObject`s are owned by the engine and GC, not by C++ smart pointers.
4. **Not differentiating instance vs asset path.**
    
    - `TWeakObjectPtr<AActor>` → points to a _runtime instance_ in the world.
    - `TSoftObjectPtr<UDataAsset>` → points to an _asset file_ on disk (can be loaded many times / in multiple levels).

---

## 4. Summary in one table

| Use case                                       | Type                                                      | Notes                          |
| ---------------------------------------------- | --------------------------------------------------------- | ------------------------------ |
| Own another `UObject` as part of your object   | `UPROPERTY(TObjectPtr<T>)`                                | Strong GC reference            |
| Non-owning reference to `UObject` that may die | `UPROPERTY(TWeakObjectPtr<T>)` or raw `TWeakObjectPtr<T>` | Must check `IsValid()`         |
| Asset reference that may not be loaded         | `UPROPERTY(TSoftObjectPtr<T>)`                            | Forces cook, load when needed  |
| Editable class to spawn later                  | `UPROPERTY(TSubclassOf<T>)`                               | Type-safe `UClass*`            |
| Soft reference to class type                   | `UPROPERTY(TSoftClassPtr<T>)`                             | Class by path                  |
| Interface pointer                              | `UPROPERTY(TScriptInterface<IMyInterface>)`               | UObject + interface            |
| Pure C++ sole ownership                        | `TUniquePtr<T>`                                           | Like `std::unique_ptr`         |
| Pure C++ shared ownership                      | `TSharedPtr<T>`                                           | Like `std::shared_ptr`         |
| Pure C++ weak reference to shared              | `TWeakPtr<T>`                                             | Like `std::weak_ptr`           |
| Short-lived raw pointer (no GC involvement)    | `T*` or `UObject*`                                        | Only safe if you know lifetime |

---

