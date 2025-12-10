## 1. Concept: what is a delegate in UE?

In normal C++:

```cpp
void FreeFunction(int X) { }

struct FObj {
    void Method(int X) { }
};
```

If you want to store a _callable_:

- You might use a function pointer: `void (*FuncPtr)(int)`
- Or a member function pointer + object: `void (FObj::*MemFunc)(int)` and `FObj* Instance`
- Or `std::function<void(int)>` that wraps lambdas, free functions, methods, functors.

**Unreal delegate = UE’s “std::function”** that:

- Can point to:
    - Free functions
    - Static functions
    - Member functions (on `UObject` or non-UObject)
    - Lambdas (in some cases)
- Stores both:
    - What to call (function pointer / member function pointer)
    - Who to call it on (object pointer / context)

So yes, a delegate contains pointers, but it’s:

> “A _callable pointer_ (function + optional object) that can be bound/unbound and invoked later.”

---

## 2. Where delegates live in the families you saw

From the [[UE Pointers]] families:

### A. UObject-related family

- **Delegates that are `UPROPERTY()`-exposed or `BlueprintAssignable`**:
    - Use `DECLARE_DYNAMIC_DELEGATE` / `DECLARE_DYNAMIC_MULTICAST_DELEGATE`.
    - Store `UObject*` receivers in a **GC-aware / weak** way.
    - Integrate with reflection, GC, and Blueprints.

These are in the **UObject world**, but they’re not “ownership” pointers. They don’t keep their target alive; if the UObject dies, the dynamic delegate is auto-unbound or becomes invalid safely.

### B. Non-UObject C++ family

- **Native C++ delegates** (non-dynamic):
    - `DECLARE_DELEGATE`, `DECLARE_MULTICAST_DELEGATE`, etc.
    - Can bind to:
        - Non-UObject C++ classes
        - Lambdas
        - Static/free functions
    - No reflection / no GC, just like `TSharedPtr`, `TUniquePtr`, raw function pointers.

These belong in the **C++-only** family, beside `TSharedPtr` etc., but for _callables_ rather than _objects_.

---

## 3. Delegate types and how “pointer-like” they are

Think of three axes:

1. **Dynamic vs non-dynamic** (reflection/Blueprint vs C++-only)
2. **Single vs multicast** (one listener vs many listeners)
3. **What they store** (plain function pointer, `UObject*`, lambda, etc.)

### 3.1 Non-dynamic (C++ only) delegates

These are most like “smart function pointers.”

Common forms:

- `DECLARE_DELEGATE` – no parameters, single-cast
- `DECLARE_DELEGATE_OneParam` – one parameter, etc.
- `DECLARE_MULTICAST_DELEGATE` – multi-cast (multiple bound functions)

Example:

```cpp
// Declaration (in a header)
DECLARE_DELEGATE_OneParam(FOnHealthChanged, float);

// Use inside a class
class FHealthComponent
{
public:
    FOnHealthChanged OnHealthChanged;

    void SetHealth(float NewHealth)
    {
        Health = NewHealth;
        OnHealthChanged.ExecuteIfBound(Health);
    }

private:
    float Health;
};
```

**Pointer relation:**

- Internally stores something akin to `std::function<void(float)>`.
- When bound to a member function, it stores:
    - A pointer to the object: `FSomeClass*`
    - A pointer to the member function: `void (FSomeClass::*)(float)`
- No GC awareness. If the object is freed and delegate still holds a pointer, you get UB on call.  
    → You must unbind manually or guarantee lifetime.

**Analogy:**  
Like a `std::function` containing a method + raw `this` pointer. You must manage the lifetime of `this` yourself.

---

### 3.2 Dynamic delegates (GC-aware, Blueprint-friendly)

Dynamic delegates are:

- Reflectable (can appear in Blueprints, are exposed with UFUNCTION).
- Store target as a `UObject` reference in a GC-safe way.
- Can be marked as `BlueprintAssignable` or `BlueprintCallable`.

