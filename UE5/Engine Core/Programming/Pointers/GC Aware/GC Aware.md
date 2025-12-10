# UObject-related pointers (GC-aware / engine-aware)

These point to classes deriving from `UObject` (`UActorComponent`, `AActor`, etc.):

1. `UObject*` + `UPROPERTY`
    - Strong GC reference. Keeps the object alive.

2. `TWeakObjectPtr<UObject>`
    - Weak GC reference. Does _not_ keep alive; lets you test if it’s still valid.

3. `TSoftObjectPtr<UObject>` / `TSoftClassPtr<UClass>`
    - Soft reference by _asset path_. Object may be unloaded; can be loaded on demand.

4. `TObjectPtr<UObject>`
    - Introduced for UE5: a type-safe wrapper widely used instead of raw `UObject*` in UPROPERTYs.

5. Raw `UObject*` (no UPROPERTY)
    - ***Non-GC-owned***, unsafe if the object gets destroyed.


Plus some helpers: 
- `TScriptInterface<IMyInterface>` – wrap interface pointers.
- `TSubclassOf<USomeBase>` – pointers to classes (types) rather than instances.



## UObject-related pointers in detail

### 1.1 Raw `UObject*` vs `TObjectPtr<UObject>`

**Raw pointer analogy**: Same as `MyClass*` in C++. It just points somewhere.

In Unreal:

```cpp
UTexture2D* Texture;           // raw pointer, engine doesn't track
TObjectPtr<UTexture2D> TextureObjectPtr;  // UE5 wrapper, engine-aware when UPROPERTY
```

#### When it’s inside a `UCLASS` with `UPROPERTY`

```cpp
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    UPROPERTY()
    UStaticMeshComponent* MeshRaw;

    UPROPERTY()
    TObjectPtr<UStaticMeshComponent> Mesh;
};
```

- Both `MeshRaw` and `Mesh` are **GC-tracked** because they’re marked `UPROPERTY`.
- In UE5, Epic prefers `TObjectPtr` because it:
    - Provides internal safety checks.
    - Works better with 64-bit/GC compaction internals.

Think of `TObjectPtr` as: _“this is a UObject reference that should participate in UE’s memory model”_.

**Key rule:**
- If it’s a **member of a `UCLASS`/`USTRUCT`** and should keep the object alive → use `UPROPERTY(TObjectPtr<T>)` (or `UPROPERTY(TObjectPtr<AActor>)` etc.).

#### When it’s _not_ a UPROPERTY

```cpp
UStaticMeshComponent* TempMesh;
TObjectPtr<UStaticMeshComponent> TempMeshObjPtr;
```

- GC **does not see** them.
- They might become invalid if the object is destroyed/GC’d.
- Only safe as _short-lived_ or you must guarantee the target stays alive for their lifetime (e.g. within a function, or known permanent objects).

**Analogy:** Using raw pointers to objects that a separate garbage collector might free under you if you don’t register them as strong references.

---

### 1.2 Strong vs weak UObject references

#### Strong reference: `UPROPERTY(TObjectPtr<T>)` / `UPROPERTY(TObjectPtr<AActor>)`
- Keeps object from being GC’d.
- Use for:
    - Components of your actor
    - Key references you own or use consistently
    - Things you want to serialize and be saveable

**Example:**

```cpp
UCLASS()
class AWeapon : public AActor
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TObjectPtr<USkeletalMesh> WeaponMesh;

    UPROPERTY()
    TObjectPtr<ACharacter> OwningCharacter;
};
```

- As long as the `AWeapon` exists, its `WeaponMesh` and `OwningCharacter` won’t be garbage collected.

#### Weak reference: `TWeakObjectPtr<T>`

Used when:
- You **don’t own** the object.
- You want to **avoid preventing** its destruction.
- You want to be able to ask “Is this still alive?” safely.

**Analogy:** Phone number of a friend vs living with them. Having their phone number doesn’t keep them alive or with you; you can call to see if they pick up.

**Example: cache of nearby enemies**

```cpp
UPROPERTY()
TArray<TWeakObjectPtr<AEnemy>> NearbyEnemies;
```