Common forms:

- `DECLARE_DYNAMIC_DELEGATE_*`
- `DECLARE_DYNAMIC_MULTICAST_DELEGATE_*`

Example:

```cpp
// In a UCLASS UHealthComponent
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnHealthChangedDynamic, float, NewHealth);

UCLASS()
class UHealthComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UPROPERTY(BlueprintAssignable, Category="Health")
    FOnHealthChangedDynamic OnHealthChanged;

    void SetHealth(float NewHealth)
    {
        Health = NewHealth;
        OnHealthChanged.Broadcast(Health);
    }

private:
    UPROPERTY()
    float Health;
};
```

**Pointer relation:**

- Stores:
    - A `UObject*` to the listener.
    - A reference to a `UFunction` (Unreal’s reflectable function representation).
- These references are **GC-aware**: the engine knows “this delegate references that UObject”.

**Lifetime behavior vs pointers:**

- If the bound `UObject` is destroyed:
    - The dynamic delegate _won’t_ leave a dangerous raw pointer.
    - Typically it’s safely unbound or becomes invalid; `Broadcast` won’t call it.
- So: dynamic delegates are like **GC-tracked weak function pointers** to `UObject` methods.

They do **not** strongly own the object. They’re closer to `TWeakObjectPtr` on the object side, combined with a function pointer.

**Analogy with families:**

- For the UObject side: dynamic delegates are similar in spirit to using `TWeakObjectPtr<UObject>`—safe if object dies.
- For the function side: they store a reflectable “pointer to UFUNCTION”.

---

### 3.3 Single-cast vs multi-cast

- **Single-cast**: only one function can be bound at a time.
- **Multi-cast**: can have a list of listeners; `Broadcast()` calls each in order.

Types:

- `DECLARE_DELEGATE...` → single-cast
- `DECLARE_MULTICAST_DELEGATE...` → multi-cast
- `DECLARE_DYNAMIC_DELEGATE...` → single-cast dynamic
- `DECLARE_DYNAMIC_MULTICAST_DELEGATE...` → multi-cast dynamic

**Pointer analogy:**

- Single-cast = “one function pointer.”
- Multi-cast = “vector/array of function pointers” (stored in a safe container).

- ---

### 4.1 Comparison table

| **C++-only delegate**     | `DECLARE_DELEGATE`, etc.         | Holds raw function and/or object pointer, no GC       | Manual       |
| ------------------------- | -------------------------------- | ----------------------------------------------------- | ------------ |
| **Dynamic delegate**      | `DECLARE_DYNAMIC_DELEGATE`, etc. | Holds `UObject*` + `UFunction*` in GC/reflectable way | Weak-like    |

Key takeaway:

- Delegates are not about _owning_ objects but about _calling_ functions on objects.
- Non-dynamic delegates = **C++-only**, like smart function pointers, no GC.
- Dynamic delegates = **UObject-aware**, like **weak references** to `UObject` methods.

---

## 5. Examples that show relationships

### 5.1 C++-only observer with `TWeakPtr` and delegate

You might have a pure C++ system:

```cpp
DECLARE_DELEGATE(FOnJobDone);

class FJob
{
public:
    FOnJobDone OnDone;
    void Run()
    {
        // ...
        OnDone.ExecuteIfBound();
    }
};

class FJobOwner : public TSharedFromThis<FJobOwner>
{
public:
    void CreateJob()
    {
        TSharedPtr<FJob> Job = MakeShared<FJob>();
        TWeakPtr<FJobOwner> WeakSelf = AsShared(); // like TWeakObjectPtr but for C++ objects

        Job->OnDone.BindLambda([WeakSelf]()
        {
            if (TSharedPtr<FJobOwner> Self = WeakSelf.Pin())
            {
                Self->OnJobFinished();
            }
        });
    }

    void OnJobFinished() { /* ... */ }
};
```

Connections:

- `TSharedPtr` / `TWeakPtr` manage **C++ object lifetime**.
- Delegate stores a lambda, which internally stores `TWeakPtr`.
- No GC, all manual RAII.

Here, the delegate’s relationship to pointers is similar to storing a `std::function` that closes over a `std::weak_ptr`.

---

### 5.2 UObject with dynamic multicast delegate

Now in the UObject world:

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnHitDynamic, AActor*, OtherActor);

UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    UPROPERTY(BlueprintAssignable, Category="Events")
    FOnHitDynamic OnHit;

    void SimulateHit(AActor* Other)
    {
        OnHit.Broadcast(Other);
    }
};
```

And in another actor:

```cpp
UCLASS()
class AListenerActor : public AActor
{
    GENERATED_BODY()

public:
    UPROPERTY()
    TObjectPtr<AMyActor> ListenedActor;

    virtual void BeginPlay() override
    {
        Super::BeginPlay();
        if (ListenedActor)
        {
            ListenedActor->OnHit.AddDynamic(this, &AListenerActor::OnHitHandler);
        }
    }

    UFUNCTION()
    void OnHitHandler(AActor* OtherActor)
    {
        UE_LOG(LogTemp, Log, TEXT("Hit: %s"), *GetNameSafe(OtherActor));
    }
};
```

Relations:

- `ListenedActor` (a `TObjectPtr<AMyActor>`) is a **strong UObject pointer**: keeps the actor alive via GC.
- `OnHit` delegate stores:
    - A **weak-like reference** to `this` (listener). If `this` is destroyed, the delegate becomes safe; it doesn’t crash.
- `AddDynamic(this, &AListenerActor::OnHitHandler)` is like:  
    “Please store a safe pointer to this `UObject` and its UFUNCTION for later calls.”

So relative to the pointer family:

- `TObjectPtr` here = strong ownership / GC root.
- Dynamic delegate = reference to a **method on that object** with lifetime checked by UE.

---

## 6. Typical pitfalls from a “raw pointer” mindset

1. **Assuming delegates own their target.**
    
    - Delegates **never own**; they don’t free anything. They’re more like `std::function` than `std::unique_ptr`.
    - They need correct lifetime management of the targets (C++ or UObject).
2. **Assuming non-dynamic delegates are GC-safe.**
    
    - `DECLARE_DELEGATE` etc. do NOT interact with UE GC.
    - Binding them to UObjects is dangerous if the UObject can be destroyed; you need manual unbind or lifetime guarantees.
3. **Not distinguishing dynamic vs non-dynamic.**
    
    - If you want Blueprints / reflection / safe `UObject` handling → use dynamic (`DECLARE_DYNAMIC_...`).
    - If you want speed / C++ only / lambdas / non-UObject → use non-dynamic (`DECLARE_DELEGATE...`).

---

## 7. Short “recipe” view

### When to use which delegate type?

- **C++ only, performance, may bind lambdas or non-UObject classes:**
    
    cpp
    
    ```cpp
    DECLARE_DELEGATE / DECLARE_DELEGATE_OneParam
    DECLARE_MULTICAST_DELEGATE / _OneParam
    ```
    
- **Needs Blueprint exposure / UPROPERTY / BlueprintAssignable / UFUNCTION:**
    
    cpp
    
    ```cpp
    DECLARE_DYNAMIC_DELEGATE / _OneParam
    DECLARE_DYNAMIC_MULTICAST_DELEGATE / _OneParam
    ```
    

### How they relate to pointer families

- Non-dynamic delegates belong to the **“C++-only, non-UObject”** family, like `TSharedPtr` and raw `T*`.
- Dynamic delegates belong to the **“UObject-aware / GC-aware”** family, like `TWeakObjectPtr`:
    - They internally reference `UObject`s in a way that won’t crash when GC destroys them.
    - But they still don’t _own_ those objects.