Later:

```cpp
for (int32 i = NearbyEnemies.Num() - 1; i >= 0; --i)
{
    if (NearbyEnemies[i].IsValid())
    {
        AEnemy* Enemy = NearbyEnemies[i].Get();
        // Use Enemy safely
    }
    else
    {
        NearbyEnemies.RemoveAt(i); // enemy was destroyed
    }
}
```

Key API:
- `IsValid()` – also checks not pending kill.
- `Get()` – returns raw pointer or `nullptr`.
- `Get()` is safe only if `IsValid()` was true or if you accept null checks.

Use `TWeakObjectPtr` for:
- Caches
- Observers/listeners
- “Remember this thing, if it still exists”

---

### 1.3 Soft references: `TSoftObjectPtr` and `TSoftClassPtr`

These are **references by asset path**, not by direct pointer.

**Analogy:** Instead of holding a live `UTexture2D*` (the texture in memory), you hold “`/Game/Textures/MyTexture`” (the address of the file). The texture may not be loaded; you can ask the engine to load it later.

```cpp
UPROPERTY(EditAnywhere)
TSoftObjectPtr<UTexture2D> Icon;
```

In editor/blueprints, you can pick any texture. But at runtime:
- The level might load, but not all soft-referenced assets will be put into memory.
- When you want to use it:

```cpp
UTexture2D* LoadedIcon = Icon.Get();
if (!LoadedIcon)
{
    // Not loaded yet; load synchronously:
    LoadedIcon = Icon.LoadSynchronous();
}
```

Or asynchronously:

```cpp
StreamableManager.RequestAsyncLoad(Icon.ToSoftObjectPath(), FStreamableDelegate::CreateUObject(...));
```

Use `TSoftObjectPtr` when:
- You want **referencing without forcing it to be loaded**.
- You care about **cooking/packaging** references so the asset is included, but not necessarily in memory at start.
- Large assets (textures, levels, sounds, data assets).

`TSoftClassPtr<UObject>` is the same idea, but for **class types** (like a class you can spawn later).

**Common confusion vs `TWeakObjectPtr`:**
- `TWeakObjectPtr` = weak reference to a **live object instance**.
- `TSoftObjectPtr` = soft reference to an **asset path**, may be loaded/unloaded many times, not tied to a single instance.

---

### 1.4 Class pointers: `UClass*`, `TSubclassOf<T>`, `TSoftClassPtr<T>`

#### `UClass*`
- Runtime representation of a class type (like reflection type information).
- Example: `UClass* PlayerClass = AMyPlayer::StaticClass();`

#### `TSubclassOf<T>`
Template that ensures the `UClass*` is a subclass of `T`.

```cpp
UPROPERTY(EditAnywhere)
TSubclassOf<APawn> PawnClass;
```

In the editor, you can pick any Blueprint or C++ class that derives from `APawn`. To use it:

```cpp
APawn* SpawnedPawn = GetWorld()->SpawnActor<APawn>(PawnClass, SpawnTransform);
```

Internally it holds a `UClass*`, but with type-safety and editor filtering.

#### `TSoftClassPtr<T>`
Soft reference to a class by path. Just like `TSoftObjectPtr`, but for class types:


```cpp
UPROPERTY(EditAnywhere)
TSoftClassPtr<APawn> EnemyPawnClass;
```

Use it when the class is defined in some asset that might not be loaded on startup (e.g., blueprint in a DLC pack).

---

### 1.5 Interface pointers: `TScriptInterface`

If you have:

```cpp
UINTERFACE(MinimalAPI)
class UInteractable : public UInterface
{
    GENERATED_BODY()
};

class IInteractable
{
    GENERATED_BODY()

public:
    virtual void Interact() = 0;
};
```

You can store an interface pointer like:

```cpp
UPROPERTY()
TScriptInterface<IInteractable> InteractableTarget;
```

This stores:
- A `UObject*` (the actual object)
- Plus a pointer to the interface vtable.

It’s GC-aware and blueprint-friendly.

Usage:

```cpp
if (InteractableTarget)
{
    InteractableTarget->Interact();
}
```